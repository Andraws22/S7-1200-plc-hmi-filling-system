# HMI Design Documentation
## Integration with Siemens S7-1200 PLC — Filling Level Control System

---

## 1. Introduction

The HMI (Human-Machine Interface) is configured in **TIA Portal WinCC** and communicates directly with the Siemens S7-1200 PLC over PROFINET. It provides:

- Secure operator access with role-based authentication
- Real-time process visualization
- Manual control override
- Active alarm management
- Historical trend analysis

---

## 2. Navigation Structure

```
┌──────────────────────────────────────────┐
│            HOME / WELCOME SCREEN          │
│    (Project identity + Login button)      │
└──────────────────┬───────────────────────┘
                   │ [Login Button]
                   ▼
┌──────────────────────────────────────────┐
│               LOGIN SCREEN               │
│   Username / Password / Access Level      │
└──────┬──────────────┬────────────────────┘
       │              │
  [Operator]    [Engineer / Admin]
       │              │
       ▼              ▼
  ┌─────────┐   ┌──────────────┐
  │Monitoring│   │Control Screen│
  │Alarms    │   │Monitoring    │
  └─────────┘   │Alarms        │
                │Trends        │
                │Settings(Admin)│
                └──────────────┘
```

---

## 3. Screen Descriptions

### 3.1 Home Screen

The welcome/splash screen that appears on system startup.

**Elements:**
- Project title and logo
- Brief system description
- **[Login]** button — the only navigation element available before authentication
- No direct access to any functional screen is permitted from here

![HMI Welcome Splash Screen Layout](../assets/hmi_home_screen.png)

---

### 3.2 Login Screen

Handles user authentication before granting access to functional screens.

**Elements:**
- Username input field (`User Name` tag — WString, internal connection)
- Password input field (masked)
- Access level selector
- **[Login]** / **[Cancel]** buttons

**Access Levels:**

| Level | Username Role | Granted Screens |
|---|---|---|
| Level 1 | Operator / Human | Monitoring, Alarms |
| Level 2 | Engineer | Monitoring, Alarms, Trends, Control |
| Level 3 | Administrator | All screens including Settings |

---

### 3.3 Operation Control Screen

Allows authorized users (Engineer+) to issue commands to the PLC.

**Elements:**
- **[START]** button → writes `M0.0 = 1` → starts conveyor
- **[STOP / E-STOP]** button → writes `M0.1 = 1` → stops all operation
- **[Position Sensor SIM]** toggle → writes `M0.3` (for testing/simulation)
- **[Level Sensor SIM]** toggle → writes `M0.4` (for testing/simulation)
- Status indicators:
  - Motor: ON / OFF (reads `Q0.0`)
  - Valve: OPEN / CLOSED (reads `Q0.1`)

![HMI Main Simulation Control Panel Dashboard](../assets/hmi_control_screen.png)

---

### 3.4 Monitoring Screen

Real-time process visualization available to all logged-in users.

![HMI Integrated Real-time Process Monitoring Screen Layout](../assets/hmi_control_screen.png)

**Elements:**
- Animated conveyor belt (bottle moving → stopping → filling → moving)
- Filling tank graphic with animated level indicator
- Sensor status indicators:
  - Position Sensor: Active / Inactive (reads `Q0.5`)
  - Level Sensor: Active / Inactive (reads `Q0.4`)
- Motor status badge: RUNNING / STOPPED (reads `Q0.0`)
- Valve status badge: OPEN / CLOSED (reads `Q0.1`)
- Indicator lights for Start (`Q0.2`) and Stop (`Q0.3`) inputs
---

### 3.5 Alarms Screen

Displays active, acknowledged, and historical alarms.

**Alarm Configuration:**

| Alarm Name | Tag | Type | Address | Trigger Condition |
|---|---|---|---|---|
| Motor Speed Alarm | `motor_alarm` | Analog | MW23 | Speed out of range |
| Level Sensor Alarm | `Level_alarm` | Discrete | MW26 | Unexpected level signal |
| Position Sensor Alarm | `position_alarm` | Discrete | MW28 | Unexpected position signal |
| Valve Alarm | `Valve_alarm` | Discrete | MW30 | Valve failed to open/close |

