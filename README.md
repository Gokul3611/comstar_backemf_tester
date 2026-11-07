# COMSTAR Back-EMF Tester

Industrial PLC application for automated measurement of the back electromotive force
(back-EMF) produced by an electric motor. The machine clamps the device under test
(DUT), couples it to a VFD-driven motor, records three-phase voltage and speed data,
and returns the station to a safe idle state.

> **Safety notice:** This project controls a motor, pneumatic cylinders, and safety
> interlocks. Commissioning must be performed by qualified personnel using the
> approved electrical, pneumatic, and machine-safety procedures. Never bypass the
> E-stop, guard-door, pressure, or dual-hand controls.

## Final project

The final Automation Builder project is:

```text
BACKEMFTESTER.project
```

Open it with **ABB Automation Builder 2.8 Basic** and download it to the target
controller after verifying the hardware configuration, I/O wiring, network settings,
and safety circuits.

| File | Purpose |
|---|---|
| `BACKEMFTESTER.project` | Final project for deployment |
| `BACKEMFTESTERstable.project` | Stable/reference project retained in the repository |
| `Archive.prj` | Archived project package |
| `BACKEMF_TESTER_Commissioning_Documentation.md` | Engineering and commissioning reference |
| `hmi_handoff_package.md` | HMI implementation, Modbus map, screens, and handover checklist |
| `CAD_testbench.jpg` | Test-bench reference image |

## What the system does

1. Verifies safety, air pressure, actuator positions, and drive readiness.
2. Clamps the DUT motor.
3. Extends the coupling sled and performs the alignment routine.
4. Runs the motor at the configured speed through the VFD.
5. Samples U-V, V-W, and W-U back-EMF voltages and measured RPM.
6. Captures peak values and stores time-series test samples.
7. Decelerates, retracts the sled, releases the clamp, and returns to `IDLE`.
8. Latches faults and disables motion when a safety or sequence condition fails.

## Main features

- 10 ms cyclic PLC task with IEC 61131-3 Structured Text logic.
- Nine-state machine: `INIT`, `IDLE`, `WAIT_CLAMP`, `WAIT_MOVE`,
  `TEST_READY`, `TESTING`, `TEARDOWN`, `WAIT_UNCLAMP`, and `FAULT`.
- Dual-hand start control for the test cycle.
- Pneumatic clamp and sled control with position feedback and travel timeouts.
- E-stop, door, air-pressure, sensor-conflict, and actuator supervision.
- VFD Modbus TCP polling, command handling, readiness checks, and telemetry.
- Hardware-counter RPM measurement using a six-tooth speed sensor.
- Three-phase analog voltage scaling, peak capture, and 100 ms sample logging.
- HMI register interface with heartbeat watchdog and diagnostic values.
- Retained completed-cycle counter.

## Hardware and software

| Component | Configuration |
|---|---|
| PLC | ABB AC500 PM5032-T-ETH |
| Engineering software | ABB Automation Builder 2.8 Basic |
| PLC task | `Task`, 10 ms cycle, 20 ms watchdog |
| Main program | `PLC_MAIN` |
| Drive communication | Modbus TCP over Ethernet |
| Actuation | Pneumatic clamp and coupling sled |
| Measurement | Three phase-to-phase voltage channels and RPM pulse input |

The project uses ABB/System libraries for standard timers, Ethernet, Modbus TCP,
onboard I/O, diagnostics, and process management. The exact library versions and
controller settings are documented in
[`BACKEMF_TESTER_Commissioning_Documentation.md`](BACKEMF_TESTER_Commissioning_Documentation.md).

## I/O summary

### Inputs

| Input | Function |
|---|---|
| DI0–DI1 | Left and right dual-hand start buttons |
| DI2–DI3 | Clamp work and home sensors |
| DI4 | RPM pulse input |
| DI5–DI6 | Sled work and home sensors |
| DI7 | Air-pressure-OK feedback |
| DI8 | E-stop relay feedback |
| DI9 | Reset button |
| DI10 | Second clamp home sensor |
| DI11 | VFD running feedback |
| DC12 | VFD healthy/ready feedback |
| DC13 | Second clamp work sensor |

### Outputs

| Output | Function |
|---|---|
| DO0 | VFD run |
| DO1–DO2 | VFD jog forward/reverse |
| DO3–DO4 | Clamp extend/retract solenoids |
| DO5–DO6 | Sled extend/retract solenoids |
| DO7 | Main air-valve command |

The complete terminal map, signal types, and wiring notes are in the commissioning
document.

## HMI and Modbus TCP

The PLC communicates with the HMI using a holding-register interface. The HMI
provides test parameters and a continuously changing heartbeat; the PLC returns
state, fault, speed, voltage, cycle-count, and VFD telemetry.

Important values include:

| Data | Description |
|---|---|
| Speed setpoint | 1–18,000 RPM, subject to PLC limits |
| Acceleration/deceleration | VFD ramp times |
| Test duration | Configured test window |
| State and fault | Machine state and latched fault code |
| Voltage results | U-V, V-W, and W-U values, transmitted with documented scaling |
| Sample buffer | Timestamp, RPM, and three voltage values |

The HMI heartbeat must update continuously. If it stops changing within the configured
timeout, the PLC raises an HMI communication fault and blocks a new cycle. See
[`hmi_handoff_package.md`](hmi_handoff_package.md) for the authoritative register
layout, IP settings, screen requirements, scaling, and commissioning checklist.

## Data acquisition

During `TESTING`, the PLC records samples containing:

- Timestamp at 100 ms resolution
- Measured RPM
- U-V voltage
- V-W voltage
- W-U voltage

The acquisition data is intended for HMI result screens and CSV extraction. Peak
voltage values and the retained completed-cycle count are also exposed for production
and diagnostic use.

## Commissioning workflow

1. Install the required Automation Builder libraries and open
   `BACKEMFTESTER.project`.
2. Confirm the PM5032-T-ETH controller, firmware, task cycle, and onboard I/O
   configuration.
3. Verify all digital, configurable, and analog wiring against the commissioning
   document.
4. Configure the VFD network address and Modbus TCP settings.
5. Configure the HMI network address and Modbus server/register mapping.
6. Verify the HMI heartbeat before attempting a test.
7. Test E-stop, guard door, air pressure, reset, and both start buttons.
8. Commission clamp and sled motion at safe conditions before enabling the motor.
9. Verify VFD ready/running feedback, RPM measurement, voltage scaling, and sample
   retrieval.
10. Run a complete test cycle and confirm that the station returns to `IDLE`.

Do not treat a successful download as proof of machine safety. Complete the documented
commissioning checks and validate every protective function on the real equipment.

## Repository documentation

- [Commissioning and engineering documentation](BACKEMF_TESTER_Commissioning_Documentation.md)
- [HMI handoff package](hmi_handoff_package.md)
- [HMI specification](HMI_Spec_Complete_DAQ_BackEMF.pdf)
- [HMI handover specification](HMI_Handover_Specification_V1.pdf)
- [ABB PM5032-T-ETH details](DETAILS%20PM5032-T-ETH.pdf)
- [Test-bench CAD image](CAD_testbench.jpg)

The remaining PDF files contain the applicable drive, PLC, and HMI manuals used during
design and commissioning.

## Project status

This repository contains the final PLC project and supporting engineering deliverables
for the COMSTAR Back-EMF Tester. PLC source elements are stored inside the Automation
Builder project package rather than as standalone source files in this repository.
