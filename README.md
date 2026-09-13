# Genli — hardware

KiCad projects for the **Revali** hopcopter: a quadrotor with a passive
spring-loaded telescopic leg that hops along the ground instead of hovering.

Firmware, and all design documentation, live in the companion repository
**[eli-lame/Revali](https://github.com/eli-lame/Revali)**. This repository is
boards only.

## Layout

```
lib/                            symbols and footprints shared by every board
  HopCopter.pretty/               our own footprints
  Genli.kicad_sym                 our own symbols
  ESP32_Footprints.pretty/        third party, MIT
  esp32_30pin_revised.kicad_sym   third party, MIT
  ESP32-DevKit-V1-DOIT.LICENSE    its licence — keep it with the files
v1-prototype/                   first board — archived, never fabricated
stage2-carrier/                 current work — README.md is the spec
```

One KiCad project per board, side by side. A new board is a **new project**,
not a copy of the previous one: copying inherits the previous netlist, and
v1's contains a reversed `ESC1`–`ESC4` mapping and a power section built around
a part no longer used. Footprints are worth reusing; net topology is not.

## The pin map lives in Revali

[`docs/hardware.md`](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md)
is the single source of truth for every GPIO assignment, and it wins any
disagreement. **Do not copy the pin table into this repository** — link to it.
Two copies of a pin map is how a board and its firmware quietly diverge.

Other documents worth reading before changing a board:

| Doc | Covers |
|---|---|
| [hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md) | **The pin map**, buses, sensor quirks |
| [hardware_roadmap.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware_roadmap.md) | Board architecture, the 4-in-1 stack, power topology, the staging plan |
| [safety.md](https://github.com/eli-lame/Revali/blob/main/docs/safety.md) | Arming, failsafes, the voltage thresholds the divider feeds |
| [current_devtasks.md](https://github.com/eli-lame/Revali/blob/main/docs/current_devtasks.md) | Phase H — the hardware task list |

Because the boards and the firmware are in separate repositories, **record the
Revali commit** a board was designed against in its `production/` directory
when gerbers are generated. That is the only link back from a fabricated board
to the pin map it assumed.

## Libraries are vendored, not installed

Every custom or third-party symbol and footprint lives in `lib/` and is
registered **per project**, never globally. A globally installed library is
invisible to the repository: the project opens on the machine that installed it
and nowhere else. v1 had exactly this problem — its symbols were referenced by
an absolute path into `OneDrive/Documents/`, so a clone of this repo could not
resolve them.

Register in *Preferences → Manage Symbol / Footprint Libraries → Project
Specific Libraries*, always with `${KIPRJMOD}`-relative paths:

| Nickname | Path |
|---|---|
| `HopCopter` | `${KIPRJMOD}/../lib/HopCopter.pretty` |
| `ESP32_Footprints` | `${KIPRJMOD}/../lib/ESP32_Footprints.pretty` |
| `Genli` | `${KIPRJMOD}/../lib/Genli.kicad_sym` |
| `esp32_30pin_revised` | `${KIPRJMOD}/../lib/esp32_30pin_revised.kicad_sym` |

`fp-lib-table` and `sym-lib-table` in each board directory already contain
these; they are committed, so a fresh clone resolves without any setup.

**The nicknames are load-bearing — do not rename them.** Assignments are stored
as `Nickname:Name` across every sheet and board, so a nickname is a hard
reference. `HopCopter` and `Genli` are leftovers from earlier names for this
project; renaming them breaks every existing assignment for no benefit. Moving
the *files* is safe; renaming the nickname is not.

If a nickname is also in your global table, remove the global entry —
duplicates across the two tables conflict.

### Known gap

v1's PCB references `PCM_SparkFun-Connector:ScrewTerminal_1x02_P5.0mm`, from a
library installed through KiCad's Plugin and Content Manager, which is not
vendored here. v1 is archived and that screw terminal is deleted in Stage 2, so
this is recorded rather than fixed. **Introduce no new PCM dependencies** —
vendor anything that is not a KiCad stock library.

## Moving or renaming a project

One change at a time, opening the project between each. A KiCad project is
`<name>.kicad_pro`, `<name>.kicad_sch` and `<name>.kicad_pcb`, and the root
schematic must share the project's base name — so renaming means renaming all
three consistently. Use *File → Save As*, which does it correctly, rather than
renaming files by hand.

`v1-prototype/` deliberately keeps its original `Genli.*` file names. The
directory name carries the meaning, and renaming is the riskiest operation
available for no gain on a board that will never be edited again.

## What is not committed

`.gitignore` excludes KiCad's per-user and regenerable files: `*-backups/`,
`*.kicad_prl`, `fp-info-cache`, `*.net`, autosave and lock files, and
`.history/` from the VS Code local-history extension.

Fabrication outputs — gerbers, drill files, BOM, pick-and-place — go in a
`production/` subdirectory of the board they belong to and **are** committed.
They are the exact bytes sent to the fab, which is worth being able to
reproduce.
