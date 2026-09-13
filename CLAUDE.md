# Genli — hardware for Revali

KiCad projects for a **hopcopter**: a quadrotor with a passive spring-loaded
telescopic leg that hops along the ground instead of hovering. This repository
is boards only. Firmware and all design documentation live in
[eli-lame/Revali](https://github.com/eli-lame/Revali).

## The design docs are the source of truth, and they are in the other repo

Board decisions are recorded in Revali's `docs/`, not here and not in chat
history. Read the relevant one before changing a board, and update it in the
same change when a decision moves.

| Doc (in Revali) | Authoritative for |
|---|---|
| `docs/hardware.md` | **The pin map.** Buses, sensor quirks |
| `docs/hardware_roadmap.md` | Board architecture, the two-board stack, power topology, staging |
| `docs/safety.md` | Arming, failsafes, the voltage thresholds the divider feeds |
| `docs/current_devtasks.md` | Phase H — the hardware task list |

`docs/hardware.md` wins any disagreement about a GPIO number. **Never copy the
pin table into this repository** — link to it. Two copies is how a board and
its firmware quietly diverge, and they are already in separate repositories.

When gerbers are generated, record the Revali commit the board was designed
against in that board's `production/` directory. It is the only link back from
a fabricated board to the pin map it assumed.

## Layout

One KiCad project per board, side by side.

- `lib/` — symbols and footprints shared by every board
- `v1-prototype/` — first board, archived, never fabricated
- `stage2-carrier/` — current work; its `README.md` is the full schematic spec

A new board is a **new KiCad project**, not a copy. Copying inherits the
previous netlist — v1's contains a reversed `ESC1`–`ESC4` mapping and a power
section built around a part no longer used. Reuse footprints, not net topology.

## Library rules

Every custom or third-party symbol and footprint is **vendored** in `lib/` and
registered per project with `${KIPRJMOD}`-relative paths, never installed
globally. A globally installed library is invisible to the repository.

Library **nicknames are load-bearing**: assignments are stored as
`Nickname:Name` throughout every sheet and board. `HopCopter` and `Genli` are
leftovers from earlier names for this project — moving the files is safe,
renaming a nickname breaks every assignment. Do not rename them.

Introduce no new Plugin-and-Content-Manager dependencies; vendor anything that
is not a KiCad stock library. See `README.md` for the registration table and
the one known gap.

## Current state

Stage 2 is a carrier board: it replaces roughly fifty wires with a PCB using
only parts already proven on the breadboard. No bare chips, no fine pitch,
nothing that cannot be reworked with an iron. `stage2-carrier/README.md` is the
net-by-net spec, BOM and bring-up order — **read it before drawing anything.**

Deliberately not on that board: motor current (a Skystars KO50A 4-in-1 carries
all of it), the ToF sensors themselves (they mount out at the frame — the
estimator needs the baseline), and ELRS (reserved footprint only, since the
link stays ESP-NOW for this hardware generation).

Analog inputs go on **ADC1** only — ADC2 is unusable while the WiFi radio is
active, and with ESP-NOW it always is.

## Working rules

1. **Props off** for every bench and bring-up step.
2. **One change at a time**, verified before the next. This applies to moving
   project files as much as to circuit changes.
3. **Print footprints 1:1 and check them against the physical part** before
   ordering. Devkit header row spacing varies between clones; a mismatched
   footprint makes the board scrap on arrival.
4. **Never skip a bring-up step.** The order in the spec exists so each failure
   is cheap and isolated.
