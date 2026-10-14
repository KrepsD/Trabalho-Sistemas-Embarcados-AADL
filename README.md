# Trabalho-Sistemas-Embarcados-AADL

Final assignment for **DAS410056 — Projeto e Desenvolvimento de Sistemas
Embarcados** (Embedded Systems Design), Control and Automation Engineering,
Federal University of Santa Catarina (UFSC). The project states the
requirements and models, in **AADL** (Architecture Analysis and Design Language)
using the **OSATE** tool, the software/hardware architecture of a wheeled mobile
robot that combines an **ESP32** (real-time sensing/actuation) with a **Raspberry Pi**
(navigation, mapping and agent execution).

Authors: Daniel Victor Krepsky, Dener, Henrique Bueno de Morais.

## What the project is about

A mobile robot needs sensors, motors and software to share limited CPU, memory
and communication resources. Instead of building the robot first, this work models
the proposed architecture in AADL and uses OSATE's built-in analyses to check,
**before implementation**, whether the design can plausibly meet its timing, memory,
communication and power budgets. Results come from the model and from
estimated parameters — they are **not** experimental measurements from real
hardware.

### Proposed architecture

- **ESP32** (RTOS: Zephyr, 2 cores) — acquires encoder/IMU/battery data, filters the
  IMU signal, drives the motors, and exchanges telemetry/commands with the
  Raspberry Pi over a serial link.
- **Raspberry Pi** (OS: chonOS, ROS 2 based) — runs odometry, SLAM
  (SLAM Toolbox), navigation (Nav2), and JaCaMo agents integrated with ROS 2.
- **Intel RealSense D435i** camera — perception sensor, connected to the Raspberry
  Pi over USB.

See `System_diagram.jpeg` for a high-level view of the architecture.

## Repository layout

The model was built incrementally; each `MobileRobot*` folder is a self-contained
OSATE/AADL project representing one stage of that evolution:

| Folder | Stage / focus |
|---|---|
| `MobileRobot` | v1 — ESP32 only: sensor data, motor control, dual-core task split. |
| `MobileRobot_v2` | Adds IMU filtering, battery monitoring; first scheduling/latency/bus-load reports. |
| `MobileRobot_v3` | Introduces the Raspberry Pi hardware model and power analysis. |
| `MobileRobot_v4` | Adds Raspberry Pi software (navigation/mapping/agents) and the first `ReqSpec` requirements file. |
| `MobileRobot_v5` | Reference model with **Raspberry Pi 3 B+**; adds goals/stakeholders files and RAM-budget notes (`RAM_ESTIMATE.md`, `ESP32_RAM_ESTIMATE.md`, `ANALYSIS_CORRECTIONS.md`). |
| `MobileRobot_v6` | Most detailed model, with **Raspberry Pi 4 B**; adds the D435i USB connection, power-supply network (battery + regulators), and OS scheduler components (`platform_os.aadl`). Includes **two scenarios**: a reference case and a higher-memory-demand ("peak") case. See `PLATFORM_MODELS.md` for provenance of components reused from the ProVANT library. |

Each `MobileRobot_vN` folder follows the same internal structure:

