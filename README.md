# StasisGate – RTL Clock Gating Core for CubeSat Sensor Nodes

**StasisGate** is a synthesizable Verilog power management IP that implements VLSI‑style integrated clock gating (ICG) for CubeSat payloads. It dynamically stops clocks to the radio, IMU, and temperature sensor based on orbital phase (sunlit/eclipse) and a 2‑bit task queue. The design is simulated entirely in EDA Playground – no hardware required.

**Key achievement:** 71.9% reduction in simulated dynamic power compared to an ungated baseline.

## Concept

Modern smartphones (Apple Silicon, Snapdragon) use **clock gating** inside the chip to save power. StasisGate brings that same technique to a CubeSat sensor node – but at the RTL level, inside an FPGA or ASIC.

A dedicated finite‑state machine (FSM) monitors:
- **sunlit / eclipse** (from an orbit timer)
- **pending tasks** (radio or sensor requests)

It then enables or disables the clock to each domain using **latch‑based integrated clock gates** – the exact cell used in real VLSI libraries.


## System Architecture:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         StasisGate – RTL Architecture                       │
└─────────────────────────────────────────────────────────────────────────────┘

External inputs
┌──────────────────┐     ┌─────────────────┐
│ push_task[1:0]   │────▶│                 │
│ (mission comp)   │     │   task_queue    │
└──────────────────┘     │ 2-bit pending   │
                         │ register        │
┌──────────────────┐     │ (pop clears)    │
│ rst_n            │────▶│                 │
└──────────────────┘     └────────┬────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │ pending[1:0]          │
          │ fsm_pop               ▼                       │
          │                 ┌─────────────────┐           │
          └─────────────────│   power_fsm     │◀──────────┘
                            │ Moore FSM       │
                            │ ACTIVE/IDLE/    │
                            │ DEEP_SLEEP      │
                            └────────┬────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │ radio_en                   │ imu_en                     │ temp_en
        ▼                            ▼                            ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  clk_gate_unit  │          │  clk_gate_unit  │          │  clk_gate_unit  │
│ (ICG latch)     │          │ (ICG latch)     │          │ (ICG latch)     │
└────────┬────────┘          └────────┬────────┘          └────────┬────────┘
         │                            │                            │
         │ en_clk_radio               │ en_clk_imu                 │ en_clk_temp
         ▼                            ▼                            ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  Radio domain   │          │   IMU domain    │          │ Temp sensor     │
│  (gated clock)  │          │  (gated clock)  │          │  domain (gated) │
└────────┬────────┘          └────────┬────────┘          └────────┬────────┘
         │                            │                            │
         │ toggle_radio               │ toggle_imu                 │ toggle_temp
         └────────────────────────────┼────────────────────────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ toggle counters │
                             │ (count rising   │
                             │  edges of gated │
                             │  clocks)        │
                             └────────┬────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ power_estimator │
                             │ P = toggles ×   │
                             │ Csw × Vdd²      │
                             └────────┬────────┘
                                      │
                                      ▼
                             ┌─────────────────┐
                             │ power_mW[31:0]  │
                             │ fsm_state[2:0]  │
                             │ sunlit, eclipse │
                             └─────────────────┘


┌─────────────────┐
│   orbit_timer   │
│ counts cycles   │
└────────┬────────┘
         │
         │ sunlit, eclipse
         ▼
┌─────────────────┐
│   power_fsm     │
│   (as above)    │
└─────────────────┘





## The FSM transitions:

| Current State | Condition -> Next State |
|---------------|-------------------------|
| ACTIVE        | no tasks & sunlit → IDLE |
| ACTIVE        | no tasks & eclipse → DEEP_SLEEP |
| IDLE          | tasks or eclipse → ACTIVE |
| IDLE          | eclipse & no tasks → DEEP_SLEEP |
| DEEP_SLEEP    | sunlit & tasks → ACTIVE |
| DEEP_SLEEP    | sunlit & no tasks → IDLE |

Clock enables per state:
- **ACTIVE** – radio (if task pending), IMU on, temperature on
- **IDLE** – radio off, IMU off, temperature on (low‑rate monitoring)
- **DEEP_SLEEP** – all clocks stopped

## 🔗 Live Simulation

[Click here to run StasisGate on EDA Playground](https://www.edaplayground.com/x/Yric)
