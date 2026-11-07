# 🏭 HMI ENGINEER HANDOFF PACKAGE
## ABB Back-EMF Tester — PM5032-T-ETH (Automation Builder 2.8)

> **Project**: `BACKEMFTESTER.project`  
> **PLC**: ABB PM5032-T-ETH  
> **PLC IP**: `192.168.0.1`  
> **HMI IP (expected)**: `192.168.0.7` (`0xC0A80007`)  
> **Protocol**: Modbus TCP, Port 502  
> **Task Cycle**: 10ms  
> **Date**: 2026-06-15  

---

## 1. Network Architecture

```
┌─────────────────┐     Modbus TCP      ┌─────────────────┐
│  ABB PM5032      │◄───── Port 502 ────►│  HMI Panel      │
│  192.168.0.1     │     Unit ID: 1      │  192.168.0.7    │
│                  │                     │                  │
│  ETHx_MOD_MAST   │     Unit ID: 2      │                  │
│  (Modbus Client) ├────────────────────►│  (VFD Gateway)  │
└─────────────────┘                     └─────────────────┘
         │
         │ Hardwired DO0-DO7, DI0-DI11
         ▼
   ┌───────────┐
   │ VFD Drive │ (Run/Jog via Digital I/O)
   │ Clamp/Sled│ (Pneumatic Solenoids)
   │ Sensors   │ (Proximity, Pressure, E-Stop)
   └───────────┘
```

> [!IMPORTANT]
> The PLC is the **Modbus Client (Master)**. It actively polls the HMI. The HMI must act as a **Modbus TCP Server (Slave)** on Unit ID 1 at `192.168.0.7:502`.

---

## 2. COMMUNICATION PROTOCOL

### 2.1 Polling Cycle (PLC-initiated, Round-Robin)

The PLC's `FB_CommManager` runs a round-robin state machine with these steps every cycle:

| Step | Direction | FCT | Unit ID | Target | Address | Count | Description |
|------|-----------|-----|---------|--------|---------|-------|-------------|
| 1 | PLC ← HMI | FC3 (Read) | 1 | HMI | 0 | 5 | Read test parameters + heartbeat |
| 2 | PLC → HMI | FC16 (Write Multiple) | 1 | HMI | 10 | 7 | Write live telemetry to HMI |
| 3 | PLC ← VFD | FC3 (Read) | 2 | VFD | 0x8100 | 7 | Read VFD status registers |
| *On-demand* | PLC → VFD | FC6 (Write Single) | 2 | VFD | Various | 1 | Speed/Accel/Decel/Reset writes |

### 2.2 Heartbeat Protocol

> [!CAUTION]
> **If the heartbeat stops changing for 5 seconds, the PLC declares an HMI comm fault and blocks new test cycles.**

The HMI must **continuously increment** a counter in its holding register address `4` (Modbus address 40005). The PLC reads this every cycle. If the value does not change within 5 seconds, `bHMICommFault` triggers.

**Implementation**: Use a 1-second timer on the HMI to increment a UINT counter. It will naturally roll over at 65535.

---

## 3. MODBUS REGISTER MAP

### 3.1 HMI → PLC (HMI Server Holding Registers — Read by PLC via FC3)

These registers live in the **HMI's Modbus server memory**. The PLC reads them.

| HMI Register | PLC Variable | Type | Unit | Range | Description |
|---|---|---|---|---|---|
| **40001** | `GVL_Modbus.uiSpeedRPM` | UINT | RPM | 1–18000 | Test speed setpoint |
| **40002** | *(Reserved — read as index 1)* | — | — | — | *(Index 1 in buffer, not used)* |
| **40003** | `GVL_Modbus.uiAccelTime` | UINT | seconds | 0–600 | VFD acceleration ramp time |
| **40004** | `GVL_Modbus.uiDecelTime` | UINT | seconds | 0–600 | VFD deceleration ramp time |
| **40005** | `GVL_Modbus.uiTestTimeSec` | UINT | seconds | 0–3600 | Test duration (0 = default 5s) |