- **`*.aadl` files** — the AADL model, split by concern:
  - `robo_data.aadl` — shared data types (encoder/IMU/battery readings, motor
    commands, telemetry, perception/navigation/agent data).
  - `esp32_hardware.aadl` / `esp32_software.aadl` — ESP32 cores, RAM/flash,
    I2C/serial buses, devices, and its tasks (motor control, IMU acquisition,
    telemetry, command handling, battery reading).
  - `raspberry_hardware.aadl` / `raspberry_software.aadl` (v3+) — Raspberry Pi
    cores/RAM/storage, the D435i camera (v6), and its tasks (serial comms,
    odometry, SLAM, Nav2, agents, agent↔ROS 2 integration).
  - `platform_os.aadl` (v6) — virtual processors representing the Zephyr and
    chonOS schedulers.
  - `robot_usb.aadl`, `robot_power.aadl` (v6) — USB bus and the battery/regulator/
    power-rail network.
  - `processor_properties.aadl`, `electricity_properties.aadl` — reusable AADL
    property sets (clock frequency, electrical properties), adapted from the
    ProVANT library.
  - `robot_system.aadl` — top-level integration: connects all components, binds
    tasks to cores and memory budgets to RAM, and declares the scenario(s)
    used for analysis (e.g. v6's reference vs. peak-memory configurations).
  - `robot_requirements.reqspec` (v4+), `robot_goals.goals`,
    `robot_stakeholders.org` (v5+) — requirements engineering artifacts (see
    below).
- **`instances/`** — OSATE model instances (`.aaxl2`) generated from the
  declarative model, plus `reports/` with the **raw analysis output** (CSV/XLS/result
  files) for Bound Resource Budgets (RAM), Bus Load, Flow Latency and Scheduling
  Analysis (and Power, from v3 on).
- **`diagrams/`** — exported OSATE graphical diagrams (`.aadl_diagram`) of the
  instantiated model.

Top-level supporting files:

- `System_diagram.jpeg` — architecture overview diagram.

## Requirements engineering (ReqSpec)

Requirements are declared in `robot_requirements.reqspec` and linked to the AADL
model elements they constrain. They are grouped into:

1. **ESP32 task deadlines** — e.g. motor control and command validation: 10 ms;
   IMU acquisition/filter, telemetry, command reception: 20 ms; battery reading:
   500 ms.
2. **Raspberry Pi task deadlines** — serial comms and odometry: 20 ms; Nav2: 50 ms;
   SLAM: 100 ms; agent↔ROS 2 integration: 200 ms; agent execution: 500 ms.
3. **Raspberry Pi memory and interface requirements** — at least 20% of usable RAM
   free; USB/power interface contracts (v6).

Each requirement has an id, a natural-language description, and a `value predicate`
that OSATE/ReqSpec-style tooling can compare against analysis results (e.g.
`worstResponse <= deadlineLimit`). It can be automatically verified with the ALISA tool.

Explicitly **out of scope**: total cost, battery autonomy, robot dimensions/mass,
navigation accuracy, environmental operating conditions, and safety requirements
(e.g. emergency stop).

## How to open and reproduce the analyses

1. Install **OSATE** (Eclipse-based AADL toolset) — https://osate.org.
2. Import the desired `MobileRobot_vN` folder via
   **File > Import > General > Existing Projects into Workspace** (leave
   "Copy projects into workspace" unchecked, per `MobileRobot_v6/PLATFORM_MODELS.md`).
3. Open `robot_system.aadl`, instantiate the system implementation(s) of interest
   (e.g. `robot.impl`, and in v6 also `robot.peak_30min`) to regenerate the model
   under `instances/`.
4. Run the OSATE analyses from the **Analyses** menu: Scheduling (MAST/fixed
   priority), Bound Resource Budgets, Bus Load, Flow Latency, and (v3+) Power.
   Compare the generated reports against the limits declared in
   `robot_requirements.reqspec`.

## Summary of results (v5 vs. v6)

- **Memory**: Raspberry Pi 3 B+ (v5, 1024 MiB) is **insufficient** — the ~1410 MiB
  RAM budget exceeds its capacity. Raspberry Pi 4 B (v6, 4096 MiB) has margin in
  both the reference (65.6% free) and peak-memory (42.4% free) scenarios. The
  ESP32's 240 KiB budget fits its 320 KiB provisional capacity (25% margin) in all
  scenarios.
- **Scheduling**: all 12 modeled tasks meet their deadlines; busiest cores are ESP32
  core 1 (67.5%, motor control + IMU) and Raspberry Pi core 1 (60%, SLAM).
- **Latency**: IMU-activation-to-filtered-result ≤ 15.03 ms (limit 20 ms);
  IMU-activation-to-odometry ≤ 83.04 ms (limit 100 ms).
- **Bus load**: I2C (IMU) 0.6 KB/s, serial (commands/telemetry) 3.7 KB/s, USB
  (camera depth stream, v6) 9216 KB/s — all within declared capacities.
- **Power** (v6): all power rails stay within budget, but the motor rail uses 100%
  of its allotted 30 W, leaving no margin.

These results support choosing the Raspberry Pi 4 B and the current task/core
distribution, but they rely on modeling hypotheses (estimated execution times,
aggregated application tasks, provisional memory/power figures) and do not
replace hardware tests.

## Limitations and suggested next steps

The model simplifies real application internals (e.g. the camera driver and
depth-to-SLAM data conversion are represented as a single logical connection), and
does not cover cost, battery life, size/mass, navigation accuracy, environmental
conditions, or safety behavior (e.g. emergency stop). Suggested follow-up work:
Hardware-in-the-Loop simulation, physical robot testing to refine timing/memory/
power estimates, deeper ROS 2 perception integration, evaluating additional
features (e.g. YOLO-based object detection), and automating the comparison
between analysis results and the ReqSpec requirements.