**Features:**
- Alarm list with timestamp, tag name, and description
- Acknowledge button per alarm
- Alarm history log
- Color coding: Red = Active, Yellow = Unacknowledged, Green = Cleared

---

### 3.6 Trend Screen

Graphical historical view of key process variables. Available to Engineers and Administrators.

**Displayed Trends:**
- Bottle fill cycle timing (cycle duration)
- Position sensor activation history
- Level sensor activation history
- Motor ON/OFF state over time
- Valve OPEN/CLOSE state over time

**Features:**
- Time range selector (last 1 min / 10 min / 1 hour)
- Zoom and pan controls
- Export to CSV (Administrator only)

---

## 4. Tags & Data Connections

### Control Tags

| Tag Name | HMI Type | Data Type | PLC Address | Description |
|---|---|---|---|---|
| `User_Name` | I/O Field | WString | Internal | Login username display |
| `Start` | Button | Bool | M0.0 | Sends start command to PLC |
| `Stop_Emergency` | Button | Bool | M0.1 | Sends stop/e-stop command |
| `Position_Sensor` | Button/Indicator | Bool | M0.3 | Position sensor simulation / status |
| `Level_Sensor` | Button/Indicator | Bool | M0.4 | Level sensor simulation / status |

### Status / Feedback Tags

| Tag Name | HMI Type | Data Type | PLC Address | Description |
|---|---|---|---|---|
| `Motor` | I/O Field | Bool | Q0.0 | Motor running indicator |
| `Valve` | I/O Field | Bool | Q0.1 | Valve open indicator |
| `Ind_Start` | Indicator | Bool | Q0.2 | Start input indicator light |
| `Ind_Stop` | Indicator | Bool | Q0.3 | Stop input indicator light |
| `Ind_Level` | Indicator | Bool | Q0.4 | Level sensor indicator light |
| `Ind_Position` | Indicator | Bool | Q0.5 | Position sensor indicator light |

### Alarm Tags

| Tag Name | Data Type | Address | Alarm Type |
|---|---|---|---|
| `motor_alarm` | INT | MW23 | Analog (speed threshold) |
| `Level_alarm` | INT | MW26 | Discrete |
| `position_alarm` | INT | MW28 | Discrete |
| `Valve_alarm` | INT | MW30 | Discrete |

---

## 5. PLC–HMI Communication

- **Protocol**: PROFINET (Ethernet)
- **Connection type**: S7 Communication (integrated TIA Portal connection)
- **Update cycle**: 100ms (configurable)

**Data Flow:**

```
PLC → HMI (status feedback):
  Q0.0 (Motor)     → Motor status indicator
  Q0.1 (Valve)     → Valve status indicator
  Q0.2–Q0.5        → Input indicator lights
  MW23, MW26, MW28, MW30 → Alarm values

HMI → PLC (operator commands):
  M0.0 (Start)     ← Start button
  M0.1 (Stop)      ← Stop / E-Stop button
  M0.3 (Pos.Sim)   ← Position sensor simulation
  M0.4 (Lvl.Sim)   ← Level sensor simulation
```

---

## 6. Security Configuration

Configure in TIA Portal → HMI → Runtime Settings → User Administration:

```
Administrator
├── Password: [set during commissioning]
├── Access: All screens, all tags, export
└── Can create/delete users

Engineer
├── Password: [set during commissioning]
├── Access: Control, Monitoring, Alarms, Trends
└── Cannot modify user accounts or settings

Operator
├── Password: [set during commissioning]
├── Access: Monitoring, Alarms only
└── Read-only on all process data
```

---

## 7. Setup Steps in TIA Portal

1. Add HMI device (e.g., KTP700 Basic PN) to your TIA project
2. Configure PROFINET connection between PLC CPU and HMI panel
3. Create tags as listed in the Tags table above
4. Build screens in WinCC:
   - Use **Screen Navigation** to link buttons between screens
   - Set screen protection levels per the access matrix
5. Configure **Discrete Alarms** for MW26, MW28, MW30
6. Configure **Analog Alarm** for MW23 with HIGH/LOW limits
7. Add **Trend** objects to the Trend screen and bind to relevant tags
8. Test with **HMI Runtime Simulator** before deploying to hardware
