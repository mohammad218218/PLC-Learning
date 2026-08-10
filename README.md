# 🏭 PLC Learning: Embedded SoftPLC on Raspberry Pi

Welcome to my **PLC Learning** repository! This project documents my journey in building an industrial-grade SoftPLC system using an Embedded Linux environment. The goal is to bridge the gap between embedded hardware (Raspberry Pi) and industrial automation standards (IEC 61131-3).

---

## 🛠️ System Architecture

### Hardware Stack
The physical layer is built to simulate a real industrial control panel, handling noise and inductive loads properly:
*   **Controller:** Raspberry Pi (acting as the SoftPLC brain)
*   **Inputs:** 2x Push Buttons (Start / Stop) 
    *   *Design:* **Active-Low** configuration with **10kΩ Hardware Pull-Up** resistors to eliminate floating pin noise.
*   **Output:** 5V Relay Module
    *   *Driver Circuit:* Custom-built on a breadboard using an **NPN Transistor (SS8050)**, base resistor, and a **Flyback Diode** to protect the Pi from inductive spikes.

### Software Stack
*   **Runtime:** OpenPLC v3 (Embedded Linux version)
*   **Language:** Structured Text (ST) conforming to IEC 61131-3 standards
*   **Task Configuration:** Running on a cyclic task with a `20ms` interval.

---

## 🚀 Development Phases

### Phase 1: Hardware Integration & Basic Latch Logic (Completed)
The primary goal of this phase was to establish a stable communication between the OpenPLC runtime and the GPIOs of the Raspberry Pi.
*   Successfully mapped I/O pins (`%IX0.0`, `%IX0.1`, `%QX0.0`).
*   Implemented a standard motor starter logic (Latch Circuit) using Structured Text.
*   Successfully tested hardware debounce and relay switching without Pi resets or crashes.

#### 📸 Hardware Setup
![Hardware Wiring and Relay Circuit](Images/hardware_setup.jpeg)

### Phase 2: Advanced Control Logic with Timers (Completed)
Building upon the stable hardware layer, the core logic was upgraded to simulate a real-world industrial fluid control system (e.g., pipe clearing or surge prevention).

#### Industrial Scenarios Implemented:
1.  **3-Second Start Delay (TON):** The Start button must be held continuously for 3 seconds to activate the pump. This filters out accidental touches and electrical noise.
2.  **5-Second Line Clearing / Stop Delay (TOF):** When the Stop command is issued, the pump continues to run for exactly 5 seconds before shutting down to clear remaining fluids from the pipeline.

#### I/O & Variables Mapping:
| Variable Name | Data Type | PLC Address | Description |
| :--- | :--- | :--- | :--- |
| `Start_Btn` | `BOOL` | `%IX0.0` | Active-Low input to start the sequence |
| `Stop_Btn` | `BOOL` | `%IX0.1` | Input to break the latch (Emergency/Stop) |
| `Relay_Pump` | `BOOL` | `%QX0.0` | Output signal to the NPN relay driver |
| `System_Run` | `BOOL` | Internal | Software latching state |
| `Delay_Timer` | `TON` | Internal | Timer On Delay (Start Delay) |
| `Delay_Timer_2` | `TOF` | Internal | Timer Off Delay (Stop Delay) |

#### Structured Text (ST) Code:
```st
Program main
    Var
        Start_Btn AT %IX0.0 : BOOL;
        Stop_Btn AT %IX0.1 : BOOL;
        Relay_Pump AT %QX0.0 : BOOL;
    End_Var
    
    Var
        System_Run : BOOL;
        Delay_Timer : TON;
        Delay_Timer_2 : TOF;
    End_Var

    // Latch logic for Active-Low Start Button
    System_Run := ((NOT Start_Btn) OR System_Run) AND Stop_Btn;
    
    // 3-Second Start Delay
    Delay_Timer(IN := System_Run, PT := T#3s);
    
    // 5-Second Stop Delay / Line Clearing
    Delay_Timer_2(IN := Delay_Timer.Q , PT := T#5s);
    
    // Output assignment
    Relay_Pump := Delay_Timer_2.Q;
End_Program

Configuration Config0
    Resource Res0 ON PLC
        Task Task0 (Interval := T#20ms, Priority := 0);
        Program Inst0 WITH Task0 : main;
    End_Resource
End_Configuration
```

### Phase 3: HMI Dashboard & Bidirectional Control (Modbus TCP)

In this phase, a graphical Human-Machine Interface (HMI) was developed using **Node-RED**. The system now supports bidirectional communication (Closed-Loop) via Modbus TCP, allowing real-time monitoring of the physical relay and providing software override controls from a web browser.

