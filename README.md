# Two-Stage Miller-Compensated CMOS Opamp

Analog course project: a 1.2 V two-stage opamp designed in 90 nm (GPDK090) with the gm/ID methodology and simulated in Cadence Virtuoso (ADE L / ADE XL).

## 1. Objective and specifications

| Parameter | Specification |
|---|---|
| Supply voltage (VDD) | 1.2 V |
| DC gain | ≥ 50 dB |
| Unity-gain bandwidth (UGB) | ≥ 20 MHz |
| Phase margin (PM) | ≥ 60° |
| Slew rate | ≥ 10 V/µs |
| Load capacitance (CL) | 1 pF |
| Power | ≤ 1 mW |

## 2. Architecture

A single stage at L = 500 nm cannot reach 50 dB in 90 nm, so two gain stages are used. The overall gain is the product of the stage gains:

```
A0 = gm1·(ro_NM1 ‖ ro_PM1) × gm2·(ro_PM2 ‖ ro_NM4)
```

### Transistor roles

| Device | Role |
|---|---|
| NM3 | Diode-connected reference of the bias mirror; carries IBIAS (20 µA) |
| NM2 | Tail current source; mirrors NM3 and sets the 20 µA bias of stage 1 |
| NM0, NM1 | NMOS differential input pair; converts differential input voltage to current and sets gm1 |
| PM0, PM1 | PMOS current-mirror load; converts differential current to single-ended output (node V1) and doubles the gain |
| PM2 | Second-stage common-source amplifier; adds gain and inversion and sets gm2 |
| NM4 | Current-source load of stage 2; mirrors NM3 with multiplier 4 (about 80 µA) |
| Cc, Rz | Miller capacitor and nulling resistor for compensation |

**Signal path:** IN± → NM0/NM1 (voltage to current) → PM0/PM1 (differential to single-ended, node V1) → PM2 (gain and inversion) → VOUT.

![Opamp schematic](images/schematic.png)
*Two-stage Miller-compensated opamp schematic.*

**Current balance:** each side of the pair carries about 10 µA. PM2 is 8× the size of PM0/PM1 (m = 8), so it draws about 80 µA, which matches NM4 (4 × 20 µA). This keeps systematic offset low.

## 3. gm/ID design methodology

Square-law equations are inaccurate in 90 nm (velocity saturation, channel-length modulation, moderate inversion). The gm/ID method sizes devices from simulated characterization curves instead.

- **gm/ID** measures transconductance efficiency. High values give more gain per µA but need larger, slower devices. Low values mean strong inversion: fast but power-hungry.
- **ID/W** is the current density. After choosing gm/ID, the characterization plot gives ID/W, and the width follows from `W = ID / (ID/W)`.
- **gm·ro** is the intrinsic gain. It grows with channel length, which is why L = 500 nm was used on all devices instead of the 100 nm minimum.

**Workflow:** characterize NMOS and PMOS in a testbench (ID/W vs gm/ID and gm/ID vs gm·ro), choose gm/ID per device, compute W from the target current, then verify with DC, AC and transient simulation.

## 4. Final design

All devices use L = 500 nm.

| Device | Size |
|---|---|
| PM0, PM1 | W = 1.4 µ |
| PM2 | W = 1.4 µ, m = 8 |
| NM0, NM1 | W = 1.8 µ |
| NM2, NM3 | W = 4 µ |
| NM4 | W = 4 µ, m = 4 |
| Rz | 3 kΩ |
| Cc | 400 fF |
| IBIAS | 20 µA |

Supply current is about 100 µA (20 µA tail + 80 µA second stage) plus the 20 µA bias branch, so power is roughly 0.12 to 0.15 mW at 1.2 V, well under the 1 mW limit.

## 5. Frequency response and compensation

### Why compensation is needed
Without compensation, the poles at node V1 and at VOUT sit close together at low frequency. The phase reaches about −180° before the gain falls to 0 dB, so phase margin is near zero.

### Pole splitting with the Miller capacitor
Cc connects V1 to VOUT, across the second stage, and moves the poles in opposite directions:

- **Dominant pole moves down** through the Miller effect: `p1 ≈ 1 / (gm2·R1·R2·Cc)`.
- **Second pole moves up:** at high frequency Cc makes PM2 diode-connected, so `p2 ≈ gm2 / CL`.

A grounded capacitor would need to be several nF to give the same dominant pole and would not move p2. The Miller capacitor achieves both with 0.4 pF.

### Unity-gain bandwidth
```
UGB = gm1 / (2π·Cc)
```
UGB depends only on the input-pair transconductance and Cc. Raising the tail current and reducing Cc were the levers used to meet UGB at the slow corner.