> [!NOTE]
> The heartbeat (`uiHeartbeat`) is mapped at `%MW4` but the PLC reads it from the HMI register buffer index position. The HMI should put it at **register address 1** (the second register in the FC3 read block starting at address 0). Check: The PLC reads 5 registers starting at address 0, so indices are `[0]=Speed, [1]=Heartbeat?, [2]=AccelTime, [3]=DecelTime, [4]=TestTimeSec`. The PLC code maps: `awHMIReadBuffer[0]` → SpeedRPM, `awHMIReadBuffer[2]` → AccelTime, `awHMIReadBuffer[3]` → DecelTime, `awHMIReadBuffer[4]` → TestTimeSec. **Index [1] is read but not mapped to a named parameter** — this is where the heartbeat counter goes.

**Corrected HMI server register layout (address 0–4):**

| Index | HMI Address | Content |
|---|---|---|
| 0 | 40001 | Speed RPM (1–18000) |
| 1 | 40002 | **Heartbeat Counter** (increment every 1s) |
| 2 | 40003 | Accel Time (seconds) |
| 3 | 40004 | Decel Time (seconds) |
| 4 | 40005 | Test Duration (seconds, 0 = 5s default) |

### 3.2 PLC → HMI (Written by PLC via FC16 to HMI Register Address 10)

The PLC writes 7 registers to the HMI's holding register starting at address 10.

| HMI Register | Buffer Index | PLC Source | Type | Unit | Description |
|---|---|---|---|---|---|
| **40011** | [0] | `GVL_Modbus.uiState` | UINT | enum | Machine state code (see §4) |
| **40012** | [1] | `GVL_Modbus.uiFault` | UINT | enum | Active fault code (see §5) |
| **40013** | [2] | `GVL_DACQ.uiMeasuredRPM` | UINT | RPM | Measured RPM (from pulse counter) |
| **40014** | [3] | `GVL_Modbus.uiVfdActualRpm` | UINT | RPM | VFD reported RPM |
| **40015** | [4] | Peak Voltage UV × 100 | UINT | V×100 | Phase U-V peak (divide by 100 for volts) |
| **40016** | [5] | Peak Voltage VW × 100 | UINT | V×100 | Phase V-W peak (divide by 100 for volts) |
| **40017** | [6] | Peak Voltage WU × 100 | UINT | V×100 | Phase W-U peak (divide by 100 for volts) |

> [!TIP]
> **Voltage Scaling**: All voltage values are transmitted as `UINT × 100`. To display to operator: `Display = RegisterValue / 100.0` with 2 decimal places. Example: register value `3456` → display `34.56 V`.

---

## 4. STATE MACHINE CODES

The PLC writes an integer state code to register **40011**. Use this for screen navigation, animation, and status indicators.

| Code | State Name | Description | HMI Behavior |
|---|---|---|---|
| **0** | `INIT` | Power-up / post-fault recovery. Retracting sled, releasing clamp. | Show "INITIALIZING..." spinner. Disable all controls. |
| **1** | `IDLE` | Ready for operator. Waiting for dual-hand start. | Show **green "READY"**. Enable parameter entry. Show "Press Both Start Buttons". |
| **2** | `WAIT_CLAMP` | Clamp extending onto motor. | Show "CLAMPING..." with animation. Disable controls. |
| **3** | `WAIT_MOVE` | Sled extending + coupling alignment (wiggle routine). | Show "COUPLING..." with jog animation. |
| **4** | `TEST_READY` | VFD readiness check (500ms delay). | Show "STARTING TEST..." brief pause indicator. |
| **5** | `TESTING` | Motor running at test speed, collecting data. | **Primary screen**: Show live RPM, voltages, elapsed time, progress bar. |
| **6** | `TEARDOWN` | Motor decelerating, sled retracting. | Show "COMPLETING..." with cooldown animation. |
| **7** | `WAIT_UNCLAMP` | Releasing clamp after sled is home. | Show "UNCLAMPING..." |
| **8** | `FAULT` | Fault condition — all motion stopped. | **RED FAULT BANNER**. Show fault code from register 40012. Show "Press RESET". |
| **9** | `UNKNOWN` | Invalid/unexpected state. | Show "SYSTEM ERROR — Contact maintenance". |

