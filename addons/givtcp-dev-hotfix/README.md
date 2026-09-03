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

- `getRaw()`, `getBatteries()`: `plant.HVStack` -> `plant.hv_stacks`, `stack[0]`/`stack[1]`
  -> `stack.bcu`/`stack.bmus`, renamed `Bcu` fields, `Bmu.get(...)` -> `getattr(...)`,
  `.getsn()` -> `.serial_number`.
- `getall()` helper: removed a `model.to_dict()` call that doesn't exist on `Bcu`/`Meter`
  pydantic models (this broke `getRaw()`'s meter dump too, independent of the HVStack bug).
- `runAll2()`: `except KeyError` now returns instead of falling through to the
  `UnboundLocalError`.

All renamed field names and method removals were verified directly against the installed
`givenergy-modbus==2.12.0` package (`model_fields` introspection), and the patched HV
battery-stack code path was exercised against real `Bcu`/`Bmu`/`HvStack` instances before
this was built. See the diagnosis conversation for the full trace.

## How this is packaged

None of the branches in this repo (`main`, `dev3`, `modbusv2`) match what's actually
shipping in the 3.5.47 dev image — they're all on an older/different library. Rather than
guess at reconstructing the full app from source, this Dockerfile builds `FROM` the exact
upstream `givtcp.docker.scarf.sh/britkat/giv_tcp-dev:3.5.47` image and layers on a
corrected `GivTCP/read.py`. Nothing else changes.

## Installing

Add `https://github.com/Texelo/giv_tcp` (branch `hotfix/3.5.47-hvstack-api`) as a custom
repository in Home Assistant → Settings → Add-ons → Add-on Store → ⋮ → Repositories, then
install **GivTCP-DEV (Texelo hotfix)**. It builds locally on your HA host from this
Dockerfile the first time you install it.