### RHP zero and the nulling resistor
The feedforward path through Cc creates a zero at `gm2 / Cc` in the right half plane, adding phase lag. A series resistor Rz changes the zero to:

```
z = 1 / [Cc · (1/gm2 − Rz)]
```

With Rz = 3 kΩ (greater than 1/gm2) the zero moves into the left half plane and adds phase lead.

### Slew rate
`SR ≈ I_tail / Cc`. Measured values fall below the ideal figure because the second stage and the 1 pF load also limit the output.

## 6. Simulation setup

- **DC operating point:** every device in saturation (vds > vdsat), especially tail NM2 and current source NM4.
- **AC analysis:** open-loop gain and phase give DC gain, UGB and PM. Both inputs are set to 750 mV DC, with 1 V AC on one input.
- **Transient analysis:** unity-gain buffer with a step input and CL = 1 pF; slew rate from rise and fall slopes.
- **Corners (ADE XL):** NN at 27 °C, SS at 125 °C, FF at −40 °C.

### Choice of 750 mV input common mode
The input common mode must keep the tail in saturation: `VICM ≥ Vgs(pair) + Vdsat(NM2)`. At 0.7 V the tail fell into triode at 125 °C and DC gain collapsed to 42.7 dB. At 750 mV, together with the later device widening, the tail stays in saturation at every corner.

## 7. Results

### AC (final design, IBIAS = 20 µA)

| Corner | DC gain | UGB | Phase at UGB | PM |
|---|---|---|---|---|
| NN, 27 °C | 60.87 dB | 43.10 MHz | −112.8° | about 67° |
| SS, 125 °C | 59.89 dB | 26.91 MHz | −116.9° | about 63° |
| FF, −40 °C | 60.17 dB | 63.70 MHz | −106.8° | about 73° |

![AC response at NN corner](images/ac_nn.png)
*Open-loop response at NN (27 °C): DC gain 60.87 dB, UGB 43.10 MHz, phase at UGB −112.82°, so PM ≈ 67.2°.*

### Transient (unity-gain buffer, IBIAS = 20 µA)

| Corner | Rise slew rate | Fall slew rate |
|---|---|---|
| NN, 27 °C | about 31.8 V/µs | about 35.7 V/µs |
| SS, 125 °C | about 22.5 V/µs | about 29.4 V/µs |
| FF, −40 °C | about 38.6 V/µs | about 40.6 V/µs |

![Transient response at NN corner](images/tran_nn.png)
*Unity-gain buffer step response at NN (27 °C), CL = 1 pF, output stepping 0.6 V to 1.0 V (time axis in µs). Slew rate measured between 720 mV and 980 mV: rise about 32 V/µs, fall about 36 V/µs.*

All specifications are met at every corner.

### Operating point checks
- SS 125 °C: NM2 in saturation (vds 310.5 mV, vdsat 210.4 mV); NM4 in saturation (vds 416.9 mV, vdsat 210.8 mV).
- FF −40 °C: NM2 vds 414.2 mV vs vdsat 84.8 mV; NM4 vds 559.2 mV vs vdsat 85.2 mV.

## 8. Design iterations

1. **Placeholder schematic:** all core transistors at W = 120n, L = 100n to set up the schematic and testbench.
2. **gm/ID sizing at L = 500n:** with Cc = 0.5 p and Rz = 3k, gain 64.8 dB, UGB 20.77 MHz, PM about 64°. An earlier odd result (97 MHz, about −2° PM) was caused by a wrong Cc value in the testbench and was corrected.
3. **SS 125 °C failure:** UGB dropped to 13.53 MHz because gm falls with mobility at the hot, slow corner.
4. **Revision 1:** IBIAS raised to 20 µA, Cc reduced to 0.4 p, NM4 m = 4, PM2 m = 8. UGB reached about 37.7 MHz at NN, but at 125 °C the tail NM2 entered triode and gain fell to 42.7 dB.
5. **Revision 2:** input common mode raised from 0.7 V to 0.75 V and the current sources and diff pair widened. All devices stayed in saturation and all corners met specification.

## 9. Conclusion

The opamp meets every specification at all three corners: DC gain about 60 dB, UGB 27 to 64 MHz, phase margin 63° to 73°, slew rate above 22 V/µs and power around 0.15 mW. Key lessons:

- Longer channel length (500 nm) is needed for gain.
- Miller compensation with a nulling resistor provides pole splitting and a left-half-plane zero.
- The slow, hot corner limits UGB and tail-device headroom.
- Input common mode and current-source sizing must keep headroom at the worst corner.

## Tools

Cadence Virtuoso (schematic, ADE L, ADE XL), GPDK090 process design kit.