#### 🌐 Modbus Architecture & UI Integration
* **Monitoring (Read):** Reading the actual pump relay state (`%QX0.0` ➔ Modbus Coil `0`) using `FC 1: Read Coils`. The raw boolean data is parsed via a JavaScript function for dynamic UI display.
* **Control (Write):** Soft Start and Stop buttons on the Node-RED dashboard write directly to OpenPLC variables (`%QX10.0` ➔ Modbus Coil `80` and `%QX10.1` ➔ Modbus Coil `81`) using `FC 5: Force Single Coil`.
* **Push-Button Simulation:** A professional 300ms trigger pulse technique was implemented in Node-RED. This mimics a momentary physical push-button press, preventing the software coils from getting stuck in a `TRUE` state.

#### 📊 Updated I/O & Variables Mapping

| Variable Name | Data Type | PLC Address | Modbus Address | Description |
| :--- | :--- | :--- | :--- | :--- |
| Start_Btn | BOOL | `%IX0.0` | Discrete Input 0 | Hardware Active-Low Start |
| Stop_Btn | BOOL | `%IX0.1` | Discrete Input 1 | Hardware Emergency/Stop |
| Relay_Pump | BOOL | `%QX0.0` | Coil 0 | Physical Output & Read Monitor |
| HMI_Start | BOOL | `%QX10.0` | Coil 80 | Software Start (from Node-RED) |
| HMI_Stop | BOOL | `%QX10.1` | Coil 81 | Software Stop (from Node-RED) |

#### 💻 Updated Structured Text (ST) Code:

```pascal
Program main
    Var
        Start_Btn AT %IX0.0 : BOOL;
        Stop_Btn AT %IX0.1 : BOOL;
        Relay_Pump AT %QX0.0 : BOOL;
        HMI_Start AT %QX10.0 : BOOL;
        HMI_Stop AT %QX10.1 : BOOL;
    End_Var
    
    Var
        System_Run : BOOL;
        Delay_Timer : TON;
        Delay_Timer_2 : TOF;
        Stop_CMD : BOOL;
        Start_CMD : BOOL;
    End_Var

    // Combining Hardware (Active-Low) and Software (Active-High) Inputs
    Stop_CMD := Stop_Btn AND (NOT HMI_Stop);
    Start_CMD := (NOT Start_Btn) OR HMI_Start;
    
    // Core Latch Logic
    System_Run := (Start_CMD OR System_Run) AND Stop_CMD;
    
    // 3-Second Start Delay (Noise Filtration)
    Delay_Timer(IN := System_Run, PT := T#3s);
    
    // 5-Second Stop Delay (Pipe Clearing)
    Delay_Timer_2(IN := Delay_Timer.Q , PT := T#5s);
    
    // Output Assignment
    Relay_Pump := Delay_Timer_2.Q;
End_Program

Configuration Config0
    Resource Res0 ON PLC
        Task Task0 (Interval := T#20ms, Priority := 0);
        Program Inst0 WITH Task0 : main;
    End_Resource
End_Configuration
```
#### 📸 Phase 3 Snapshots

**1. Node-RED Logic Flow:**
![Node-RED Flow](Images/Nod_Red_Flow_Phase2.png)

**2. Web HMI Dashboard:**
![HMI Dashboard](Images/HMI_Dashboard_Phase2.png)

**3. OpenPLC Runtime Dashboard:**
![OpenPLC Dashboard](Images/PLC_Dashboard.png)

**4. OpenPLC Live Monitoring:**
![OpenPLC Monitoring](Images/PLC_Monitoring.png)

### Phase 4: Preventive Maintenance Lock & Counter Logic

In industrial scenarios, equipment must be periodically serviced. In this phase, a **Maintenance Lock** system was introduced using a Count-Up (`CTU`) function block. 

#### ⚙️ System Behavior
* **Cycle Tracking:** The system monitors the `System_Run` state. After the pump successfully starts **3 times**, it automatically enters a "Maintenance Lock" mode.
* **Safety Lockout:** Once locked, the main Latch circuit is broken. The pump immediately stops and ignores all physical and software start commands.
* **UI Status & Reset:** The lock status is sent back to the Node-RED dashboard (`Modbus Coil 1`), changing the system status to a red "LOCKED (Needs Service)". A new Maintenance button on the HMI triggers a Modbus write to reset the counter and restore system operation.
* **UI/UX Upgrade:** The HMI was redesigned with an industrial dark theme, high-contrast colored buttons, and bold HTML formatting for better operator visibility.

