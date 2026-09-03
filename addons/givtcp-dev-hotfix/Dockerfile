# GivTCP-DEV 3.5.47 hotfix
#
# Root cause (see README.md in this folder for the full diagnosis):
# the 3.5.47 dev image bumped requirements.txt to the rewritten `givenergy-modbus==2.12.0`
# library (https://github.com/dewet22/givenergy-modbus), which renamed Plant.HVStack ->
# Plant.hv_stacks and changed each stack entry from an indexable [bcu, [bmus]] pair to a
# proper HvStack object (.bcu / .bmus attributes), plus a couple of related field renames
# (number_of_module -> number_of_modules, battery_nominal_capacity -> *_ah,
# remaining_battery_capacity -> *_ah) and dropped the old dict-like .get()/.getall()/.getsn()
# methods in favour of plain pydantic attribute access / model_fields.
#
# GivTCP/read.py was only partially migrated to the new API, so any HV battery stack
# (HYBRID_HV_GEN3 etc.) crashes on every single read cycle with:
#   AttributeError: 'Plant' object has no attribute 'HVStack'
# which cascades into an UnboundLocalError in runAll2() that masks the real cause.
#
# Rather than reconstruct the whole app from a git branch (none of the public branches
# in this repo match what's actually shipping in the 3.5.47 dev image - it depends on a
# library version none of them use), this builds FROM the exact upstream 3.5.47 image and
# layers on a corrected GivTCP/read.py. Nothing else changes.
FROM givtcp.docker.scarf.sh/britkat/giv_tcp-dev:3.5.47
COPY read.py /app/GivTCP/read.py
