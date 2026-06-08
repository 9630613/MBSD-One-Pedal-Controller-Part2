# Functional Safety Concept — One-Pedal Drive Controller

> Extension of the [MBSD-OnePedalController-SystemDefinition-HARA-MIL](https://github.com/9630613/MBSD-OnePedalController-SystemDefinition-HARA-MIL) project with a **Functional Safety Architecture** designed in compliance with **ISO 26262** automotive safety standards. Implements hazard-driven safety requirements, ASIL classification, redundant sensing, and a dedicated Safety Function with full unit and integration test coverage in Simulink.


## Project Overview

This project addresses the **functional safety layer** of the one-pedal drive controller for an electric vehicle. Starting from the controller developed in the previous project, a **Hazard Analysis and Risk Assessment (HARA)** was performed to identify risk sources, define safety goals, assign ASIL levels, and implement a real-time **Safety Function** as a Simulink software unit.

The work covers:
- Identification of hazard sources and their ASIL classification
- Definition of safety goals and their attributes (safe state, fault tolerance time, warning concept)
- Derivation of functional and technical safety requirements
- Preliminary safety architecture design (with and without ASIL decomposition)
- Software implementation of the safety mechanism
- Unit testing and integration testing with coverage analysis

> **Scope note:** The one-pedal controller (`Controller.slx`) and the vehicle plant model were developed in prior work. **All safety architecture, safety function design, test harnesses, and test scenarios are original contributions of this project.**



## Identified Hazard Sources

Two main sources of faults were identified through HARA:

### 1. 🔌 CAN Bus Disconnection — ASIL C
A loss of communication on the CAN bus affects multiple critical signals simultaneously:

| Lost Signal | Consequence |
|---|---|
| `AutoTransmissionSelectorState` | Controller cannot determine intended driving mode |
| `VehicleSpeed_km_h` | Controller loses feedback for safe torque decisions |
| `BrakePedalPressed` | Controller unaware of driver emergency intent |
| `TorqueRequest_Nm` (outbound) | Vehicle may receive stale or zero torque command |

**Risk:** Unintended vehicle motion in the wrong direction, unintended acceleration or insufficient deceleration.

**Mitigation:** The CAN system itself provides a disconnection flag signal (`CAN_Disconnection`). This digital signal is monitored by the Safety Function; when asserted, it immediately triggers the safe state.



### 2. Throttle Pedal Position Sensor Fault — ASIL B/C
A faulty pedal position sensor delivers incorrect input to the controller, leading to:
- Unintended acceleration (incorrect positive torque)
- Unintended/insufficient deceleration (incorrect regenerative braking torque)

**Risk:** ASIL B (insufficient deceleration) and ASIL C (unintended acceleration).

**Mitigation:** A **redundant pedal position sensor** is added. The Safety Function computes the error `e` between the two readings:

```
e = ThrottlePedalPosition_main − ThrottlePedalPosition_redundant
```

A tolerance of **±2%** is applied to account for sensor precision. If `|e| ≥ 0.02`, the fault is detected and the safe state is activated.


## Safety Goals

| ID | Safety Goal | ASIL | Safe State | Fault Tolerance Time | Warning Concept | Degradation Concept |
|----|-------------|------|------------|---------------------|-----------------|---------------------|
| **SG1** | The car must stay in Neutral | C, B | `AutoTransmissionState = Neutral` | 2 s | Selector automatically moves to Neutral | — |
| **SG2** | The car must not receive torque | C, B | `TorqueRequest_Nm = 0` | 2 s | — | — |
| **SG3** | Driver must be warned | C, B | `SafeStateAlarm = ON` | 2 s | HMI alarm for driver | Warning lamp signal to alert other drivers on the road |

All three safety goals are activated **simultaneously** upon fault detection. Together they ensure the vehicle safely coasts to a stop with the driver fully informed.


## Safety Architecture

The following describes the key components and data flows:

<img width="1478" height="610" alt="image" src="https://github.com/user-attachments/assets/6fcb18ab-eed4-4f23-8927-39f997ed18d2" />


## Safety Function Implementation

The Safety Function is a dedicated Simulink software unit that wraps the output of the one-pedal controller. It takes two additional inputs on top of the standard controller outputs:

### Inputs to Safety Function

| Signal | Source | Type | Description |
|--------|--------|------|-------------|
| `TorqueRequest_Nm` | Controller | `single` | Torque computed by one-pedal controller |
| `AutoTransmissionSelectorState` | Controller | `enum` | Current gear/mode state |
| `e` | Computed | `single` | Error between main and redundant pedal sensors |
| `CAN_Disconnection` | CAN system | `boolean` | Flag: 1 = CAN fault detected |

### Outputs of Safety Function

| Signal | Type | Safe State Value | Normal Value |
|--------|------|-----------------|--------------|
| `FinalTorqueRequest_Nm` | `single` | `0` | Pass-through from controller |
| `FinalAutoTransmissionState` | `enum` | `Neutral` | Pass-through from controller |
| `SafeStateAlarm` | `boolean` | `1` | `0` |

<p align="center">
  <img width="2526" height="994" alt="image" src="https://github.com/user-attachments/assets/412dd308-9913-425d-b1d8-9c4025b2c9bb" />
<br>
  <em> Input & Output of the Safty function block along with One Pedal Controller</em>
</p>


### Decision Logic

```
IF (CAN_Disconnection == 1) OR (|e| >= 0.02) THEN
    SafeState = ON
    FinalTorqueRequest_Nm        = 0
    FinalAutoTransmissionState   = Neutral
    SafeStateAlarm               = 1
ELSE
    SafeState = OFF
    FinalTorqueRequest_Nm        = TorqueRequest_Nm  (from controller)
    FinalAutoTransmissionState   = CurrentSelectorState
    SafeStateAlarm               = 0
END
```
<p align="center">
  <img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/2be343ac-2a52-4940-802d-71ca782377df" />
  <br>
  <em> Safty function block control logic</em>
</p>

The fault tolerance time for all safety goals is **2 seconds** — the maximum delay from fault detection to safe state activation.


### Signal Plausibility Checks

| Signal | Type | Range | Check Applied |
|--------|------|-------|---------------|
| `e` (pedal error) | `single` | [−1, 1] | `abs(e) ≥ 0.02` triggers safe state; a Single-type block converter ensures signal compatibility |
| `CAN_Disconnection` | `boolean` | {0, 1} | Digital; no range check possible — presence of signal is the check |


## Software Testing

### Unit Tests

The Safety Function was tested in isolation using a dedicated Simulink test harness. The two effective inputs (`e` and `CAN_Disconnection`) were exercised across equivalence classes and boundary conditions.

<p align="center">
  <img width="600" height="250" alt="image" src="https://github.com/user-attachments/assets/1bbaf449-4580-4b93-8181-78dd25d36734" />
  <br>
  <em> Safty Function Unit Test Setup</em>
</p>

**Input equivalence classes for `e`:**

| Class | Range | Expected Output |
|-------|-------|-----------------|
| Large negative error | −1 < e < −0.02 | Safe State **ON** |
| Within tolerance | −0.02 ≤ e ≤ +0.02 | Safe State **OFF** |
| Large positive error | +0.02 < e < 1 | Safe State **ON** |

**CAN_Disconnection combinations:**

| `CAN_Disconnection` | `e` in tolerance | Expected Output |
|---------------------|-----------------|-----------------|
| 0 | Yes | Safe State **OFF** |
| 0 | No | Safe State **ON** |
| 1 | Yes | Safe State **ON** |
| 1 | No | Safe State **ON** |

**Test strategy:** Boundary values (±0.02, ±1, 0), interior values, and outside-boundary values were tested for `e`. Each test set was run twice — once with `CAN_Disconnection = 0`, once with `CAN_Disconnection = 1`. In this way all possible combinations satisfied.
<p align="center">
  <img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/eee4b88e-f8f6-4fd2-a2ea-1dda50baeee7" />
  <br>
  <em> Designed signals for e and CANDisconnection in signal editor as Inpust for Safty Function</em>
</p>

**Result:** All tests passed. When safe state is active, `TorqueRequest_Nm = 0` and `AutoTransmissionState = Neutral` in all cases.
<p align="center">
  <img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/e3b1f3d3-afb7-4a36-9196-23e0210d5ba0" />

  <br>
  <em> OutPut Results of Safty Function Unit Test</em>
</p>

<p align="center">
  <img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/389ee2bf-c47b-4b46-abe6-d384a32fdd5c" />
  
  <br>
  <em> Signal Monitoring of Safty Function Unit Test  </em>
</p>


**Coverage Unit Test:** 
<p align="center">
  <img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/f7892dc0-f369-49e3-b211-22ac1c69abc6" />

  <br>
  <em> Coverage Unit Test Results</em>
</p>


### Integration Tests

The Safety Function was integrated with the full one-pedal controller and tested end-to-end using the same stimuli as the unit tests.

**Test setup:**
- The controller's input interface was extended to receive `e` (pedal error) and `CAN_Disconnection` as additional inputs.
- The same test scenarios from the unit test were replayed at the integration level.

<p align="center">
  <img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/1f4fea8d-eaed-4fa6-b7ba-d28a3f6491f7" />

  <br>
  <em> Integration Test Setup</em>
</p>

**Result:** 
Safe state correctly propagates through the integrated system — final torque and transmission state override controller output upon fault detection.

<p align="center">
  <img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/67d2e92c-ba28-4c21-8d82-a3196b954b77" />
  <br>
  <em> Output Result of Integration Test  </em>
</p>

<p align="center">
  <img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/731eb1f7-6339-4661-89fa-ac6f0a75455d" />
  <br>
  <em> Signal Monitoring of Integration Test </em>
</p>


**Coverage note:** Integration-level coverage was lower than the Safety Function unit coverage. Two contributing factors were identified:
1. The driver-stimulus test signals do not cover all possible input combinations of the controller's own logic paths.
2. Some controller branches were not efficiently coded for coverage.
The Safety Function itself achieved full coverage at the integration level.

<p align="center">
  <img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/1adb76a2-2101-4ebc-986d-f2b16fbc4170" />

  <br>
  <em> Coverage Integration Test Results</em>
</p>

## Repository Structure

```
├── SafetyFunction.slx            # ← Safety Function Simulink model (original)
├── Controller.slx                # One-pedal controller (from prior project)
├── Plant.slx                     # Vehicle longitudinal dynamics (provided)
├── Harness.slx                   # Top-level test harness (provided)
├── SafetyFunction_Harness.slx    # ← Unit test harness for Safety Function (original)
├── Integration_Harness.slx       # ← Integration test harness (original)
├── init_fn.m                     # Model initialization parameters
├── Tests/
│   ├── Safty_Function_Coverage_Integration_Test.pdf         
│   └── Safty_Function_Coverage_Unit_Test.pdf           
└── README.md
```


## Tools & Standards

- **MATLAB / Simulink / Stateflow** — Model-based design, FSM, signal routing
- **Simulink Test** — Test harness creation, signal editor, coverage measurement
- **ISO 26262** — Automotive functional safety standard (HARA, ASIL, FSC, TSC)
- **ASIL C / ASIL B** — Automotive Safety Integrity Levels applied to identified hazards
