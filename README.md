# BGR_Designing
This repository contains the design basics of a BGR.

# Bandgap Reference (BGR) & Low Dropout Regulator (LDO)
> A full-custom Bandgap Reference (BGR) circuit designed at the transistor level to provide a stable reference voltage independent of temperature and supply voltage, followed by a Low Dropout Regulator (LDO) for load driving capability. Three progressively refined BGR architectures are explored — from an ideal-source-based design to the Banba BGR with a startup circuit. Designed and simulated using Cadence Virtuoso.

---

## Table of Contents
- [Theory](#theory)
  - [CTAT](#ctat-complementary-to-absolute-temperature)
  - [PTAT](#ptat-proportional-to-absolute-temperature)
  - [Output Voltage Derivation](#output-voltage-derivation)
- [Key Parameters](#key-parameters)
- [BGR Architectures](#bgr-architectures)
  - [Version 1 — Classic BGR (Ideal Current Source)](#version-1--classic-bgr-ideal-current-source)
  - [Version 2 — Alternate BGR](#version-2--alternate-bgr)
  - [Version 3 — Banba BGR (No External Source)](#version-3--banba-bgr-no-external-source)
- [Op-Amp Specs](#op-amp-specs)
- [Circuit Tuning](#circuit-tuning)
- [Startup Circuit](#startup-circuit)
- [Low Dropout Regulator (LDO)](#low-dropout-regulator-ldo)
- [Simulation Results](#simulation-results)

---

## Theory

### CTAT (Complementary To Absolute Temperature)

The forward voltage drop of a diode is approximately 0.7 V and remains relatively constant with supply voltage, making it a useful reference. However, it is temperature-dependent:

$$V_1 = V_T \ln\left(\frac{I}{I_S}\right)$$

Where V_T is proportional to temperature, but I_S grows with the third power of temperature and increases exponentially. This causes the forward voltage drop to **decrease by approximately 2 mV/K** — a CTAT characteristic.

### PTAT (Proportional To Absolute Temperature)

To cancel the CTAT component, a circuit with an equal and opposite temperature coefficient is needed. By taking the difference between two diode voltages with different current densities, I_S cancels out:

$$V_1 - V_n = V_T \ln\left(\frac{I}{I_S}\right) - V_T \ln\left(\frac{I}{nI_S}\right) = V_T \ln(n)$$

V_n is the voltage drop across n parallel diodes. With an amplification factor K, the temperature coefficient of the PTAT output becomes **+2 mV/K**:

$$\frac{V_T}{T} \ln(n) \times K = 2$$

At 300 K, V_T = 25 mV, so **K × ln(n) = 24**, giving a PTAT output of K × 25 mV = **0.6 V**.

### Output Voltage Derivation

The BGR output combines the CTAT and PTAT contributions:

$$V_{BG} = V_1 + \frac{R_1}{R_0} \cdot V_T \ln(n) \approx 0.7 + 0.6 = 1.3\ \text{V}$$

In the implemented circuit, the op-amp forces nodes A and C to the same voltage (V1). The voltage drop across R0 is (V1 − Vn), which is amplified by R1/R0 to produce the PTAT component. The full expression is:

$$V_{BG} = V_1 + \frac{R_1}{R_0}(V_1 - V_n) = V_1 + \frac{R_1}{R_0} V_T \ln(n)$$

---

## Key Parameters

| Parameter | Description | Formula |
|-----------|-------------|---------|
| Output Reference Voltage | Stable DC reference | $V_1 + \frac{R_1}{R_0} V_T \ln(n)$ |
| Temperature Coefficient | Change of V_ref with temperature | $\text{T.C.} = \frac{\Delta V_{ref}}{\Delta T}$ (V/°C) or $\frac{\Delta V_{ref}}{V_{ref} \cdot \Delta T}$ (ppm) |
| Line Regulation / Supply Sensitivity | Change of V_ref with VDD | $L.R. = \frac{\Delta V_{ref}}{\Delta V_{DD}}$ |
| PSRR | Rejection of supply ripple at output | $\text{PSRR} = 20 \log_{10}\left(\frac{\Delta V_{DD}}{\Delta V_{OUT}}\right)$ |
| Startup Time | Time to reach stable reference | — |
| Supply Voltage Range | Minimum VDD to operate | — |

---

## BGR Architectures

### Version 1 — Classic BGR

The bandgap reference (BGR) is implemented using an op-amp, a PMOS current mirror, and three branches that generate complementary temperature-dependent voltages.

**Figure 1: BGR Circuit Realization**

<img src="Images/1. BGR_CR.png" width="1000">


### 🔹 Branch Description

- **Left Branch (Node A)**  
  A diode-connected BJT produces a base-emitter voltage $V_1$, which exhibits a **CTAT (Complementary-To-Absolute-Temperature)** characteristic:
  $V_A$ = $V_1$

- **Middle Branch (Node C)**  
  A stack of BJTs with emitter area ratio $n$ generates a voltage $V_n$.  
  The op-amp forces:
  $V_C$ = $V_A$ = $V_1$
  
  This creates a voltage difference across resistor $R_0$:
  $V_C$ - $V_D$ = $V_1$ - $V_n$

  The resulting current is:
  $I = (V_1 - V_n)/R_0$

  Since:
  $V_1 - V_n = V_T \ln(n)$
  this term is **PTAT (Proportional-To-Absolute-Temperature)**.

---

### 🔹 Op-Amp Feedback Mechanism

The op-amp regulates the circuit to maintain equilibrium:

- If $V_C < V_A$: output decreases → PMOS current increases → $V_C$ rises  
- If $V_C > V_A$: output increases → PMOS current decreases → $V_C$ falls  

Thus, the system stabilizes at:

$V_C = V_A = V_1$

---

### 🔹 Output Branch ($V_{BG}$)

The mirrored current flows through resistor $R_1$, producing the bandgap output voltage:
$V_{BG} = V_1 + R_1 \cdot I$

Substituting $I$:

$V_{BG} = V_1 + \frac{R_1}{R_0}(V_1 - V_n)$

Using $V_1 - V_n = V_T \ln(n)$:

$V_{BG} = V_1 + \frac{R_1}{R_0} \cdot V_T \ln(n)$

---

### 🔹 Final Expression

The output consists of:

- **CTAT component:** $V_1$  
- **PTAT component:** $\frac{R_1}{R_0}(V_1 - V_n)$  

$\boxed{
V_{BG} = V_1  + \frac{R_1}{R_0}(V_1 - V_n)
}
$




**Limitations:**
- Relies on an ideal current source (not practical).
- Output voltage cannot be made lower than ~1.2 V.
- No self-startup — requires external kickstart.

---

### Version 2 — Alternate BGR

An improved version that reduces the number of branches. But this version has the same problems too.

**Figure 2: BGR Circuit Realization (Alternate Version)**

<img src="Images/2. BGR_AV.png" width="1000">

---

### Version 3 — Banba BGR (No External Source)

The Banba architecture eliminates the limitation that the output voltage must be higher than 1.2V. In this configuration, we will be using the starting circuit, which will be discussed later.

**Figure 3: Banba BGR Circuit**

<img src="Images/4. Banba_BGR.png" width="1000">

Current branches IA, IB, and IC are mirrored from a single Op-Amp-controlled loop:

$I_C = I_A = I_{A1} + I_{A2} = \frac{V_A}{R_0} + \frac{V_B - V_n}{R_3} = \frac{1}{R_0}\left(V_1 + \frac{R_0}{R_3}(V_1 - V_n)\right) = \frac{V_{conv}}{R_0}$

$V_{BG} = R_2 \cdot I_C = \frac{R_2}{R_0} \cdot V_{conv}$

This architecture enables output voltages below the classic ~1.2 V floor (achieved ~800 mV), making it suitable for low-voltage applications. A **startup circuit** is mandatory for this topology.

---

## Op-Amp Specs

The op-amp used inside the BGR feedback loop is a custom-designed amplifier. Supply: **2 V**, Bias current: **25 µA**.

| Parameter | Maximum | Minimum |
|-----------|---------|---------|
| Gain (dB) | 92.62 | 64.35 |
| Unity Gain Bandwidth | 27.93 MHz | 17.92 MHz |
| Phase Margin (°) | 74.13 | 56.45 |
| ICMR (V) | 532.4 mV (target: 500 mV) | 784.9 mV (target: 800 mV) |
| Offset (V) | 36.78 µV | −24.81 µV |

---

## Circuit Tuning

The output temperature behavior can be adjusted by modifying resistor ratios and diode multiplicity:

| Symptom | Fix |
|---------|-----|
| CTAT dominating — V_ref decreasing with temperature | Decrease R0, or increase R1, or increase n |
| PTAT dominating — V_ref increasing with temperature | Increase R0, or decrease R1, or decrease n |
| Output voltage too high | Reduce current by increasing R0 |

---

## Startup Circuit

### Problem

When VDD rises from 0, all node voltages start at 0 V. The op-amp inputs fall outside its common-mode range, causing its output to go high. This turns off the PMOS current mirror, resulting in zero current flow — a **deadlock (zero-current) state**.

### Why the Op-Amp Output Goes High

**Figure 4: Op-Amp Circuit**

<img src="Images/3. SC_OH.png" width="1000">

Transistors P3 and P4 turn fully on. S1OUT is pulled near VSS. P2's gate sits near VDD/2, causing P2 to pull the output toward VDD.

### Mechanism

The startup circuit detects this zero-current state and injects current into the loop, forcing node A away from 0 V and bringing the op-amp back into its valid operating range. Once the circuit reaches its correct operating point and VDD is sufficiently high, the startup circuit disengages — the PMOS gate voltage approaches VDD, cutting off startup influence.

---

## Low Dropout Regulator (LDO)

### Motivation

The BGR output has high impedance and cannot directly drive loads or supply significant current. The LDO uses the BGR output (V_BG) as its reference to produce a regulated, low-impedance output voltage.

**Figure 5: LDO Circuit**

<img src="Images/5. LDO.png" width="1000">

### Circuit Mechanism

The op-amp forces node A (connected to V_BG) and node B (the feedback node) to the same voltage. The output voltage is set by the resistor divider R1/R2:

$$V_{out} = I \cdot (R_1 + R_2) = \frac{V_B}{R_2}(R_1 + R_2) = V_{REF}\left(1 + \frac{R_1}{R_2}\right)$$

where V_A = V_B = V_REF and VSS = 0 is assumed.

---

## Simulation Results

### Version 1 — Classic BGR

| Parameter | Max Value | Min Value |
|-----------|-----------|-----------|
| Output Reference Voltage | 1.23 V | 1.202 V |
| Temperature Coefficient | 78.24 µV/°C (65.07 ppm) | −43.34 µV/°C (−35.24 ppm) |
| Line Regulation / Supply Sensitivity | 16.46 mV/V | 6.356 mV/V |
| PSRR (min) | −47.48 dB | −52.73 dB |
| PSRR (max) | −9.38 dB | −12.07 dB |
| Supply Voltage Starting Range | 1.64 V | 1.374 V |
| Power Consumption | 354.5 µW | 220 µW |

### Version 2 — Alternate BGR

| Parameter | Max Value | Min Value |
|-----------|-----------|-----------|
| Output Reference Voltage | 1.226 V | 1.202 V |
| Temperature Coefficient | 42.05 µV/°C (34.97 ppm) | −56.79 µV/°C (−46.32 ppm) |
| Line Regulation / Supply Sensitivity | 1.149 mV/V | 0.1951 mV/V |
| PSRR (min) | −45.93 dB | −76.5 dB |
| PSRR (max) | 17.64 dB | 12.86 dB |
| Supply Voltage Starting Range | 1.624 V | 1.302 V |
| Power Consumption | 273.1 µW | 176.2 µW |

### Version 3 — Banba BGR (Ideal Resistor)

| Parameter | Max Value | Min Value |
|-----------|-----------|-----------|
| Output Reference Voltage | 808.8 mV | 793.4 mV |
| Temperature Coefficient | 27.76 µV/°C (34.99 ppm) | −35.48 µV/°C (−43.86 ppm) |
| Line Regulation / Supply Sensitivity | 15.79 mV/V | −2.577 mV/V |
| PSRR (min) | −57.69 dB | −63.34 dB |
| PSRR (max) | −1.202 dB | −17.62 dB |
| Startup Time (10 µs ramp) | 10.46 µs | 7.897 µs |
| Supply Voltage Starting Range | 1.778 V | 1.108 V |
| Power Consumption | 841.4 µW | 190 µW |

### Version 3 — Banba BGR (Real Resistor)

| Parameter | Max Value | Min Value |
|-----------|-----------|-----------|
| Output Reference Voltage | 809.4 mV | 796.6 mV |
| Temperature Coefficient | 9.175 µV/°C (11.52 ppm) | −31.42 µV/°C (−38.82 ppm) |
| Line Regulation / Supply Sensitivity | 7.912 mV/V | −4.02 mV/V |
| PSRR (min) | −51.94 dB | −62.29 dB |
| PSRR (max) | −2.719 dB | −13.88 dB |
| Startup Time (50 µs ramp) | 50.79 µs | 34.29 µs |
| Supply Voltage Starting Range | 1.666 V | 1.265 V |
| Power Consumption | 1.111 mW | 108.8 µW |

### LDO

| Parameter | Max Value | Min Value |
|-----------|-----------|-----------|
| Output Reference Voltage | 1.012 V | 995.8 mV |
| Temperature Coefficient | 11.39 µV/°C (11.44 ppm) | −39.52 µV/°C (−39.05 ppm) |
| Load Regulation | −553.3 µV/mA | −688.7 µV/mA |