### State Flow Diagram (for screen navigation):
```
INIT(0) → IDLE(1) → WAIT_CLAMP(2) → WAIT_MOVE(3) → TEST_READY(4)
                                                          ↓
              IDLE(1) ← WAIT_UNCLAMP(7) ← TEARDOWN(6) ← TESTING(5)
                
              Any State → FAULT(8) → [Reset] → INIT(0)
```

---

## 5. FAULT CODE TABLE

The PLC writes a fault code integer to register **40012**. Display the corresponding message.

| Code | Fault Name | Severity | Operator Message | Root Cause |
|---|---|---|---|---|
| **0** | `NONE` | — | *(no fault)* | System healthy |
| **1** | `ESTOP` | 🔴 Critical | **EMERGENCY STOP ACTIVE** — Release E-Stop, then press RESET | E-stop circuit open |
| **2** | `AIR_LOSS` | 🔴 Critical | **AIR PRESSURE LOW** — Check compressor and supply line | Pressure switch < threshold |
| **3** | `DOOR_OPEN` | 🟡 Warning | **SAFETY DOOR OPEN** — Close door and press RESET | Door interlock sensor |
| **4** | `CLAMP_TIMEOUT` | 🟠 Mechanical | **CLAMP TIMEOUT** — Check cylinder, air supply, and sensors | Clamp didn't reach position in 5s |
| **5** | `CLAMP_LOST_POSITION` | 🔴 Critical | **CLAMP LOST DURING TEST** — Motor may be loose! Inspect before retry. | Clamp sensor dropped mid-test |
| **6** | `SLED_FWD_TIMEOUT` | 🟠 Mechanical | **SLED FORWARD TIMEOUT** — Check sled cylinder and coupling | Sled didn't extend in 10s |
| **7** | `SLED_LOST_POSITION` | 🔴 Critical | **SLED POSITION LOST** — Coupling may have disengaged | Sled sensor dropped mid-test |
| **8** | `SLED_REV_TIMEOUT` | 🟠 Mechanical | **SLED RETRACT TIMEOUT** — Check sled cylinder and home sensor | Sled didn't retract in 10s |
| **9** | `SENSOR_CONFLICT` | 🔴 Critical | **SENSOR CONFLICT** — Both sensors active. Wiring fault! | Both home+work sensors TRUE |
| **10** | `CMD_CONFLICT` | 🟠 Logic | **COMMAND CONFLICT** — Internal error | Shouldn't occur in normal operation |
| **11** | `WATCHDOG` | 🟠 Timeout | **STATE TIMEOUT** — Machine stuck. Press RESET to recover. | Stuck in transitional state >30s |
| **12** | `INVALID_STATE` | 🔴 Critical | **SYSTEM ERROR** — Invalid machine state. Power cycle if persists. | State machine corruption |
| **13** | `VFD_NOT_READY` | 🟡 Drive | **VFD NOT READY** — Check drive power and configuration | Drive didn't assert ready signal |
| **14** | `VFD_FAULT` | 🔴 Drive | **VFD FAULT** — Drive reported error. Check VFD display. | Drive internal fault or failed to run |
| **15** | `VFD_COMM_LOSS` | 🔴 Comms | **VFD COMMUNICATION LOST** — Check Ethernet cable to gateway | Modbus to VFD failed >2s |
| **16** | *Unmapped* | 🔴 Unknown | **UNKNOWN FAULT** — Contact engineering | Shouldn't occur |

---

## 6. ADDITIONAL PLC → HMI DATA (via Modbus TCP Server on PLC)

The PLC also runs a **Modbus TCP Server** on port 502. The HMI can optionally read these %MW registers directly:

### 6.1 VFD Telemetry (for diagnostics screen)

