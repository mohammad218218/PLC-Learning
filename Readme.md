# 🏭 PLC Learning: Embedded SoftPLC on Raspberry Pi

Welcome to my **PLC Learning** repository! This project documents my journey in building an industrial-grade SoftPLC system using an Embedded Linux environment[cite: 2]. The goal is to bridge the gap between embedded hardware (Raspberry Pi) and industrial automation standards (IEC 61131-3), culminating in a fully functional IoT-enabled control system[cite: 2].

## 📂 Project Structure

To demonstrate the evolution of the system architecture, this repository is divided into three main stages of development:

*   **[📁 01_Basic_Latch_Control](./01_Basic_Latch_Control/)**
    *   *Phases 1 to 5:* Covers the initial hardware setup, wiring, noise debouncing, and the implementation of standard Latch-based logic for motor control[cite: 2]. It also includes the initial Modbus TCP integration and basic HMI creation[cite: 2].
*   **[📁 02_Advanced_State_Machine](./02_Advanced_State_Machine/)**
    *   *Phases 6 to 12:* A complete architectural refactoring of the control logic into a strict **State Machine (`CASE OF`)**[cite: 2]. This stage introduces Modular Architecture (Function Blocks/OOP), safety interlocks (Dry Run, Network Watchdog), and bidirectional IoT control with the Telegram API and SQLite logging.
*   **[📁 03_Process_Control_PID](./03_Process_Control_PID/)**
    *   *Phase 13+ (Current):* Transitioning from discrete (On/Off) logic to Continuous Process Control. This phase introduces analog signal processing, PID controllers, and VFD (Variable Frequency Drive) simulation for precise fluid pressure regulation.

---
*Built with OpenPLC v3, Node-RED, and Raspberry Pi.*[cite: 2]