#### 📊 Updated I/O & Variables Mapping
| Variable Name | Data Type | PLC Address | Modbus Address | Description |
| :--- | :--- | :--- | :--- | :--- |
| Maintenance_Lock | BOOL | `%QX0.1` | Coil 1 | Output to HMI for lock status |
| HMI_Reset | BOOL | `%QX10.2` | Coil 82 | Software Counter Reset (Maintenance) |
| Pump_Counter | CTU | Internal | N/A | Counts pump start cycles (PV=3) |

#### 💻 Updated Structured Text (ST) Code:

```pascal
Program main
    Var
        Start_Btn AT %IX0.0 : BOOL;
        Stop_Btn AT %IX0.1 : BOOL;
        Relay_Pump AT %QX0.0 : BOOL;
        Maintenance_Lock AT %QX0.1 : BOOL;
        
        HMI_Start AT %QX10.0 : BOOL;
        HMI_Stop AT %QX10.1 : BOOL;
        HMI_Reset AT %QX10.2 : BOOL;
    End_Var
    
    Var
        System_Run : BOOL;
        Delay_Timer : TON;
        Delay_Timer_2 : TOF;
        Stop_CMD : BOOL;
        Start_CMD : BOOL;
        Pump_Counter : CTU;
    End_Var

    // Input combination
    Stop_CMD := Stop_Btn AND (NOT HMI_Stop);
    Start_CMD := (NOT Start_Btn) OR HMI_Start;
    
    // Counter Logic (Counts starts, Locks at 3)
    Pump_Counter(CU := System_Run, R := HMI_Reset, PV := 3);
    Maintenance_Lock := Pump_Counter.Q;
    
    // Core Latch Logic with Maintenance Safety Interlock
    System_Run := (Start_CMD OR System_Run) AND Stop_CMD AND (NOT Maintenance_Lock);
    
    // Delay Timers
    Delay_Timer(IN := System_Run, PT := T#3s);
    Delay_Timer_2(IN := Delay_Timer.Q , PT := T#5s);
    
    // Output
    Relay_Pump := Delay_Timer_2.Q;
End_Program

Configuration Config0
    Resource Res0 ON PLC
        Task Task0 (Interval := T#20ms, Priority := 0);
        Program Inst0 WITH Task0 : main;
    End_Resource
End_Configuration
```
📸 Phase 4 Snapshots

**1. Node-RED Flow (Maintenance Logic):**
![Maintenance Logic](Images/Nod_Red_Flow_Phase5.png)
**2. Industrial HMI Dashboard (Dark Theme):**
![OpenPLC Monitoring](Images/HMI_Dashboard_Phase5.png)


### Phase 5: Telegram Bot Integration & Two-Way IoT Control

In this final phase, the local automation system was upgraded to a fully functional Internet of Things (IoT) system. By integrating Node-RED with the Telegram API, the system now sends real-time alerts to the operator's mobile phone and allows for remote bidirectional control.

#### 🎯 Achievements in this Phase:
1. **One-Way Alarming:** Reading the Maintenance Lock status (`%QX0.1`) from OpenPLC and pushing a formatted alert message to a Telegram Bot when the system locks.
2. **Data Filtration (RBE):** Implemented an `rbe` (Filter) node to handle Modbus JSON arrays and prevent message spamming to the Telegram API.
3. **Two-Way Control (Inline Keyboard):** Added an interactive Inline Keyboard button to the Telegram alert message. Operators can now remotely reset the pump fault (`%QX10.2`) directly from the chat interface.
4. **Action Logging:** Sending an acknowledgment (ACK) text message back to the Telegram chat upon successful execution of the reset command.

#### 💻 Data Parsing & Telegram Payload (JavaScript)
Since the Modbus read node outputs an array `{"data":[true,false,...]}`, the following script was used in a Function node to safely parse the data and construct the Telegram payload:

```javascript
let isLocked = false;

// Safe extraction of the lock status
if (msg.payload && msg.payload.data) {
    isLocked = msg.payload.data[0]; 
}

// Check status and send alert with Inline Button
if (isLocked === true) { 
    msg.payload = {
        chatId: "YOUR_CHAT_ID",
        type: "message",
        content: "🚨 <b>System Alert:</b>\nThe pump has entered Maintenance Mode after 3 starts and is currently LOCKED.\nPlease inspect the system.",
        options: {
            parse_mode: "HTML",
            reply_markup: {
                inline_keyboard: [
                    [
                        {
                            text: "🔄 Reset Pump",
                            callback_data: "reset_pump"
                        }
                    ]
                ]
            }
        }
    };
    return msg;
}

return null;
```
📸 Phase 5 Snapshots
**1. Node-RED Flow (Telegram Integration):**
![Telegram Integration](Images/Telegram_With_keys.png)
**2. Telegram Bot UI (Alerts & Remote Reset):**
![Alerts & Remote Reset](Images/node_red_flow_Fifth.png)