| PLC Register | Variable | Type | Scaling | Description |
|---|---|---|---|---|
| `%MW40` (40040) | `uiVfdActualFreq_x100` | UINT | ÷100 = Hz | VFD output frequency |
| `%MW41` (40041) | `uiVfdActualRpm` | UINT | direct | VFD reported RPM |
| `%MW42` (40042) | `uiVfdActualCurrent_x100` | UINT | ÷100 = A | VFD output current |
| `%MW43` (40043) | `uiVfdFaultCode` | UINT | direct | VFD internal fault code |
| `%MW44` (40044) | `uiVfdCommOk` | UINT (0/1) | boolean | VFD Modbus comm healthy |
| `%MW45` (40045) | `uiVfdReadyFb` | UINT (0/1) | boolean | VFD ready feedback |
| `%MW46` (40046) | `uiVfdRunningFb` | UINT (0/1) | boolean | VFD running feedback |

### 6.2 Machine Status

| PLC Register | Variable | Type | Description |
|---|---|---|---|
| `%MW47` (40047) | `uiState` | UINT | Machine state (same as §4) |
| `%MW48` (40048) | `uiFault` | UINT | Fault code (same as §5) |
| `%MW49` (40049) | `uiHealthy` | UINT (0/1) | System health (Safe AND No Fault AND HMI OK) |
| `%MW50` (40050) | `uiHMIFault` | UINT (0/1) | HMI heartbeat timeout active |

### 6.3 Production Statistics

| PLC Register | Variable | Type | Description |
|---|---|---|---|
| `%MW20` (40020) | `uiCyclesLow` | UINT | Cycle count (low 16 bits) |
| `%MW21` (40021) | `uiCyclesHigh` | UINT | Cycle count (high 16 bits) |

**To reconstruct 32-bit cycle count**: `TotalCycles = (uiCyclesHigh × 65536) + uiCyclesLow`

### 6.4 Live Voltage Readings

| PLC Register | Variable | Type | Scaling | Description |
|---|---|---|---|---|
| `%MW30` (40030) | `uiMotorVoltage_x100` | UINT | ÷100 = V | Motor voltage (instantaneous) |
| `%MW31` (40031) | `uiPeakVoltage_x100` | UINT | ÷100 = V | Peak voltage captured during test |

### 6.5 Actual Speed

| PLC Register | Variable | Type | Description |
|---|---|---|---|
| `%MW10` (40010) | `uiActSpeedRPM` | UINT | Actual measured RPM |

---

## 7. DATA ACQUISITION — BULK SAMPLE RETRIEVAL

During each test, the PLC logs up to **200 samples** at 100ms intervals (10 samples/second). This data is stored in a struct array mapped to Modbus memory for HMI bulk read.

### 7.1 Sample Structure (`ST_TestSample` — 5 words per sample)

| Field | Type | Scaling | Description |
|---|---|---|---|
| `uiTime_x10` | UINT | ÷10 = seconds | Timestamp (0.1s resolution) |
| `uiRPM` | UINT | direct | Measured RPM at sample time |
| `uiUV_x100` | UINT | ÷100 = V | Phase U-V voltage |
| `uiVW_x100` | UINT | ÷100 = V | Phase V-W voltage |
| `uiWU_x100` | UINT | ÷100 = V | Phase W-U voltage |

### 7.2 Memory Layout

| PLC Register Range | Content | Count |
|---|---|---|
| `%MW100` (40100) | `uiSampleCount` — Number of valid samples (0–200) | 1 register |
| `%MW1000`–`%MW1999` | `astSamples[0..199]` — 200 × 5 words = 1000 registers | 1000 registers |

### 7.3 Bulk Read Protocol

1. **Wait** for state to return to IDLE (state code = 1) after a test
2. **Read** `%MW100` to get sample count `N`
3. **Read** `%MW1000` through `%MW(1000 + N*5 - 1)` in chunks of ≤125 registers (Modbus limit)
4. **Parse** every 5 consecutive words as one `ST_TestSample`
5. **Plot** time-series charts: RPM vs Time, UV/VW/WU voltages vs Time

