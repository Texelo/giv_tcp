# GivTCP-DEV 3.5.47 — HV stack API hotfix

The official `givtcp-dev` 3.5.47 build crashes on startup for any HV battery system
(HYBRID_HV_GEN3, etc.) with:

```
ERROR - AttributeError: 'Plant' object has no attribute 'HVStack'
ERROR - Error processing Three Phase data: ('AttributeError', 'read.py', 1847)
ERROR - inverter Update failed so using last known good data from cache: (Exception: Process Data Failure - 2061)
ERROR - Key Error getting Battery Data: Missing key 'multi_output'
ERROR - UnboundLocalError: cannot access local variable 'multi_output' where it is not associated with a value
```

on every single read cycle (every ~30-35s), forever.

## Root cause

`requirements.txt` in the 3.5.47 build pins `givenergy-modbus==2.12.0`
(https://github.com/dewet22/givenergy-modbus), a rewritten, pydantic-based library. That
rewrite:

- renamed `Plant.HVStack` -> `Plant.hv_stacks`
- changed each stack entry from an indexable `[bcu, [bmus]]` pair to an `HvStack` object
  with `.bcu` / `.bmus` attributes
- renamed `Bcu.number_of_module` -> `number_of_modules`,
  `Bcu.battery_nominal_capacity` -> `battery_nominal_capacity_ah`,
  `Bcu.remaining_battery_capacity` -> `remaining_battery_capacity_ah`
- dropped the old dict-like `.get()` / `.getall()` / `.getsn()` methods on `Bcu`/`Bmu`/`Meter`
  in favour of plain pydantic attribute access (`getattr`) / `model_fields`

`GivTCP/read.py` was only partially migrated (most of the file already uses
`plant.capabilities.is_hv` etc.) — `getRaw()` and `getBatteries()` still called the old
`plant.HVStack` API unconditionally for any HV system, which is why every read cycle for
an HV stack fails immediately.

The resulting `AttributeError` is caught, but propagates as a "Process Data Failure",
whose handling in `runAll2()` has its own bug: the `except KeyError` block logs and falls
through instead of returning, so the follow-up `return multi_output` throws an
`UnboundLocalError` because `multi_output` was never assigned — that's the second error
in every cycle, and why the real cause was hard to spot from the logs alone.

## What this fixes

`read.py` was only partially migrated to `givenergy-modbus==2.12.0`'s API. Fixed every
dead/renamed reference reachable from a three-phase HV inverter's read path
(`processThreePhaseInfo`, `getRaw`, `getBatteries`, `getInvModel`, `getMeters`,
`getTimeslots`, `getControls`), found by systematically diffing every attribute these
functions touch against the real installed library (`model_fields` introspection), not
just the one the logs happened to hit first:

- `getRaw()`, `getBatteries()`: `plant.HVStack` -> `plant.hv_stacks`, `stack[0]`/`stack[1]`
  -> `stack.bcu`/`stack.bmus`, renamed `Bcu` fields (`number_of_module` ->
  `number_of_modules`, `battery_nominal_capacity`/`remaining_battery_capacity` -> `*_ah`),
  `Bmu.get(...)` -> `getattr(...)`, `.getsn()` -> `.serial_number`.
- `getall()` helper: removed a `model.to_dict()` call that doesn't exist on `Bcu`/`Meter`
  pydantic models (this broke `getRaw()`'s meter dump too, independent of the HVStack bug).
- `processThreePhaseInfo()`: `p_load_ac1/2/3` and `p_out_ac1/2/3` were hard-removed in
  2.12.0 (they decoded per-phase power as unsigned/10x-wrong — see the library's own
  `AttributeError` message) — replaced with `p_meter_active_ac1/2/3` and
  `p_inverter_active_ac1/2/3`. Also `system_mode` and `battery_priority` are now plain
  register ints, not Enums, in 2.12.0 — dropped the now-invalid `.name.capitalize()` calls
  on those two (`System_Mode`/`Battery_Priority` sensors will report the raw register
  value instead of a decoded label until/unless GivEnergy's register meaning for these is
  confirmed and remapped).
- `runAll2()`: `except KeyError` now returns instead of falling through to the
  `UnboundLocalError`.

All renamed field names and method removals were verified directly against the installed
`givenergy-modbus==2.12.0` package (`model_fields` introspection + constructing real
`Bcu`/`Bmu`/`HvStack`/`ThreePhaseInverter` instances and exercising the patched code
against them), not just guessed from the error messages. See the diagnosis conversation
for the full trace.

## hotfix4: write.py (control commands)

While tracing a `setBatteryPauseMode` KeyError (turned out to be triggered by Predbat's
retry loop on top of the real bug below), found the same incomplete-migration pattern in
`write.py`: it calls control methods like `device.set_battery_pause_mode(val)` as instance
methods, but in `givenergy-modbus==2.12.0` most of these are free functions in
`givenergy_modbus.client.commands`, not methods on the inverter object at all.

Audited every `device.set_*(...)` call in `write.py` (85 call sites) with an AST-based
script that checks each one against the real installed library's actual signatures
(instance method vs. `commands.py` function, and argument count/names) — not just the
ones the logs happened to surface. Found and fixed:

- **12 methods entirely missing as instance calls** (`set_battery_pause_mode`,
  `set_battery_charge_limit_ac`, `set_battery_discharge_limit_ac`, `set_ems_plant`,
  `set_pause_slot(_start/_end)`, `set_export_slot(_start/_end)`) — routed through
  `commands.set_X(...)` instead of `device.set_X(...)`.
- **3 EMS-aware call sites** (`setChargeTarget2`/`setExportTarget`/`setDischargeTarget`,
  used for multi-inverter/AIO parallel systems with numbered slots) called methods that
  never existed under those names at all — mapped to the real equivalents:
  `commands.set_ems_charge_target_soc`, `set_ems_export_target_soc`,
  `set_ems_discharge_target_soc`.
- **`set_charge_target_only`** (3 call sites) doesn't exist anywhere — mapped to
  `commands.set_charge_target_soc`, which has the matching "set only the target, don't
  touch the enable bits" semantics per its docstring.
- **`device._set_charge_slot(...)`** (2 call sites) — leading-underscore typo for the
  real (public) `device.set_charge_slot(...)`.
- **`device.enable_charge_target()`** — the old bare "enable, keep whatever target was
  already set" call has no equivalent in 2.12.0 (`set_charge_target_enabled` now always
  requires a target value). Re-enables at the last cached `Target_SOC` (falling back to
  100 if none cached yet) via `commands.set_charge_target_enabled(...)`.
- A handful of call sites also passed a stale extra `GiV_Settings.inverter_type` argument
  left over from an old dual LV/HV signature — dropped, since the current API takes just
  the value.
- One pre-existing (unrelated to the library bump) syntax bug: `reqs.extend(X, Y)` with
  two positional args — `list.extend()` only takes one; the stray second argument was
  removed.
- Stale `logging.getLogger("givenergy_modbus_async")` (targeting a logger name that no
  longer exists, so the intended log-level suppression was silently a no-op) updated to
  `"givenergy_modbus"`.

Verified every fixed call resolves and type-checks against the real installed
`givenergy-modbus==2.12.0` (same AST-diff script, 0 problems remaining across all 85
call sites), and confirmed the instance-method wrappers that *do* still exist are thin
pass-throughs to the exact same `commands.py` functions — so routing through `commands.`
directly produces identical results, not just "probably works."

## hotfix5: HV battery count fed into max battery rate + missing Battery_Power_Reserve

Traced a Predbat crash (`ZeroDivisionError: float division by zero` in its charge-curve
calc, dividing by `Invertor_Max_Bat_Rate`) and a `KeyError: 'Battery_Power_Reserve'` back
to `read.py`:

- **`getInvModel()`**: `plant.number_batteries` only counts LV battery packs
  (`capabilities.lv_battery_addresses`) in `givenergy-modbus==2.12.0` — it's *always 0*
  for an HV system (this is what the very first startup log in this whole investigation
  showed: "0 batteries" despite 3 detected BCU stacks / BMUs). `batmaxrate = 25 * 80 *
  plant.number_batteries` therefore came out as exactly 0 for every HV three-phase system,
  which GivTCP publishes as `Invertor_Max_Bat_Rate: 0` — and Predbat divides by that.
  Fixed to count HV battery modules across `plant.hv_stacks` instead when
  `plant.capabilities.is_hv`.
- **`getControls()`**: three-phase systems only ever set `Battery_Power_Cutoff` (from the
  now-deprecated `battery_power_cutoff` alias) and never set `Battery_Power_Reserve` at
  all — the library's own deprecation note confirms HR(1078) is actually a single
  "Battery Reserve %" register, not a distinct "cutoff" concept, so both keys are now
  populated from the correctly-named `battery_reserve_soc`. Predbat (and anything else
  reading `Control.Battery_Power_Reserve` directly) was hitting a `KeyError` on
  three-phase/HV systems since this key never existed for them.

Verified the HV module-count fix against real `HvStack`/`Bmu` instances matching this
system's exact detected topology (3 BCU stacks, 3 BMUs) — comes out to a correct nonzero
`Invertor_Max_Bat_Rate`, not just "no longer zero by accident."

## hotfix6: Target_SOC (and other *_SOC number entities) showing "unknown"

Confirmed via direct REST query (`/readData`) that GivTCP itself correctly reports
`Control.Target_SOC: 0` once the read-path fixes above are in place - this isn't a data
bug. The cause is in `HA_Discovery.py`: every MQTT `number` entity whose name contains
"soc" (Target_SOC, Charge/Discharge/Export_Target_SOC_N, ...) is discovered with
`min=4` (matching GivEnergy's *write* validation - you can't *set* a target below 4%).
But the inverter legitimately *reports* 0 when that target/feature is disabled, and
Home Assistant's MQTT number entity rejects any received state outside its declared
min/max as invalid - so a real, correct reading of 0 was being silently turned into
"unknown" by HA itself. Changed the discovered `min` to 0 so HA can display the
disabled state; writes still go through `commands.set_charge_target_enabled`/
`set_charge_target_soc`, which enforce the real 4-100 range server-side regardless of
what the UI's slider allows.

## How this is packaged

None of the branches in this repo (`main`, `dev3`, `modbusv2`) match what's actually
shipping in the 3.5.47 dev image — they're all on an older/different library.

This started as a thin overlay (`FROM givtcp.docker.scarf.sh/britkat/giv_tcp-dev:3.5.47`
+ `COPY read.py`), but the scarf.sh gateway that fronts that image rate-limits `FROM`
pulls (`429 Too Many Requests`), which broke Supervisor's build more than once. As of
hotfix3 this is now a **self-contained build**: the full `/app` tree (`GivTCP/`,
`WebDashboard/`, `ingress/`, `startup.py`, etc.) was extracted from the exact 3.5.47
image, the same `read.py` fix applied on top, and `requirements.txt` pinned to the exact
package versions that image ships — verified byte-for-byte against `pip freeze` run
inside the original container, so there's no dependency drift. Only `python:3.14-alpine`
(pulled from Docker Hub, not scarf.sh) and its own patch-level updates aren't pinned;
everything Python-level is.

## Installing

Add `https://github.com/Texelo/giv_tcp` (branch `hotfix/3.5.47-hvstack-api`) as a custom
repository in Home Assistant → Settings → Add-ons → Add-on Store → ⋮ → Repositories, then
install **GivTCP-DEV (Texelo hotfix)**. It builds locally on your HA host from this
Dockerfile the first time you install it.
