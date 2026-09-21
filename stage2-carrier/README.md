# Stage 2 — carrier board

The first custom PCB. Its job is to replace roughly fifty wires with a board,
using only parts that already work, so that the thing being learned is the PCB
pipeline and nothing else. No bare chips, no fine pitch, nothing that cannot be
reworked with an iron.

Context and rationale: [hardware_roadmap.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware_roadmap.md).
Pin map source of truth: [hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md). Build tasks:
`[H.5]`–`[H.10]` in [current_devtasks.md](https://github.com/eli-lame/Revali/blob/main/docs/current_devtasks.md).

**Deliberately not on this board:** motor current, the ToF sensors themselves
(they mount out at the frame — the estimator needs the baseline), and ELRS
(reserved footprint only). A discrete switching regulator waits for Stage 3.

---

## Board

| | |
|---|---|
| Layers | 2, 1 oz copper, 1.6 mm FR4 |
| Outline | **58 × 88 mm** |
| Ground | **Single plane on `B.Cu`.** No top pour — see below |
| Mounting | 4 × on a 30.5 × 30.5 mm square, sized to the KO50A's soft-mount grommets |
| Assembly | Hand-soldered, all parts on the top side |

### Ground is a single plane on the bottom

`B.Cu` carries one continuous GND zone. **There is no `F.Cu` pour.**

The top layer is dense enough with routing that a top pour filled only in
patches, with large bare regions between traces, and several ground pads it was
supposed to serve were not actually being reached. A pour in fragments is not a
plane; it just makes connectivity depend on whether copper happened to find its
way somewhere.

So every ground connection here is explicit:

- **Through-hole ground pads** reach the plane through their own plated barrels
- **Every SMD ground pad** has a short, wide stub to its own via, dropping
  straight down to the plane

That via-per-pad is best practice regardless of whether a top pour exists — the
return current from a decoupling capacitor wants the shortest possible loop into
the plane, not a wander across the top layer looking for a via. The pour was
hiding that the work had not been done.

A top pour could be added back as a pure bonus now that nothing depends on it.
It is not needed, and it is not fitted.

### Why the board is this big

The 30-pin ESP32 devkit is about **52 × 25 mm** — longer than the 36 × 36 mm a
standard 30.5 flight controller occupies. It does not fit on a conventional FC
outline, so the board grows and overhangs the stack fore and aft.

That is an acceptable trade at this stage and arguably a benefit: a roomy
2-layer board is dramatically easier to route, and every component stays
reachable for rework. Weight optimisation is Stage 3's job, and Stage 3 gets it
by dropping the devkit for a bare module.

It ended up at **58 × 88 mm**, which overhangs the KO50A (41 × 46 mm) on every
side and leaves roughly 29 mm of unsupported board past the mounting square at
the long ends.

**Check that outline against the actual frame before ordering**, and keep the
heavy parts — `C10` and `U2` — near the 30.5 mm mounting square rather than out
on the overhang. `C10` is an 8 mm electrolytic on a vehicle designed to land
hard: **stake it with epoxy** at assembly.

### The footprint check that catches the classic first-board failure

Devkit header spacing is **not consistent between clones** — 0.9″ and 1.0″ row
spacing both exist under the same "30-pin ESP32 devkit" name. A library
footprint that does not match the board in your hand produces a PCB that is
scrap on arrival.

Before ordering: **print the footprint 1:1 on paper and lay the physical devkit
on top.** Check row spacing, pin count, and overall length. Do the same for the
Pololu module and the JST connector. This takes five minutes and is the
single highest-value check in the whole process.

---

## Connectors

Designators below are **as built** — they come from the board, not from an
earlier draft of this document.

| Ref | Type | Purpose |
|---|---|---|
| `J3` | JST-SH 1.0 mm, 8-pin | ESC ribbon to the KO50A |
| `U1` | 2 × 1×15 female header, 2.54 mm | ESP32 devkit socket |
| `U3` | SEN-15335 footprint | IMU (ICM-20948 breakout), soldered down |
| `J4` | 1×6 header, 2.54 mm | ToF A (front-left) — sensor mounts at the frame |
| `J5` | 1×6 header, 2.54 mm | ToF B (rear-right) — sensor mounts at the frame |
| `J2` | 1×4 header, 2.54 mm | **ELRS / CRSF — reserved, unpopulated** |
| `BZ1` | 1×2 header, 2.54 mm | Buzzer — passive piezo, mounts off-board |

There is no `J1`. The spare-GPIO header was deleted during layout, and the
**FC power switch on `SHDN` was never fitted** — `SHDN` is left no-connected, so
the regulator is always enabled whenever the pack is plugged in. The battery
connector remains the only disconnect, which was always the design.

### J3 — ESC ribbon (KO50A)

| Pin | Signal | Net | ESP32 |
|---|---|---|---|
| 1 | GND | `GND` | — |
| 2 | BAT | `BAT` | — (buck + divider only) |
| 3 | S1 | `ESC1` | GPIO 32 |
| 4 | S2 | `ESC2` | GPIO 33 |
| 5 | S3 | `ESC3` | GPIO 27 |
| 6 | S4 | `ESC4` | GPIO 14 |
| 7 | NC | — | leave unconnected |
| 8 | CURR | — | **no-connect flag** — current sense is not fitted, see below |

GPIO 32/33/27/14 are Motor 1/2/3/4 = front-left / front-right / rear-right /
rear-left, matching the mixer. **Which physical motor the ESC's own output 1
drives is decided when you solder the motors to the ESC**, not by this board —
so the mapping still has to be verified empirically in `[H.3]`. The v1
schematic got this backwards; do not assume.

---

## Power

```
J3.2 BAT ──┬── F1 ──┬── D2 (TVS) ──┬── C10 ──┬── C9 ── U2 VIN ── U2 VOUT ── 5V
           │        │              │         │
          GND      GND            GND        └── R2 ── divider ── GPIO 34

  U2 SHDN ── no-connect       regulator always enabled
  U2 PG   ── no-connect

  5V ──┬── devkit VIN (C1 22µF + C3 100nF) ── AMS1117 ── 3.3V ── IMU, ToF ×2
       ├── J2.1 (ELRS, reserved)
       └── R8 ── D1 power LED
```

| Ref | Part | Notes |
|---|---|---|
| `U2` | Pololu D24V10F5 | 5.1–36 V in, 5 V @ 1 A. **Five pins** — `VIN`, `GND`, `VOUT`, `SHDN`, `PG` at 0.1″ spacing. Board is 18 × 13 × 3.5 mm with no mounting holes. Solder **flat** through the pads, not into headers — a socketed module works loose under impact. Stake with epoxy |
| `D2` | **SMBJ20CA** TVS (bidirectional) | 20 V standoff (clear of 16.8 V full charge), ~32 V clamp — under the Pololu's 36 V limit. **4 S only**; 6 S would sit above the standoff voltage. Bidirectional, so it has no polarity and cannot be fitted backwards — see below |
| `C10` | 100 µF 35 V low-ESR electrolytic | Pololu's own docs warn that leads longer than a few inches create an LC spike at power-up that can exceed the module's rating. The ribbon plus trace run qualifies. `C9` 100 nF sits beside it, closest to `U2`'s VIN pin |
| `C1` + `C3` | 22 µF + 100 nF on `5V` | at the devkit's VIN pad |
| `C4` + `C5` | 10 µF + 100 nF on `3.3V` | at the devkit's 3.3V pad, plus 100 nF at each sensor (`C6`, `C7`, `C8`) |

Nothing else connects to `BAT`. It is 16.8 V at full charge and **must never
reach the devkit's `VIN` pin**, whose AMS1117 is rated to roughly 15 V.

### Why the TVS is bidirectional

`SMBJ20CA` rather than the unidirectional `SMBJ20A`. Same package, same price,
same clamping behaviour for the positive transients this board actually sees —
and with no polarity, it cannot be soldered backwards.

That matters because a *reversed* unidirectional TVS is forward-biased across
the supply at ~0.7 V. It is a permanent short on the battery, it looks like a
dead board, and there is nothing on the silkscreen to distinguish a correctly
fitted part from a reversed one once it is soldered. Removing that failure mode
is worth more here than what the unidirectional part buys.

What the unidirectional part would have bought: together with `F1` it forms an
accidental reverse-polarity protection — swap `BAT` and `GND` and the TVS
conducts, blowing the fuse and saving the board. With `J1` keyed, reverse
polarity at the connector is already close to impossible, so that benefit is
largely theoretical.

KiCad's `Device:D_TVS` symbol is already the bidirectional one, so no schematic
change is needed. The `Diode_SMD:D_SMB` footprint carries a cathode band on its
silkscreen; on a bidirectional part that marking means nothing. Harmless, but
worth knowing before someone at assembly tries to orient it.

### The regulator's own symbol and footprint

There is no stock KiCad symbol, and a community one is not worth trusting for a
part this simple. Build it from Pololu's own **Resources** on the
[product page](https://www.pololu.com/product/2831): the dimension diagram
gives the pad positions and the pin order as silkscreened, the drill guide
prints 1:1 as a physical check, and the STEP model drops into the footprint's
3D view for stack clearance.

**Read the pin order off the dimension diagram, not from memory** — the labels
are printed on the *back* of the module. Symbol pin numbers must equal footprint
pad numbers; the names are only labels.

| Pin | Electrical type | Why |
|---|---|---|
| `VIN` | Power input | |
| `GND` | Power input | KiCad's convention for a module's ground |
| `VOUT` | **Power output** | Makes it the driver of the `+5V` net, which is what stops ERC complaining that `+5V` has no source |
| `SHDN` | Input | Control input |
| `PG` | Open collector | Open drain; needs a pull-up wherever it is read |

Because `BAT` and `GND` arrive from a connector, whose pins are passive, ERC
will report *"power input pin not driven by any power output"* on those nets.
That is expected — drop a **`PWR_FLAG`** on `BAT` and on `GND` to tell ERC they
are externally sourced.

### Battery divider — GPIO 34

```
BAT ──[ R2 100 kΩ 1% ]──┬──[ R1 22 kΩ 1% ]── GND
                        ├──[ C2 100 nF ]── GND
                        └── GPIO 34 (ADC1_CH6, input-only)
```

16.8 V × 22/122 ≈ **3.03 V** at full charge, under the 3.3 V ceiling with room
for the ADC's nonlinearity near the rail. Quiescent draw ~138 µA. Use 1 %
resistors — divider accuracy sets voltage accuracy directly, and this is the
number `warn_voltage` / `land_voltage` / `critical_voltage` in
[safety.md](https://github.com/eli-lame/Revali/blob/main/docs/safety.md) are checked against. Calibrate against a
meter — task `[2.17]` / `[H.10]`.

### Current sense — not fitted

**`J3` pin 8 carries the KO50A's current-sense output and is deliberately left
unconnected on this board.** No trace, no components, a no-connect flag on the
connector pin. GPIO 35 stays free.

Voltage sensing is what the failsafe actually requires, and that is populated
and working. Current sense would only buy joules-per-hop, which is a nice number
for validating the efficiency premise but changes no in-flight decision.

Two reasons not to route it speculatively. Skystars does not publish the KO50A's
full-scale output voltage, and anything above 3.3 V destroys the pin — so a
routed trace is a live hazard until someone measures it, which is work this
board does not need. And the usual argument for reserving a footprint ("a trace
now versus a respin later") does not apply here: Stage 3 is already a planned
new board, so the respin exists regardless.

This differs from the reserved ELRS footprint on purpose. That one is four pads
on a UART with a known, safe signal level. This one is an unquantified analog
voltage straight onto an ADC pin.

**If it is ever wanted**, on Stage 3: with props off, run the motors up and
meter the ESC's current-sense pin first. If full scale is at or under 3.3 V a
series resistor and a 100 nF filter suffice; above that it needs a divider.
Then calibrate volts-per-amp against a clamp meter, the same way as the battery
divider.

### No FC power switch

An earlier revision planned a 1×2 header pulling `U2 SHDN` low to shut the
regulator down. **It was not fitted** — `SHDN` is no-connected, so the Pololu's
internal pull-up leaves it permanently enabled.

That is a fine outcome. A switch there was only ever a bring-up convenience for
power-cycling the FC without unplugging the pack; it was never a safety device,
since the ESC bus stays live regardless. **The battery connector is the
disconnect**, and USB-with-no-pack remains the "FC running, motors physically
dead" bench mode. See the power-states table in
[hardware_roadmap.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware_roadmap.md).

Worth noting what was deliberately *not* done: putting a switch in the `BAT`
path. That carries ~120 A on a stack like this if it were in the motor feed, and
even on the 0.4 A FC feed it puts raw pack voltage on a mechanical contact a
crash can knock.

---

## Sensors

### U3 — IMU, SPI

| Pin | Signal | ESP32 |
|---|---|---|
| 1 | 3V3 | — |
| 2 | GND | — |
| 3 | SCLK | GPIO 23 |
| 4 | MISO | GPIO 25 |
| 5 | MOSI | GPIO 19 |
| 6 | CS | GPIO 15 |
| 7 | INT | GPIO 35 |

Place `U3` **at the mounting-hole centroid** — the IMU belongs at the CG. The
breakout solders directly to this header; soft-mount the whole board rather
than the sensor. The SparkFun board's I2C/SPI jumper must be cut, per
[hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md).

### J4 / J5 — ToF pair, I2C

**1 × 6 through-hole header, 2.54 mm** — `PinHeader_1x06_P2.54mm_Vertical`.

| Pin | Signal | ESP32 | J4 (ToF A) | J5 (ToF B) |
|---|---|---|---|---|
| 1 | VCC (3V3) | — | | |
| 2 | GND | — | | |
| 3 | SCL | GPIO 21 | shared | shared |
| 4 | SDA | GPIO 22 | shared | shared |
| 5 | GPIO1 | — | no-connect | no-connect |
| 6 | XSHUT | | GPIO 18 | GPIO 26 |

**Six positions, not five.** The VL53L0X symbol has six pins, and every symbol
pin needs a matching pad number — a 5-pad footprint fails with unmapped pins
even though `GPIO1` is unused. Position 5 simply sits unconnected.

Keep the `VL53L0X` symbol rather than a generic `Conn_01x06`: the pin names
document what each wire is. Only the footprint field decides that this is a
connector.

Separate XSHUT lines per sensor are what make the boot address-assignment
sequence possible — both sensors come up at `0x29` and the address does not
persist. See [hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md).

XSHUT is not 5 V tolerant and has an internal pull-up. Drive it directly from
the GPIO.

#### The sensors mount at the frame, not on this board

This is the whole reason these are connectors. The estimator turns the
*difference* between the two ranges into a ground slope, so the sensors go
front-left and rear-right of the CG, as widely spaced as the frame allows, each
with a clear 25° cone below it. Two sensors a few centimetres apart in the
middle of the stack measure nothing useful.

**Strain-relieve every cable.** Put two Ø2 mm holes beside each connector for a
zip-tie. Without it the wire flexes at the solder joint on every landing until
the joint cracks, and that failure is *intermittent* — a ToF that drops out
mid-hop produces garbage the estimator cannot distinguish from a real reading.

On connector choice, for a vehicle whose purpose is repeated impact: plain
2.54 mm header plus Dupont sockets is the easiest to wire and the least secure,
since Dupont housings walk loose under vibration. Acceptable for bring-up **if
the joint is actually glued or tied**. Soldering the wires straight into the
through-holes is more secure and less reworkable; a latching JST-GH housing is
the proper answer if a crimp tool is available. The 1 × 6 footprint supports
all three.

Run each sensor's wires as a single bundle **including its ground return** —
the return has to travel with the signals, not find its own path through the
frame — and keep those bundles away from the motor phase wires. At frame
lengths (10–15 cm) I2C is comfortable; much longer and bus capacitance starts
to matter.

#### VCC must be 3.3 V — this one can destroy the ESP32

The breakouts carry an onboard regulator and a MOSFET level shifter, so `VIN`
accepts 5 V happily. **Do not give it 5 V.** Their I2C pull-up resistors
connect to `VIN` itself, so SDA and SCL sit at whatever `VIN` is — feed it 5 V
and 5 V lands on GPIO 21 and 22, which are not 5 V tolerant.

Put that on the silkscreen beside both connectors:
`VCC = 3.3V ONLY — breakout pulls SDA/SCL to VIN`.

#### I2C pull-ups — footprints fitted, not populated

Checked against the breakout schematic: each carrier has **10 kΩ** pull-ups on
SDA and SCL. Two of them on one bus is ~5 kΩ in parallel, already in the normal
range, so **`R5`/`R6` stay unpopulated.**

Place the footprints anyway — 4.7 kΩ to 3V3, marked DNP. The reason is not
"the bus might not work", which is now settled, but bus capacitance: the
sensors sit on 10–15 cm of cable, and at 400 kHz I2C allows only 300 ns of rise
time. 5 kΩ against that capacitance lands near 250 ns — fine, but without much
margin. If the bus turns out unreliable at 400 kHz, fitting these two in
parallel brings the total to ~2.4 kΩ and halves the rise time.

The fallback, already in
[hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md),
is dropping the bus to 100 kHz, which allows 1000 ns and makes the question
moot.

---

## Remaining circuits

### Buzzer — GPIO 13

```
GPIO 13 ──[ R 100 Ω ]──[ BZ1 piezo ]── GND
```

Two parts, no transistor. A **passive** piezo transducer — sold as "passive
buzzer" or "external drive", *not* the active kind with a built-in oscillator —
is essentially a capacitor and draws almost no current, so a GPIO drives it
directly. The 100 Ω limits the inrush into that capacitance.

Standard parts: TDK **PS1240P02BT** or Murata **PKM13EPYH4000-A0**. What matters
when substituting: passive/external-drive, two leads, resonant near 4 kHz — that
is both where the element is loudest and where human hearing is most sensitive.
A passive piezo is non-polarised, so orientation is irrelevant.

**`BZ1` is a 1×2 header; the piezo mounts off-board on wires.** Buried under the
devkit in the middle of a stack it is muffled — put it somewhere audible on the
frame, and give it the same zip-tie strain relief as the ToF cables.

Firmware drives it with PWM (LEDC) near 4 kHz. A passive piezo makes no sound
from a DC level.

Its job is **audible arming feedback** — knowing whether the motors are live
while picking the vehicle up between hops, without looking at a screen. The
status LED on GPIO 2 is the visual half of the same signal.

### Status LED — GPIO 2

Onboard on the devkit. No external part. Do not load GPIO 2 with anything else
— it is a strapping pin. It is the visual half of the arming indication, with
the buzzer as the audible half.

### Power LED

`+5V ──[ R9 1 kΩ ]── D2 ── GND`. Not required, genuinely useful: it tells you
at a glance whether the regulator is alive, which is the first question during
bring-up.

---

## Protection and defensive design

Cheap things that make a first board survivable. None of these cost meaningful
space or money; all of them cost a respin if left out.

### The devkit can be inserted backwards — prevent it on the silkscreen

Two 1×15 sockets are mechanically symmetric. A devkit rotated 180° puts `+5V`
where `GND` should be and will destroy it, the regulator, or both, instantly.
Nothing electrical stops this.

Silkscreen a **full devkit outline** with the USB end clearly marked —
`USB THIS END` in text, plus a filled arrow. Mark pin 1 on both socket rows.
This is the single most likely way to kill this board.

### `F1` — PPTC on the BAT feed

A 0.5 A hold polyfuse in series with `BAT`, right after `J1`. If the regulator
ever fails short, this is what stops the pack from dumping into the board.
Roughly 0.5 Ω, so ~0.2 V and 80 mW at the 0.4 A this path carries — irrelevant
against the Pololu's 5.1 V minimum input.

### `R4`–`R7` — 100 Ω series on the ESC signal lines

One in series with each of S1–S4, at the **devkit** end, not the connector. They limit fault current
into a GPIO if something goes wrong off-board, and damp edges on the DShot
lines. At DShot600 the RC delay against the ESC's input capacitance is a couple
of nanoseconds — immaterial.

### There is no protection against a reversed ribbon

Worth stating plainly, because the instinct is to add a series diode. It does
not help: an 8-pin connector reversed end-for-end swaps pin 1 with pin 8, so
`GND` lands on `CURR` and `BAT` lands on `S4` — putting 16.8 V directly on
GPIO 14. No power-path diode saves that.

The defence is procedural, so do it properly:

- JST-SH is keyed, so the cable cannot go in upside down — but **the connector
  can be soldered rotated**, and the ESC's own cable can be reversed end for
  end.
- Silkscreen pin 1 on `J3` at both the board and, with a paint pen, on the
  cable.
- **Before the first plug-in, meter the assembled cable**: continuity from the
  ESC's `GND` pad to what you believe is `J3` pin 1. Thirty seconds.

### Test points

Exposed 1 mm pads, no header, for: `BAT`, `+5V`, `+3V3`, `GND` (three of them,
spread across the board), `SDA`, `SCL`, and the `GPIO 34` divider node.

These cost nothing and are the difference between clipping a scope probe on in
two seconds and trying to find bare copper on a soldermasked board while a
motor is spinning. Put a ground test point near each signal one.

### No spare GPIO breakout

A 1×4 header exposing GPIO 15, 36 and 39 was planned and **deleted during
layout** — its three nets were scattered across the board and it was obstructing
routes that mattered more. GPIO 15 subsequently became `IMU_CS`.

Still free and reachable at the devkit socket pads if something is needed later:
**GPIO 4 and 5** (full-function), **36 and 39** (input-only, ADC1).

### Mechanical

- **Match the mounting holes to the soft-mount grommets the KO50A ships with.**
  Plain M3 clearance is Ø3.2 mm, but FC grommets commonly need Ø4 mm. Measure
  the supplied parts before committing — the wrong hole means no soft-mounting,
  on a vehicle where gyro isolation genuinely matters.
- **Leave the mounting holes non-plated with a keepout ring**, so a metal
  standoff cannot short into a ground pour.
- **Two Ø2 mm holes near `J4`/`J5`** to zip-tie the ToF cables. Cable flex at
  the connector is a real failure mode on a vehicle that lands hard.
- **Round the board corners**, 1–2 mm radius. Corners are where FR4 chips on
  impact, and square ones chafe wiring.

### Silkscreen

Beyond the above: label every connector pin with both its signal name **and its
GPIO number**, mark polarity on `C1` and `D2`, and put `REVALI STAGE 2 · rev A ·
<date>` on the bottom copper. In November the board will have to explain itself.

---

## Pin budget

Every ESP32 pin this board uses, checked against the constraints in
[hardware.md](https://github.com/eli-lame/Revali/blob/main/docs/hardware.md):

| GPIO | Use | | GPIO | Use |
|---|---|---|---|---|
| 4 | *free* (full-function) | | 21 | I2C SCL |
| 5 | *free* (full-function) | | 22 | I2C SDA |
| 13 | Buzzer | | 23 | IMU SCLK |
| 14 | ESC 4 | | 25 | IMU MISO |
| 15 | IMU CS | | 26 | ToF B XSHUT |
| 16 | ELRS **TX** (reserved) | | 27 | ESC 3 |
| 17 | ELRS **RX** (reserved) | | 32 | ESC 1 |
| 18 | ToF A XSHUT | | 33 | ESC 2 |
| 19 | IMU MOSI | | 34 | Battery sense (ADC1, in-only) |
| | | | 35 | IMU INT (in-only) |

Clear: 6–11 (flash), 1/3 (UART0 — the console and the bootloader), and 12 are
all unused. GPIO 2 is the onboard LED only. Free and available: **4 and 5**
(full-function), **36 and 39** (input-only, ADC1).

**Every SPI pin and both I2C pins are off their framework defaults.** These
assignments came out of routing this board — the first move was taking SCLK off
GPIO 18 so the two XSHUT lines could sit on opposite pin rows, matching `J4` and
`J5` being on opposite edges; the rest followed.

Firmware must name them all explicitly:

```cpp
SPI.begin(23, 25, 19, 15);   // sck, miso, mosi, ss
Wire.begin(22, 21);          // sda, scl
```

A bare `SPI.begin()` would drive GPIO 18 as the clock — which now resets a ToF
— and a bare `Wire.begin()` has SDA and SCL crossed relative to this board.
Neither errors; the sensors simply never answer, which reads as dead hardware.
Watch the argument orders too: **sck, miso, mosi, ss** and **sda, scl**.

Cost is nil. Non-default pins route through the GPIO matrix instead of the
IOMUX, capping SPI at 40 MHz rather than 80 and adding ~25 ns of MISO input
delay; this bus runs at 7 MHz. It is all-or-nothing, so once one signal is off
its default, moving the rest is free. I2C has no preferred pins at all.

**CS on GPIO 15 is deliberate.** That pin is a strapping pin needing to be high
at boot, and it has an internal pull-up. SPI chip-select is active-low, so the
pull-up holds the IMU deselected through reset and before firmware configures
the pin — the safe state — while satisfying the strapping requirement as a side
effect. **Never add anything that pulls GPIO 15 low at boot.**

Layout consequence: the clock is separated from MISO, MOSI and CS. Skew at
7 MHz is a fraction of a nanosecond against a 71 ns half-period, so it is
harmless — but do not run SCLK closely parallel to an ESC signal, since both are
high-activity lines. Cross at right angles.

**IMU INT is on GPIO 35**, one of the input-only pins, which frees GPIO 4 as a
full-function pin. The interrupt line is only ever an input, so spending an
input-only pin on it costs nothing.

The catch is that GPIO 34–39 have **no internal pull-up or pull-down**, and it
is not configurable. That is fine while the ICM-20948's INT output stays
push-pull (its default) — but there is no pad on this board for the external
pull-up an open-drain configuration would need, so do not reconfigure it. The
pin also floats until the IMU initialises, so firmware must attach the
interrupt handler *after* init rather than before.

The battery divider is on **ADC1**, which is required because ADC2 is unusable
whenever the WiFi radio is active — and with ESP-NOW it always is. Any analog
input added later must also land on ADC1: 35, 36 and 39 are the input-only
ADC1 pins still available.

### J2 — ELRS, reserved

| Pin | Signal | ESP32 |
|---|---|---|
| 1 | +5V | — |
| 2 | GND | — |
| 3 | TX (to RX on the receiver) | GPIO 16 |
| 4 | RX (from TX on the receiver) | GPIO 17 |

Routed and unpopulated. Stage 2 flies on ESP-NOW; this exists so that adopting
ELRS later is a firmware change rather than a board respin. Note the crossover
— the receiver's TX goes to the ESP32's RX.

---

## Bring-up order

Do not skip steps. Each one makes the next failure cheap.

1. **Bare board, no parts.** Continuity check `BAT`–`GND` and `+5V`–`GND` for
   shorts. Verify `J1` pin 1 really is the pin you think it is.
2. **Power section only** — `U2`, `F1`, `D2`, `C9`, `C10`. Feed `BAT`
   from a **current-limited bench supply at 12 V, limit 100 mA**, not a pack.
   Confirm 5 V out and no heating. Walk the supply up to 16.8 V.
3. **Measure `+5V` at the devkit socket's VIN pin** before the devkit goes in.
   This is the step that saves the devkit.
4. **Devkit in.** Confirm 3V3, blink an LED, serial at 115200.
5. **Sensors.** I2C scan, then the ToF address sequence, then the IMU. `[1.4]`,
   `[2.8]`.
6. **Battery divider.** Calibrate against a meter.
7. **ESC ribbon, props off.** Identify which physical motor each of S1–S4
   drives and record it. `[H.3]`.

### Before the devkit is ever powered from the board

Check whether your devkit's `VIN` pin is tied straight to the USB 5 V rail —
many are, with no isolation diode. Continuity between the USB connector's 5 V
pin and the `VIN` header pin, board unpowered. If they are tied, decide
deliberately whether to flash with the pack connected; if there is a diode,
there is nothing to think about. Either way: **props off.**

---

## Bill of materials

As built, read off the board. 39 footprints.

| Ref | Part | Footprint |
|---|---|---|
| `U1` | ESP32-DevKit 30-pin, socketed | `ESP32_Footprints:ESP32_30pin` |
| `U2` | Pololu **D24V10F5** | `HopCopter:Pololu_D24V10F5` |
| `U3` | SparkFun ICM-20948 breakout | `HopCopter:SEN-15335` |
| `D1` | LED | `LED_SMD:LED_0805_2012Metric` |
| `D2` | **SMBJ20CA** TVS, bidirectional | `Diode_SMD:D_SMB` |
| `F1` | PPTC, **0.5 A hold**, 1206 | `Fuse:Fuse_1206_3216Metric` |
| `C10` | **100 µF 35 V** low-ESR electrolytic | `Capacitor_THT:CP_Radial_D8.0mm_P3.50mm` |
| `C1` | 22 µF **16 V X5R** (5 V bulk) | `Capacitor_SMD:C_0805_2012Metric` |
| `C4` | 10 µF **16 V X5R** (3.3 V bulk) | `Capacitor_SMD:C_0805_2012Metric` |
| `C2`, `C3`, `C5`–`C9` | 100 nF **50 V X7R** | `Capacitor_SMD:C_0805_2012Metric` |
| `R1` | 22 kΩ **1 %** (divider, lower leg) | `Resistor_SMD:R_0805_2012Metric` |
| `R2` | 100 kΩ **1 %** (divider, upper leg) | `Resistor_SMD:R_0805_2012Metric` |
| `R3` | 100 Ω (buzzer series) | `Resistor_SMD:R_0805_2012Metric` |
| `R4`–`R7` | 100 Ω (ESC signal series) | `Resistor_SMD:R_0805_2012Metric` |
| `R8` | 1 kΩ (power LED) | `Resistor_SMD:R_0805_2012Metric` |
| `J3` | JST-SH 1.0 mm 8-pin receptacle, SMD | `Connector_JST:JST_SH_SM08B-SRSS-TB_1x08-1MP_P1.00mm_Horizontal` |
| `J2` | 1×4 header 2.54 mm | `Connector_PinHeader_2.54mm:PinHeader_1x04_P2.54mm_Vertical` |
| `J4`, `J5` | 1×6 header 2.54 mm | `Connector_PinHeader_2.54mm:PinHeader_1x06_P2.54mm_Vertical` |
| `BZ1` | 1×2 header 2.54 mm + passive piezo on wires | `Connector_PinHeader_2.54mm:PinHeader_1x02_P2.54mm_Vertical` |
| `TP1`–`TP11` | test points (no `TP6`), 10 total | `TestPoint:TestPoint_THTPad_D2.0mm_Drill1.0mm` |
| — | 2 × 1×15 female header for the `U1` socket | |

**Not fitted:** the I2C pull-up footprints. Settled against — the VL53L0X
carriers each have their own 10 kΩ, giving ~5 kΩ in parallel on the bus. If
400 kHz proves marginal because of the cable runs, the fallback is dropping the
bus to 100 kHz rather than bodging resistors on.

### Value fields to clean up before generating a BOM

The board's value fields are for the designer, not the purchasing list. Before
exporting:

- `U1` is empty (`~`) — set it to `ESP32-DevKit-30pin`
- `D2` reads `D_TVS` — set it to `SMBJ20CA`
- `F1` reads `Fuse` — set it to `PPTC 0.5A 1206`
- `U2` reads `Pololu_D24V10Fx` — set it to `D24V10F5`, the specific variant
- `C2`, `C6`, `C7`, `C8` read `100n` while `C3`, `C5`, `C9` read `100nF`.
  Identical parts, two strings — they will appear as two BOM lines. Normalise.

On the ceramics, **rate them at 2× the rail or better**: ceramic capacitance
falls sharply under DC bias, and a 10 µF 6.3 V part on a 5 V rail can measure
nearer 4 µF in circuit.

0805 throughout rather than 0402 — this board is hand-soldered, and the space
saving is worth nothing here.

---

## Layout notes

- **Ground pour on both layers**, stitched with vias. It is a 2-layer board, so
  the bottom pour is the return path for everything; keep slots in it short.
- **Keep the `U1` input loop tight** — `C1` and `D1` right at the `J1` entry,
  short and fat traces to `U1 VIN`.
- **Route SPI away from the regulator.** `U1` is a switcher; the IMU's SPI bus
  and the two analog inputs are what you least want coupling into.
- **`BAT` traces:** 0.5 mm is generous for 0.4 A, but keep the clearance
  appropriate for 17 V, and keep the net physically short.
- **Analog input:** run `GPIO 34` as a short trace with its filter cap close to
  the devkit socket, over unbroken ground.
- Silkscreen every connector with its pin 1 marker and signal names, and put
  the board revision and date on the bottom. You will thank yourself in
  November.

## Open items before layout

- Frame clearance for the 58 × 88 mm outline — it overhangs the ESC on every side.
- Physical positions for `J4`/`J5` cable runs to the ToF mounts — the sensors
  go front-left and rear-right of the CG, as widely spaced as the frame allows.
- Mounting hole diameter, sized to the KO50A's supplied soft-mount grommets
  rather than to plain M3 clearance.
