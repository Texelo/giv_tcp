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