> [!TIP]
> Also check `bLogComplete` at `%MX100.0` — when TRUE, the buffer is full (200 samples). You can use this as a trigger for auto-download.

### 7.4 Example Chart Data (what to display)

For each completed test, generate:
- **Line Chart 1**: RPM (Y) vs Time (X) — "Speed Profile"
- **Line Chart 2**: UV, VW, WU Voltages (Y, 3 series) vs Time (X) — "Back-EMF Waveform"
- **Summary Table**: Peak UV, Peak VW, Peak WU, Average RPM, Test Duration

---

## 8. RECOMMENDED HMI SCREENS

### Screen 1: MAIN / DASHBOARD
- Large state indicator (colored banner: Green=IDLE, Blue=Running, Red=FAULT)
- Current state text from §4
- Live RPM (large numeric display)
- Live peak voltages (3 numeric displays with labels UV, VW, WU)
- Cycle counter
- Health indicator (green dot if register 40049 = 1)
- Fault banner (only shown when state = 8)

### Screen 2: PARAMETER SETUP
- Speed RPM input field (1–18000, write to HMI register 40001)
- Accel Time input (seconds, write to HMI register 40003)
- Decel Time input (seconds, write to HMI register 40004)  
- Test Duration input (seconds, write to HMI register 40005)
- "Apply" confirmation (the PLC reads continuously, but consider user confirmation UX)

> [!WARNING]
> **Speed is clamped** by the PLC to 1–18000 RPM. The VFD frequency command is capped at 1350 (135.0 Hz). Values outside range will be silently clamped.

### Screen 3: TEST RESULTS
- Triggered after test completes (transition from state 6→7→1)
- Fetch sample data per §7
- RPM vs Time chart
- 3-phase voltage vs Time chart
- Peak voltage summary
- PASS/FAIL indicator (if thresholds are defined on HMI side)
- Print/Export button

### Screen 4: VFD DIAGNOSTICS
- VFD frequency, RPM, current (from §6.1 registers)
- VFD comm status
- VFD fault code
- VFD ready/running indicators
- Useful for commissioning and troubleshooting

### Screen 5: FAULT HISTORY / ALARMS
- Log of fault codes with timestamps (HMI-side logging)
- Fault descriptions from §5

### Screen 6: I/O STATUS (Commissioning)
- Read PLC digital inputs/outputs for wiring verification
- See §9 for I/O map

---

## 9. PHYSICAL I/O WIRING MAP (For Commissioning Reference)

### Digital Inputs (PM5032 Onboard)

| Terminal | PLC Variable | Signal | Description |
|---|---|---|---|
| DI0 | `GVL_IO.bBtnLeft` | N/O | Left-hand start button |
| DI1 | `GVL_IO.bBtnRight` | N/O | Right-hand start button |
| DI2 | `GVL_IO.bRPMPulse` | Pulse | RPM proximity sensor (6 teeth/rev) |
| DI3 | `GVL_IO.bClampHome` | N/O | Clamp retracted proximity |
| DI4 | `GVL_IO.bClampWork` | N/O | Clamp extended proximity |
| DI5 | `GVL_IO.bSledWork` | N/O | Sled extended proximity |
| DI6 | `GVL_IO.bSledHome` | N/O | Sled retracted proximity |
| DI7 | `GVL_IO.bAirPressOK` | N/O | Air pressure switch (1s debounce) |
| DI8 | `GVL_IO.bEStopOK` | N/C | E-Stop relay (TRUE = safe) |
| DI9 | `GVL_IO.bBtnReset` | N/O | Reset pushbutton |
| DI10 | `GVL_IO.bDoorClosed` | N/C | Door interlock (TRUE = closed) |
| DI11 | `GVL_IO.bVfdRunningFb` | N/O | VFD running feedback relay |

### Digital Outputs (PM5032 Onboard)

