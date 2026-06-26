# Ladder Logic Networks — Detailed Documentation

## Overview

The PLC program consists of **4 networks** written in Ladder Diagram (LAD) language, programmed in TIA Portal for the Siemens S7-1200.

The logic enforces two critical interlocks throughout:
- **Motor and Valve never run simultaneously** — prevents filling while the conveyor is moving
- **Self-latching** on both Motor and Valve — ensures stable operation without requiring held buttons

---

## Network 1 — Motor (Conveyor) Control

### Purpose
Controls the conveyor belt motor. Starts on `START` press, stops when a bottle reaches the position sensor or when `STOP` is pressed.

### Rung Structure

```
 I0.0          I1.4           I0.1          Q0.1          Q0.0
"start"    "pos_sensor"      "stop"        "valve"        "motor"
  ┤├──────────┤/├──────────────┤/├────────────┤/├──────────( )
  │
 Q0.0
"motor"
  ┤├
```

### Contact Logic

| Element | Type | Address | Reason |
|---|---|---|---|
| `start` | NO (Normally Open) | I0.0 | Initiates motor start |
| `motor` (latch) | NO (Normally Open) | Q0.0 | Self-latch to sustain motor without holding START |
| `position_sensor` | NC (Normally Closed) | I1.4 | Opens (breaks rung) when bottle detected → stops motor |
| `stop` | NC (Normally Closed) | I0.1 | Opens when STOP pressed → kills motor |
| `valve` | NC (Normally Closed) | Q0.1 | Interlock: motor cannot run while valve is open |
| `motor` | Output Coil | Q0.0 | Energizes conveyor motor |

### Behavior Notes
- The **self-latch** (`Q0.0` parallel to `I0.0`) allows a momentary press of START to sustain motor operation.
- `position_sensor` is wired as **NC** in the logic because it is a **NO physical sensor** — when it detects the bottle, it closes physically (I1.4 = 1), which opens the NC contact in the rung and stops the motor.
- The valve interlock (`Q0.1 NC`) ensures the motor is always OFF when the valve is ON.

---

## Network 2 — Valve Control

### Purpose
Controls the filling valve. Opens when the bottle is at the filling station (position sensor triggered), closes when the bottle is full (level sensor triggered).

### Rung Structure

```
 I1.4          I1.3           Q0.0          Q0.1
"pos_sensor" "lvl_sensor"   "motor"        "valve"
  ┤├──────────┤/├──────────────┤/├──────────( )
  │
 Q0.1
"valve"
  ┤├
```

### Contact Logic

| Element | Type | Address | Reason |
|---|---|---|---|
| `position_sensor` | NO (Normally Open) | I1.4 | Triggers valve open when bottle detected |
| `valve` (latch) | NO (Normally Open) | Q0.1 | Self-latch to keep valve open after bottle detected |
| `level_sensor` | NC (Normally Closed) | I1.3 | Opens (breaks rung) when bottle is full → closes valve |
| `motor` | NC (Normally Closed) | Q0.0 | Interlock: valve cannot open while motor is running |
| `valve` | Output Coil | Q0.1 | Energizes solenoid valve |

### Behavior Notes
- `position_sensor` is **NO** in this rung because when the bottle arrives, I1.4 = 1 → starts valve.
- `level_sensor` is **NC** because it is a **NO physical sensor** — when liquid reaches the set level, I1.3 = 1, which opens the NC contact and de-energizes the valve.
- The **valve latch** (`Q0.1` in parallel) keeps the valve open independently of the position sensor once energized (the bottle may shift slightly away from the sensor during filling).

---

## Network 3 — Motor Resume After Fill

### Purpose
Re-enables the motor once the bottle has been filled and the level sensor is satisfied.

### Rung Structure

```
 I1.3          Q0.0
"lvl_sensor"  "motor"
  ┤├────────────( )
  │
 Q0.0
"motor"
  ┤├
```

### Contact Logic

| Element | Type | Address | Reason |
|---|---|---|---|
| `level_sensor` | NO (Normally Open) | I1.3 | Triggers when bottle is full (level reached) |
| `motor` (latch) | NO (Normally Open) | Q0.0 | Self-latch to sustain motor for next bottle |
| `motor` | Output Coil | Q0.0 | Energizes conveyor motor to advance to next bottle |

### Behavior Notes
- This network acts as an automatic restart. When `level_sensor` = 1 (bottle full), the valve is killed by Network 2 and the motor is re-enabled here.
- The **self-latch** sustains motor operation until the next bottle triggers the position sensor again.

---

## Network 4 — Indicator Light Outputs

### Purpose
Maps physical inputs and sensor states to output indicator lights (for both the physical PLC panel lights and HMI indicators). These provide visual confirmation of what is active without interfering with control logic.

### Rung Structure

```
 I0.1           Q0.3
"stop"         "1stop"
  ┤├─────────────( )

 I0.0           Q0.2
"start"        "1start"
  ┤├─────────────( )

 I1.3           Q0.4
"lvl_sensor"  "1level_sensor"
  ┤├─────────────( )

 I1.4           Q0.5
"pos_sensor"  "1position_sensor"
  ┤├─────────────( )
```

### Mapping Table

| Input | Address | Output Indicator | Address |
|---|---|---|---|
| `stop` | I0.1 | `1stop` | Q0.3 |
| `start` | I0.0 | `1start` | Q0.2 |
| `level_sensor` | I1.3 | `1level_sensor` | Q0.4 |
| `position_sensor` | I1.4 | `1position_sensor` | Q0.5 |

### Behavior Notes
- These are **pass-through rungs** — each input directly energizes its corresponding indicator output.
- They serve as diagnostic feedback: an operator can see at a glance which inputs are active.
- These outputs are also read by the HMI for the monitoring screen status indicators.

---

## Complete I/O Truth Table

| Condition | I0.0 | I0.1 | I1.3 | I1.4 | Q0.0 (Motor) | Q0.1 (Valve) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Idle (no start) | 0 | 0 | 0 | 0 | OFF | OFF |
| START pressed | 1 | 0 | 0 | 0 | **ON** | OFF |
| Conveyor running | 0 | 0 | 0 | 0 | **ON** (latched) | OFF |
| Bottle detected (pos. sensor) | 0 | 0 | 0 | 1 | OFF | **ON** |
| Filling in progress | 0 | 0 | 0 | 0 | OFF | **ON** (latched) |
| Bottle full (level sensor) | 0 | 0 | 1 | 0 | **ON** | OFF |
| STOP pressed | 0 | 1 | — | — | OFF | OFF |

---

## Design Principles

- **Interlock Safety**: Motor and Valve are mutually exclusive through NC contacts of each other's coil — they can **never** be ON simultaneously.
- **Latching**: Both motor and valve use self-latch patterns so momentary sensor triggers don't cause erratic behavior.
- **NC vs NO Contact Choice**: Each contact type is deliberately chosen based on the fail-safe behavior required. Fail-open on stop (NC stop) ensures the system halts if the stop circuit is broken.
