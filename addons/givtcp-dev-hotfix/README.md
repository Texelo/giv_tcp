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

## hotfix7: graceful shutdown so the dongle doesn't hold a stale session

Neither `startup.py` nor `read.py` handled `SIGTERM` at all - every container stop or
rebuild abandoned the Modbus connection mid-air rather than closing it. Some GivEnergy
dongles keep an abandoned session "active" and refuse/ignore new connections for a
while afterwards (TCP handshake succeeds, but the actual Modbus request/response times
out) - plausible contributor to the connection issues seen during this session's many
rebuild cycles.

- `startup.py` (PID 1, no init/tini wrapper) now catches `SIGTERM` and forwards it
  (`.terminate()`) to the `read.py` subprocess(es) it tracks in `selfRun`, waiting up to
  5s for them to exit - previously nothing propagated SIGTERM to them at all, so they
  only ever died via SIGKILL when Docker's grace period expired.
- `read.py`'s `start()` now installs its own `SIGTERM` handler via
  `loop.add_signal_handler` and calls a new `GivClientAsync.close_connection()`
  (added to `GivLUT.py`, reuses the existing connection lock) before exiting.
- **Verified, not just written**: an isolated repro proved `self_run()`'s bare
  `except:` swallows `CancelledError`, so `task.cancel()` alone does *not* reliably
  stop it - it just catches the cancellation and loops back into `watch_plant()`,
  which would reopen the very connection we're trying to close. Fixed by calling
  `os._exit(0)` right after our own cleanup completes, rather than trusting
  cooperative cancellation to finish in time. Confirmed via real `docker kill
  --signal=SIGTERM` + timing: exits in ~425ms with the connection-close completing
  first, vs. the full 10s SIGKILL grace period before this fix.

## hotfix8: write.py error surfacing

Every write function's error handling discarded the actual reason for a failure. Two
compounding issues, both in the same boilerplate repeated ~40-50 times across the file:

- `if 'error' in result: raise Exception` was a *bare* `raise Exception` - it never
  passed through `result['error']`, the descriptive message `sendAsyncCommand()` had
  already built (e.g. `"Error in write command: <real reason>"`). Now
  `raise Exception(result.get('error'))`.
- Every `except:` block's `e=sys.exc_info()[0].__name__, os.path.basename(...), ...`
  only ever captured the exception's *class name*, filename and line number - never
  `str(sys.exc_info()[1])`, the exception's own message. So even with the above fix,
  the real text still wouldn't have reached the log. Now included in the tuple, so a
  failure like `Setting Battery Pause Slot failed: ('Exception', 'Error in write
  command: <real underlying reason>', 'write.py', 703)` actually says why, instead of
  just the file/line every single time.

Mechanical, logging-only change (verified no other `raise Exception(...)` call sites
were touched, and no non-`'error' in result:`-guarded `raise Exception` sites were
affected) - doesn't alter control flow or write behaviour, only what gets logged when
something fails.

## hotfix9: enable Battery Pause (mode/slot) writes on HV Gen3