| Terminal | PLC Variable | Signal | Description |
|---|---|---|---|
| DO0 | `GVL_IO.bDriveRun` | Relay | VFD Run command |
| DO1 | `GVL_IO.bDriveJogFwd` | Relay | VFD Jog Forward |
| DO2 | `GVL_IO.bDriveJogRev` | Relay | VFD Jog Reverse |
| DO3 | `GVL_IO.bValveClampFwd` | 24V | Clamp extend solenoid |
| DO4 | `GVL_IO.bValveClampRev` | 24V | Clamp retract solenoid |
| DO5 | `GVL_IO.bValveSledFwd` | 24V | Sled extend solenoid |
| DO6 | `GVL_IO.bValveSledRev` | 24V | Sled retract solenoid |
| DO7 | `GVL_IO.bAirValveCmd` | 24V | Main air supply valve (E-Stop AND Door) |

### Configurable I/O (DC12/DC13)

| Terminal | PLC Variable | Direction | Description |
|---|---|---|---|
| DC12 | `GVL_IO.bVFDHealthyFb` | Input | VFD Healthy/Ready feedback |
| DC13 | *(unmapped)* | Output | Available |

---

## 10. TIMING PARAMETERS (Configurable in PLC)

These are set in `GVL_Config` and can be adjusted in the PLC project if needed:

| Parameter | Default | PLC Variable | Description |
|---|---|---|---|
| Clamp Timeout | **5 seconds** | `tClampTimeout` | Max time for clamp to reach position |
| Sled Timeout | **10 seconds** | `tSledTimeout` | Max time for sled to reach position |
| Alignment Jog Speed | **5 RPM** | `uiAlignSpeedRPM` | Wiggle speed during coupling |
| State Watchdog | **30 seconds** | `tWatchdog` | Max time in any transitional state |
| VFD Check Delay | **500 ms** | `tVFDCheckDelay` | Delay before checking VFD ready |
| HMI Heartbeat Timeout | **5 seconds** | *(hardcoded in PLC_MAIN)* | HMI comm loss detection |
| VFD Comm Loss | **2 seconds** | *(hardcoded in FB_StateMachine)* | VFD Modbus timeout |
| VFD Ready Loss | **2 seconds** | *(hardcoded in FB_StateMachine)* | VFD ready signal debounce |
| VFD Run Check | **5 seconds** | *(hardcoded in FB_StateMachine)* | Time allowed for VFD to assert running |
| Modbus Retry Strikes | **10 failures** | *(hardcoded in FB_CommManager)* | Consecutive errors before comm fail |

---

## 11. SAFETY NOTES FOR HMI ENGINEER

> [!CAUTION]
> 1. **The HMI cannot start the machine.** Start requires **physical dual-hand buttons** (both pressed simultaneously). This is hardwired, not Modbus. The HMI is data-display and parameter-entry only.
> 2. **The HMI cannot stop a running test.** The only ways to stop are: E-Stop, door open, or let the test timer expire. There is no software stop command.
> 3. **HMI heartbeat is safety-critical.** If your heartbeat stops, the PLC will fault. Make this the very first thing you implement and test.
> 4. **Never bypass fault screens.** When state = 8 (FAULT), the operator MUST see the fault banner. Don't allow navigation away from fault state without acknowledging.

---

## 12. QUICK COMMISSIONING CHECKLIST

- [ ] HMI IP configured to `192.168.0.7`, subnet `255.255.255.0`
- [ ] Modbus TCP Server enabled on HMI, port 502, Unit ID 1
- [ ] Heartbeat counter incrementing in HMI register 40002 (index 1)
- [ ] Verify PLC reads: Speed, Accel, Decel, TestTime from HMI
- [ ] Verify PLC writes: State, Fault, RPM, Voltages appear on HMI
- [ ] Fault banner displays correctly for all 16 fault codes
- [ ] State transitions animate correctly through full cycle
- [ ] Bulk data download works after test completion
- [ ] Voltage values display correctly (÷100 scaling verified)
- [ ] Cycle counter reads correctly (32-bit reconstruction)
- [ ] Parameter limits enforced on HMI side (speed 1–18000, etc.)
