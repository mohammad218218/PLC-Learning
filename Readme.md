# 🏭 PLC Learning: Embedded SoftPLC on Raspberry Pi

Welcome to my **PLC Learning** repository! This project documents my journey in building an industrial-grade SoftPLC system using an Embedded Linux environment. The goal is to bridge the gap between embedded hardware (Raspberry Pi) and industrial automation standards (IEC 61131-3), culminating in a fully functional IoT-enabled control system.

## 📂 Project Structure

To demonstrate the evolution of the system architecture, this repository is divided into two main stages of development:

*   **[📁 01_Basic_Latch_Control](./01_Basic_Latch_Control/)**
    *   *Phases 1 to 5:* Covers the initial hardware setup, wiring, noise debouncing, and the implementation of standard Latch-based logic for motor control. It also includes the initial Modbus TCP integration and basic HMI creation.
*   **[📁 02_Advanced_State_Machine](./02_Advanced_State_Machine/)**
    *   *Phase 6 (Current):* A complete architectural refactoring of the control logic into a strict **State Machine (`CASE OF`)**. This phase solves industrial relay chattering bugs and introduces bidirectional IoT control with the Telegram API (Remote Fault Reset).

---
*Built with OpenPLC v3, Node-RED, and Raspberry Pi.*