hotfix8's better error surfacing revealed the real reason `setPauseSlot`/
`setBatteryPauseMode` failed: `HR(319) is not permitted for HYBRID_HV_GEN3 inverter`.
This isn't a migration bug - `givenergy-modbus` deliberately blocks battery-pause
writes (HR 318-320) at the client's per-model gate for *every* model, citing a
tracked-but-parked upstream issue (dewet22/givenergy-modbus#115, referenced from
#268): some GivEnergy firmware sends a malformed/error-shaped Modbus response
(function code 0x86) to a battery-pause write, which the old GivTCP-vendored library
silently treated as success but the new library has no decoder for. The block is
blanket and conservative - the issue itself says the affected hardware is
"presumably an LV battery" on **Gen1**, not confirmed on HV Gen3, and PDU-level
validation (`WRITE_SAFE_REGISTERS`) already recognises 318-320 as legitimate,
correctly-named registers ("pause battery", "pause battery start/end time").

Confirmed via the GivEnergy Cloud portal's own remote-control history that Battery
Pause is a genuinely supported, working control on this exact HV Gen3 inverter today
- strong evidence the block doesn't apply to this hardware, just catches it under a
blanket "we don't know which models are affected, so exclude everyone" precaution.

Added a narrow bypass: `sendAsyncCommand()` takes a `bypass_model_gate` flag that,
when set, sends each request via `client.send_request_and_await_response()` directly
instead of `one_shot_command()` - skipping *only* the per-model gate (Gate 1), while
every request still goes through `request.encode()`'s PDU-level validation (Gate 2,
`WRITE_SAFE_REGISTERS`) exactly as before, so this is not an unchecked write. The 8
call sites that build battery-pause requests (`setPauseSlot`, `setPauseStart`,
`setPauseEnd`, `setBatteryPauseMode`, and the pause-mode-inclusive branches of
`FEResume`/`forceExport`/`FCResume`/`forceCharge`) now pass
`bypass_model_gate=(device.model==Model.HYBRID_HV_GEN3)` - the bypass only ever
activates for this specific model, leaving the upstream block fully intact (and thus
whatever protection it's providing) for every other model, including the Gen1
hardware the block was actually written about.

Verified `device.model==Model.HYBRID_HV_GEN3` evaluates correctly against a real
`ThreePhaseInverter` instance, and that `request.encode()` (Gate 2) genuinely accepts
HR(318)/HR(319) rather than assuming it from reading the source.

**Residual risk**, stated plainly: if this specific inverter's firmware *does* turn
out to exhibit the 0x86 response anomaly the block exists for, a pause-mode write
will surface as a confusing/crashy error on that one attempt rather than a clean
"not permitted" rejection - not data loss or a hardware-safety issue, since 318-320
are a pause toggle and two time registers, not a protection/limit register. Worth
reporting upstream regardless (dewet22/givenergy-modbus#115) with this HV Gen3
confirmation, to help get the block properly scoped.

## hotfix10: fix the HV Gen3 model check itself (hotfix9 never actually activated)

hotfix9's bypass was correct in design but never fired: `device.model==Model.HYBRID_HV_GEN3`
was always False on this system. `device.model` decodes HR(0) *alone*, via the
library's plain `Model(dtc)` constructor - which, per the library's own docstring,
"only yields the coarse family" when the raw code isn't an exact 1-character match.
This system's raw device type code is `0x8103` -> hex string `"8103"` -> no exact
`Model` match -> falls back (via `Model._missing_`) to just the first character,
`Model("8")` = **`Model.ALL_IN_ONE`** - not `HYBRID_HV_GEN3`. This is exactly what
the very first startup log in this whole investigation showed ("All_in_one(8103)")
and was hiding in plain sight the entire time.

The correctly-resolved specific model (what every `detect:` log line actually shows)
comes from `resolve_model(raw_dtc, arm_fw)`, which additionally uses the firmware
version to disambiguate `"81"` (HYBRID_HV_GEN3) from `"82"`/`"83"` etc. - the
library uses exactly this pattern internally (`inverter.py`'s
`battery_energy_source` lookups) via two separate fields, `device_type_code` (the
raw hex string) and `arm_firmware_version`, both already present on `device`.

Added `_is_hv_gen3(device)`, calling `resolve_model(int(device.device_type_code, 16),
int(device.arm_firmware_version)) == Model.HYBRID_HV_GEN3` - swapped into all 8
`bypass_model_gate=...` call sites from hotfix9. Verified directly: `device.model`
for this exact `device_type_code`/`arm_firmware_version` pair is confirmed
`Model.ALL_IN_ONE`, while `_is_hv_gen3()` correctly returns `True` for this system
and `False` for another model and for missing/malformed data (doesn't crash).

## hotfix11: extend the model-gate bypass to Charge/Discharge/Export Target SOC

`setChargeTarget2`/`setExportTarget`/`setDischargeTarget` call `commands.set_ems_*`,
targeting the EMS-tier register block (HR2040-2071). Same root cause as the
pause-mode bug (hotfix9/10): Gate 1 (`write_safe_registers()`, the per-model
capability gate) only admits this block for `Model.EMS`/`Model.EMS_COMMERCIAL` -
`is_ems` capability, keyed purely off model taxonomy, never off actual observed
firmware behaviour. `HYBRID_HV_GEN3` is never `is_ems`, so these three calls were
always rejected: `HR(2046)/(2064)/(2055) is not permitted for HYBRID_HV_GEN3 inverter`.

Confirmed via the pre-rewrite vendored library (`givenergy_modbus_async`, still
sitting in this repo's `main`/`dev3`/`modbusv2` branches) that this system's own
history of working Charge/Discharge/Export Target SOC writes went through this
exact same register block with **zero write-safety gating at all** - the old
library never had a Gate 1/Gate 2 concept, it just sent the request. So there's no
new evidence the firmware rejects these writes; the new library's model taxonomy
is just incomplete, same conclusion as hotfix9/10.

Checked before bypassing: HR2044-2071 (the whole EMS scheduling block, including
2046/2049/2052 discharge, 2055/2058/2061 charge, 2064/2067/2070 export) is already
in Gate 2's universal `WRITE_SAFE_REGISTERS` set - the library's own author
considers these registers inherently safe to write, independent of model. Only
Gate 1's per-model check is what's excluding this device. This is the same
justification structure as hotfix9/10, not a new risk class.

**Explicitly NOT touched**, pending stronger evidence or explicit sign-off:
- `commands.set_soc_target`/`set_charge_target_only`'s *legacy* per-slot registers
  (HR242-298, the old non-EMS "Charge/Discharge Target SOC 1-10" block the old
  library also used) - these are **not** in Gate 2's universal safe set at all
  (only HR299 has been added, based on GivEnergy-app evidence per the library's
  own comments), so bypassing would mean skipping the universal PDU-level safety
  check too, not just the per-model one. Different, larger risk than the EMS-tier
  bypass above.
- `setEmsPlant`'s `commands.set_ems_plant(True/False)` (HR2040, `EMS_PLANT_ENABLE`)
  - also EMS-tier and also blocked by the same Gate 1 taxonomy gap, but this
  toggles the inverter's actual EMS-controlled operating mode rather than writing
  a target value; not currently exercised by Predbat or seen failing in any log
  from this system, so left alone rather than flipped speculatively.

Wired into the same 3 call sites: `sendAsyncCommand(reqs,readloop,bypass_model_gate=_is_hv_gen3(device))`.

## hotfix12: read-back for registers the library can write but can't decode for this model

Two gaps found after hotfix11 shipped and started writing successfully:

**1. EMS Target SOC read-back was never confirmed.** hotfix11's writes go through
`sendAsyncCommand(bypass_model_gate=True)`, which skips Gate 1 (the per-model write
check) but nothing else - but `plant.ems` (the only place the library exposes
HR2044-2071 as Python fields) is hard-gated to `Model.EMS`/`Model.EMS_COMMERCIAL` by
`model/plant.py`'s `ems` property, keyed off `device_type` directly, not a capability
flag. And the client's own `load_config()`/`refresh()` never even *polls* HR2040+
unless `caps.is_ems`. So there was no way, in the normal poll path, to tell whether
those writes actually stuck versus landing in a register nobody ever reads back.

**2. `Timeslots.Battery_pause_start_time_slot` went missing entirely once the louder
error was fixed.** Reported by Predbat: `KeyError: 'Battery_pause_start_time_slot'`
in its `adjust_pause_mode`, climbing steadily. Root cause: `ThreePhaseInverterRegisterGetter`
has *no* pause-slot field in its register map at all (`grep -n pause` on
`inverter_threephase.py` returns nothing) - only `SinglePhaseInverterRegisterGetter`
defines `battery_pause_slot_1` (HR319/320). Confirmed empirically:
`ThreePhaseInverter.from_register_cache(...).battery_pause_slot_1` returns `None`
unconditionally, regardless of what's actually in the cache. `getTimeslots()`'s
`GEInv.battery_pause_slot_1 is not None` guard was therefore always False for this
model, so the two pause-slot keys were silently never added - not a crash, just a
quietly-omitted key, which is exactly what a naive `dict[key]` access downstream
(Predbat) turns into a `KeyError`. This has nothing to do with hotfix11 specifically;
it's been true since the library rewrite. It only became the *visible* error once
hotfix11 fixed the louder `HR(2046) not permitted` rejection that was previously
dominating Predbat's error log.

Both are the same shape of problem as the write-side bug: the library's per-model
Python class is missing a field/poll for something HR-addressable and already
proven writable on this hardware. Fix is a raw diagnostic read, same technique as
the write bypass but for reads (which were never gated to begin with - Gate 1/2
only apply to writes):

- `_readEmsTargetDiag()` (new, in `read.py`) issues two extra `ReadHoldingRegistersRequest`s
  per refresh cycle when `not is_ems and device_type == Model.HYBRID_HV_GEN3`:
  `HR(2044,28)` for the EMS scheduling block, decoded field-by-field into the
  existing `EMS_Discharge/Charge_Target_SOC_N` / `Export_Target_SOC_N` key names;
  and `HR(319,2)` for the pause slot, decoded via `TimeSlot.from_repr()` (the same
  decoder `Converter.timeslot` uses internally, including its `60`-means-unset
  sentinel guard) rather than hand-rolling the bit layout.
- Wired into both refresh points in `watch_plant()` (cold-start and the main loop).
- `getControls()` merges the SOC keys in; `getTimeslots()` falls back to the pause-slot
  values only when `battery_pause_slot_1 is None` (i.e. only for models with no native
  field), preserving the original single-phase path untouched.
- Each of the two raw reads fails independently and never raises into the main
  refresh cycle - a timeout on one bank doesn't blank the other's last-good values,
  and `validateTimeslot()`'s existing last-known-good/midnight-default fallback
  covers the case where the diagnostic hasn't completed yet (e.g. first cycle after
  a restart).

## hotfix13: fix HA discovery crash introduced by hotfix12

hotfix12 broke MQTT discovery entirely on the next cold start:
`publish_discovery2: Error connecting to MQTT Broker: ('KeyError', 'HA_Discovery.py', 66)`.

Root cause: `getControls()` merged `_ems_target_diag` into `Control` with a blanket
`controlmode.update(_ems_target_diag)` - but that cache dict also holds
`Battery_pause_start_time`/`Battery_pause_end_time` (raw `datetime.time` values,
meant only for `getTimeslots()`'s internal `.get()` lookup, never for direct
publishing). Those un-suffixed keys have no `entity_lut.py` entry (only the
`..._time_slot`-suffixed pair does), and `HA_Discovery.py`'s topic-to-entity-type
lookup (`Entity_Type.entity_type[key]`) has no fallback for an unrecognised key -
it KeyErrors, and since the whole discovery publish runs as one MQTT connection
inside one try block, one bad key takes down every entity's discovery message for
that cycle, not just the one field.

Fixed by whitelisting exactly the 9 SOC keys `getControls()` should publish
(`_EMS_TARGET_SOC_KEYS`) instead of blanket-merging the diagnostic cache -
`Battery_pause_start_time`/`_end_time` stay internal to `_ems_target_diag`,
read only via `getTimeslots()`'s existing `.get()` calls.

## hotfix14: upgrade to givenergy-modbus 2.13.0 + upstream fork patch for legacy Target SOC

Two-part fix, both stemming from the same discovery: upstream's 2.13.0 release
(one version ahead of the 2.12.0 this addon had pinned since hotfix3) contains a
real-hardware-verified fix (dewet22/givenergy-modbus#412, hass#295) that
`Model.HYBRID_HV_GEN3` was misclassified as three-phase - it's actually
single-phase. Confirmed directly against this system's own live data before
touching anything: `SOC` was reading `0` while `SOC_kWh: 80.73` and the battery
stack's own `Stack_SOC_High/Low` read 62/61% - along with `PV_Power`, all
`PV_Voltage/Current_String_*`, `Grid_Phase1/2/3_Voltage`, `Load_Phase1/2/3_Power`,
`Export_Phase1/2/3_Power`, and `Battery_Charge/Discharge_Power` all flat zero. This
is the exact "live but idle plant" fingerprint from the upstream capture, present
on this system too, not just the similar unit upstream captured.

**Part 1: the reclassification itself (upstream, adopted as-is).** Bumped
`givenergy-modbus` 2.12.0 -> 2.13.0. This flips `plant.capabilities.is_three_phase`
to `False` for this model, which changes which top-level function
`processData()` dispatches to: `processInverterInfo()` instead of
`processThreePhaseInfo()` - a code path this system has never actually run
before (the misclassification put it on the three-phase path since day one of
the library rewrite). Audited before shipping, not assumed safe:

- Every `GEInv.*` attribute `processInverterInfo()`/`getControls()`/
  `getTimeslots()`/`getBatteries()`/`getInvModel()` touch verified against a real
  `SinglePhaseInverter` instance from 2.13.0 - all resolve; the one exception
  (`p_inverter_out`) is dead/commented code, never executed.
- Found and fixed 3 places that assumed `plant.number_batteries`/`plant.batteries`
  (LV-only, always 0/empty for this HV system - the exact hotfix5 finding,
  previously only fixed in the three-phase path) instead of using the HV-stack
  count like `getInvModel()`/`getBatteries()`/`getRaw()` already correctly did:
  `getInvModel()`'s `batmaxrate` calc (was gated on `is_three_phase` alone, now
  also fires for this model specifically via the correctly-resolved
  `plant.capabilities.device_type` - NOT `GEInv.model`, which only decodes the
  coarse family for this device, see `_is_hv_gen3()` in write.py), and two spots
  in `processInverterInfo()` (`Battery_Charge/Discharge_Energy_Total_kWh` and the
  entire `SOC`/battery-power block, which was skipping `SOC` entirely rather than
  just reading it wrong).
- Deliberately scoped narrowly: `getInvModel()`'s fix checks `device_type ==
  Model.HYBRID_HV_GEN3` specifically rather than widening to `is_hv` generally,
  because `Model.ALL_IN_ONE` is also `is_hv` and must keep its existing flat-6000
  `batmaxrate` path unchanged - not verified/tested and out of scope here.
- `getTimeslots()`'s pause-slot fallback (hotfix12/13) and its `EMS_target_diag`
  reads (hotfix12) are unaffected either way: `SinglePhaseInverter` natively has
  `battery_pause_slot_1` (unlike `ThreePhaseInverter`), so the hotfix12/13 raw-read
  path simply stops being exercised for this model rather than needing removal.
- `write.py`'s phase branching (`if "3ph" in GiV_Settings.inverter_type.lower()`)
  is driven by a stored GivTCP config value, not `plant.capabilities.is_three_phase`
  - unaffected by this change either way. That config value was independently
  found to already be wrong for this system (`Model_1 = All_in_one` in
  `/settings`, the same coarse-decode issue as `GEInv.model` above) but left
  alone - a pre-existing, unrelated display/config issue, not something this
  library bump introduces or fixes.

**Part 2: legacy Target SOC write-safety (our own patch, forked from 2.13.0).**
2.13.0's reclassification does not by itself fix the actual bug this session was
chasing (Predbat's `setDischargeTarget` succeeding at the Modbus layer via the
hotfix11 EMS-tier bypass but never changing `Control.Discharge_Target_SOC_1`,
because that field reads from a completely different, legacy register HR272
that the EMS-tier write never touches). Gate 1 (`write_safe_registers()`) never
included the legacy per-slot Target SOC registers (HR242-269 charge, HR272-299
discharge) for any model; Gate 2 only had HR299 (added under upstream's
original app-inventory sweep, #48).

Forked `dewet22/givenergy-modbus` -> `Texelo/givenergy-modbus`, branch
`hv-gen3-legacy-target-soc` off the `v2.13.0` tag. Re-auditing
`docs/reference/registers/app_4.0.7_inventory.json` directly (not assumed)
found the other 19 registers in this family equally present in the app's own
inventory (`model_codes` in that file confirms `0x8103` = `"GIV-HY-10.0-G3-HV
10KW"`, i.e. this exact model) - an omission in the original sweep, not a gap
in evidence. Added all 20 to Gate 2 (alongside HR299, same evidence tier) and a
new `WRITE_SAFE_HYBRID_HV_GEN3_LEGACY_TARGETS` set to Gate 1, unioned only for
`Model.HYBRID_HV_GEN3` specifically - the app inventory isn't per-model scoped,
so it doesn't by itself establish every extended-slot model (HYBRID_GEN4,
ALL_IN_ONE, ALL_IN_ONE_HYBRID) supports every register in this family; only
this one was cross-checked. Corroborated further by the pre-rewrite vendored
client (`givenergy_modbus_async`, no write-safety gating of any kind) having
used this exact register family against this exact hardware for an extended
period, per this system's own working history before the library rewrite.

Full upstream test suite (1666 tests) and ruff pass against the patch. Added two
new tests pinning the exact register set and that it doesn't leak to other
extended-slot models; updated the existing 1:1 write-surface fence test (#412)
for the new union. Regenerated the app-reconciliation baseline via the
project's own documented script - clean +20 diff, no other changes. Pushed to
`Texelo/givenergy-modbus`, branch `hv-gen3-legacy-target-soc`
(https://github.com/Texelo/givenergy-modbus/pull/new/hv-gen3-legacy-target-soc)
- worth opening as a real PR upstream given the evidence, not done yet pending
confirmation this actually resolves the live issue.

**How it's wired into this addon**: not installed via pip from the fork (avoids
a build-time network/git dependency after already being burned once by
scarf.sh rate-limiting). `requirements.txt` still pins the plain PyPI
`givenergy-modbus==2.13.0`; the two patched files
(`model/manifest.py`, `pdu/write_registers.py`) are vendored into this repo
under `vendor_patches/` and `COPY`'d over the pip-installed package in the
`Dockerfile`, after `pip install`. Same diff either way, no extra build-time
dependency.

**Not yet done**: the write.py bypass calls (`bypass_model_gate=_is_hv_gen3(device)`)
for the three EMS Target SOC functions (hotfix11) were left as-is rather than
switched to the now-correctly-permitted legacy registers - untangling which of
EMS-tier vs legacy-per-slot is what Predbat/the real inverter actually need is
the next thing to verify against live behaviour before touching write.py again.

## hotfix15: retry writes instead of failing on a single timeout; stop swallowing blank exceptions

Diagnosed via a live log entry: `Setting Discharge Target 1 failed: ('Exception',
'Error in write command: ', 'write.py', 445)` - note the blank message, nothing
after the colon. Traced to `sendAsyncCommand()`'s `except Exception as e:
output['error']="Error in write command: "+str(e)`: the caught exception's
`str()` was empty, consistent with a bare `TimeoutError()` (that's what
`asyncio.wait_for()`/the library's own timeout machinery raises with no args) -
not a register/permission problem (those raise `InvalidPduState` with a specific
`"HR(N) is not permitted"` message, distinguishable from this).

Root cause: both write paths in `sendAsyncCommand()` were calling into the
library with `retries=0` - the bypass path explicitly (my own hotfix9 code), the
gated path implicitly (`one_shot_command()`'s own default). The library's
`send_request_and_await_response()` docstring explains it added a
`retry_delay`-backed retry specifically "to overcome the multi-second
silent-window failure mode observed in the field" - protection that never ran
with `retries=0`. A transient Modbus timeout that a retry would absorb instead
surfaced immediately as a write failure.

Two changes: `retries=2` (library default `retry_delay=0.5` backoff) on both
`one_shot_command()` and `send_request_and_await_response()` calls - safe to
retry blindly since every write here is idempotent (setting a target
percentage/timeslot to a specific value has the same effect whether sent once
or three times, unlike an increment/toggle). And a fallback in the except
block - `str(e) or type(e).__name__` - so a future blank-message exception at
least names its type instead of being a dead end to diagnose from the log alone,
same spirit as hotfix8's original error-surfacing fix but covering the case
where the underlying exception itself has no message text.

## hotfix16: Phase 1 network-hardening (from the cowork analysis)

Folds in the low-risk items from a network-contention analysis done in parallel
(a second Claude instance with Home Assistant MCP access, cross-checking the
addon's own connection architecture and the vendored library). It independently
found the same root cause hotfix15 already fixed (zero-retry writes / the blank
"Error in write command: " message) and proposed a refinement worth adopting on
top, plus two more low-risk items:

- **`queue_retries` wired up for real.** hotfix15 hardcoded `retries=2` on both
  `sendAsyncCommand()` write paths. `queue_retries` was an existing
  `settings_template.py` field ("the number of calls to the inverter when
  trying to set a register", default 2) that was dead code - never read
  anywhere outside the template. Now used for real, with the hardcoded `2` as
  a fallback if the setting is ever absent/invalid. This system already has it
  set to `4` (confirmed live), so this is a free improvement with no code
  change needed to take effect - just picks up the value that was already
  configured and previously ignored.
- **Loosened `_readEmsTargetDiag()`'s two diagnostic reads** (hotfix12) from
  `timeout=1.5, retries=1` to `timeout=3, retries=2` - closer to (though still
  below) `watch_plant()`'s own refresh defaults (`timeout=3, retries=5`).
  These are debug-only supplementary reads that count toward the same
  per-cycle read set as the main refresh, so their tighter budget was a likely
  contributor to the "X of 7 register reads failed" log noise seen after
  hotfix13 turned debug logging on.
- **Gunicorn REST workers: 3 -> 1** (`startup.py`, both the initial spawn and
  the restart-on-crash branch). Each REST worker is a separate OS process, and
  each opens its own independent Modbus TCP connection on demand
  (`sendAsyncCommand` -> `GivClientAsync.get_connection()`) - the
  `asyncio.Lock` guarding that in `GivLUT.py` only serialises coroutines
  *inside one process*, so 3 workers could open 3 connections concurrently, on
  top of the read loop's own already-open one. GivEnergy dongles are widely
  reported to tolerate only one active Modbus session - this doesn't eliminate
  contention with the read loop (that needs a real architectural fix, tracked
  separately, not done here), but it removes REST-workers-vs-each-other as an
  additional source of it. Trade-off: a slow/hung write now blocks the next
  REST request instead of another worker picking it up - accepted given
  writes are infrequent and Predbat's own REST client already retries with
  backoff on its side.

**Deliberately not done in this pass** (per the same analysis, correctly
scoped out): generalizing the Gate-1 write-safety bypass beyond HV Gen3 for
other inverter models (doesn't affect this system, which already has its own
`_is_hv_gen3()` bypass), and the larger structural fix for read-loop/REST
connection contention (routing every write through the read loop's own
already-open connection instead of a second process opening a competing one)
- correctly identified as the real fix but too large/risky to ship without
first seeing whether the changes above meaningfully reduce lockout frequency
on their own.

## hotfix17: actually write Charge/Discharge Target SOC to the register Predbat reads back

The register-permission gap (hotfix14) and the register write.py actually writes
to were two separate bugs, and only the first one was fixed until now.
`setChargeTarget2`/`setDischargeTarget` were still calling
`commands.set_ems_charge/discharge_target_soc()`, writing to the EMS-tier block
(HR2044-2071) via the hotfix9-11 model-gate bypass. But `getTimeslots()`
(read.py) populates `Control.Charge/Discharge_Target_SOC_N` - what Predbat
validates its own writes against - from the completely different legacy
per-slot registers (HR242-269/272-299) that hotfix14's write-gate patch
targeted. Confirmed live before fixing: right after a restart,
`Discharge_Target_SOC_1` read back `10`, not whatever had last actually been
requested - the EMS-tier write was landing somewhere nothing reads back for a
non-EMS system, exactly as flagged (but not yet confirmed) when hotfix14
shipped.

Added `_legacy_target_soc_request()`: builds a raw `WriteHoldingRegisterRequest`
for the correct register (`242 + 3*(slot-1)` charge, `272 + 3*(slot-1)`
discharge - matching the derivation `hass#295`'s own fixture test uses for the
sibling start/end slot registers), since no `commands.py` helper exists for
this register family (only the EMS-tier one does). Replicates the `[4,100]`
bounds check the EMS-tier command functions had, since the PDU layer itself
doesn't validate the value, only that it's a valid uint16. Wired into both
`setChargeTarget2` and `setDischargeTarget`, gated on `_is_hv_gen3(device)`
exactly like the existing pause-mode/EMS bypasses - falls back to the prior
EMS-tier behaviour if the device ever isn't detected as HV Gen3, rather than
crashing.

**`setExportTarget` intentionally left unchanged** - there is no legacy
register for Export Target SOC in either the old or new library (confirmed
earlier this session: it was always HR2062-2071, EMS-tier, even in the
pre-rewrite vendored client). Whether Export Target actually works for this
non-EMS system is still an open question, separate from this fix.

Verified all 20 slot->register mappings (both charge and discharge, all 10
slots) against the real patched library before shipping: every one passes
`ensure_valid_state()` (Gate 2) and resolves to the expected `HR272-299` /
`HR242-269` address.

## hotfix18: getRaw() missing the invalid-BMU guard getBatteries() already has

Live failure loop: every read cycle threw `TypeError: '<' not supported
between instances of 'NoneType' and 'str'` while serializing
`raw['HV_Battery_Stacks']['Stack_0']`, and the same error broke `/readData`
(`getCache()`) - the endpoint Predbat polls via `givtcp_rest` - meaning
Predbat was failing to read from GivTCP on every poll.

`getRaw()` and `getBatteries()` both loop over each stack's BMUs with the
same pattern:
```python
if b.is_valid():
    sn=b.serial_number
else:
    sn=b.serial_number
```
- a no-op `if/else` (identical in both branches). `getBatteries()` (the
correct sibling) follows this with a real guard,
`if sn and sn.upper().isupper():`, skipping invalid/serial-less BMUs before
ever using `sn` as a dict key (logging "Battery Object empty so skipping").
`getRaw()` never got that guard - it did `stack[sn]=b` unconditionally, so
a currently-invalid BMU (confirmed happening live, matching
"Battery Object empty so skipping" from `getBatteries()` in the same cycle)
landed in the dict with key `None` next to the other stacks' string keys,
and `json.dumps(..., sort_keys=True)` can't compare `None` to `str`.

Fixed `getRaw()` to use the same skip-and-log guard `getBatteries()` already
uses, and dropped the dead `is_valid()`/`else` no-op (the real check is on
`sn` itself, matching the sibling function exactly). Logged at `debug` level
rather than `error` since `getRaw()`'s output is a diagnostic/raw dump, not
the primary battery data path `getBatteries()` already logs the same event
for at `error` level in the same cycle.

## hotfix19: the "reboot inverter" button/switch has never actually rebooted the inverter

Live-investigated during an active outage (Modbus TCP port 8899 refusing every
connection while ICMP ping to the dongle stayed up - GivTCP's own
"Restarting container to detect IP change" auto-recovery kept firing every
~70-90s without fixing anything, since it only restarts this addon's own
process/container - never touches the inverter itself - and does so on
`host_network: true`, so even a full container restart doesn't reset any
networking state either).

This raised the question of why Predbat's own `auto_restart` sequence
(step 1: `switch.turn_on` on `switch.givtcp_{geserial}_reboot_invertor`) would
be expected to help, since - if it worked - it's the only step of the three
that reaches the *inverter* rather than just this addon's container. Traced
it and found it never has worked, for either the button or switch variant of
this control:

- `entity_lut.py`'s `Reboot_Invertor` declared its MQTT/button command name
  as `"rebootInverter"` (capital I).
- `write.py`'s actual function is `rebootinverter` (lowercase i).
- `read.py`'s write-command dispatcher (`if hasattr(write, command[0]):`) is a
  **silent** guard - a name that doesn't resolve just drops the command, no
  error, no log line, nothing.
- `hasattr(write, "rebootInverter")` is always `False`, so every reboot
  request sent via MQTT (the button, or the separate stale
  `switch.givtcp_{geserial}_reboot_invertor` entity left over from an older
  GivTCP version and still targeted by Predbat's `apps.yaml` today) has been
  silently dropped this entire time - the inverter never actually reboots.
- `REST.py`'s `/reboot` endpoint was already correct (calls
  `requestcommand("rebootinverter")`, lowercase, matching `write.py`) - the
  bug was scoped to the MQTT-triggered button/switch path only.

Fixed by changing `entity_lut.py`'s declared command name and `mqtt.py`'s
matching `elif` branch to `rebootinverter` (lowercase), matching the three
already-correct references (`write.py`'s function, both `REST.py` lines)
rather than renaming the function itself. `Reboot_Addon` was checked too and
found consistently cased everywhere already (`rebootAddon`) - not affected.

**Not yet resolved**: whether an actual inverter reboot (now that the control
path works) fixes this specific "Modbus port refuses connections, ping still
answers" failure mode is a live open question, being tested directly against
the outage this was found during, not assumed from the code fix alone.

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
