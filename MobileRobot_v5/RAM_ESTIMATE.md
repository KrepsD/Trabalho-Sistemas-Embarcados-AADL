# Preliminary RAM budget

Status: engineering allowances, NOT measurements or published benchmarks.
No Raspberry or Ubuntu test has been run. These scenarios are not confidence
intervals or guaranteed worst-case bounds. They must not be entered as RAMActual.

Assumptions: headless Linux/chonOS, one JVM with 1-3 simple agents, D435i depth
640x480 Z16 at 15 FPS, depth-to-LaserScan, no RGB/point cloud/camera IMU,
latest-sample sensor queues, 10x10 m room, 30 minutes of mapping/navigation.
RViz, Gazebo, IDEs and builds are outside this target budget. OS image, agent
workload and actual ROS configuration remain unknown.

| Exclusive accounting bucket | Lean MiB | Planning MiB | Pressure MiB |
|---|---:|---:|---:|
| chonOS, base services and residual kernel demand | 200 | 300 | 450 |
| JaCaMo Embedded MAS and one JVM | 150 | 250 | 450 |
| Nav2 | 150 | 250 | 400 |
| SLAM Toolbox | 100 | 200 | 400 |
| D435i driver and processing buffers | 80 | 150 | 250 |
| Depth-to-LaserScan | 30 | 60 | 100 |
| Odometry/state estimation | 20 | 40 | 60 |
| Separate bridge processes/logic | 30 | 60 | 100 |
| Shared ROS2/DDS/library allowance, counted once | 60 | 100 | 150 |
| **Total** | **820** | **1410** | **2360** |

Application rows represent private/incremental demand, including their own
queues. Shared middleware/library pages belong only in the shared row.
The JVM row includes heap, stacks, metadata, JIT and native allocations.
Embedded MAS bridge code in that same JVM must not be charged again as a
separate JVM. OS demand excludes memory already removed from usable MemTotal.
This partition is provisional until process measurements are available.

With a 20% reserve, minimum usable RAM is demand/0.8:
1025 MiB (lean), 1762.5 MiB (planning), 2950 MiB (pressure).
Nominal installed RAM is not identical to usable MemTotal. This budget does
not establish CPU, USB, scheduling or software compatibility.

## Calculable payloads

- One depth frame: 640*480*2 = 614400 bytes = 0.5859375 MiB.
- Three to six simultaneous buffers: 1.76-3.52 MiB, already inside perception
  allowances. Actual buffer count is unknown; DDS queue depth is not that count.
- Payload throughput: 9,216,000 bytes/s. Do not multiply by 30 minutes to get
  resident RAM: buffers are reused unless recording/retention is configured.
- Proposed map resolution: 0.05 m/cell. One 10x10 m grid has 40000 cells;
  at one byte/cell it is 0.038 MiB. This is not total Nav2/SLAM memory.
- SLAM scans, graph nodes and solver workspaces can grow even in a small room.
  A queue limit does not bound the complete pose graph. Document scan admission,
  history and interactive-mode settings before treating any value as a bound.

## Ubuntu desktop experiment

1. Record OS, architecture, ROS/RMW/package versions, agent count, JVM options,
   camera streams, queue settings, map resolution and launch configuration.
2. Measure the host baseline with applications stopped, then enable perception,
   SLAM, Nav2 and agents/bridges in stages to identify marginal costs.
3. Run the complete stack for 1800 seconds with all functions active and record
   startup separately. Use actual D435i data or a documented replay/simulation.
4. Sample /proc/meminfo every second; use MemAvailable/MemTotal for host reserve.
   Inspect swap, OOM events, restarts and completed navigation goals separately.
5. Sample /proc/PID/smaps_rollup Pss about every 10 seconds for application PIDs.
   Count each process once. Do not sum thread RSS or add the shared estimate to
   already measured PSS. PSS needs sufficient permissions and kernel support.
6. Exclude Gazebo/RViz/IDE processes from application attribution. Global desktop
   MemAvailable still includes them, so it cannot prove Raspberry headroom.
7. Compare peak and growth over 30 minutes, and repeat with the same workload.
   An x86 desktop run calibrates estimates; it does not verify the ARM target.

The planning column is now encoded as SEI::RAMBudget: 800 MiB on application
threads, 300 MiB on an OS accounting process, 210 MiB on a perception accounting
process, and 100 MiB on a shared-runtime accounting process. The 60 MiB bridges
allowance is split equally between the two bridge threads. All are bound to
nav_computer.rpi_ram. No duplicate parent budget or RAMActual was assigned.
Accounting processes do not implement services, drivers, conversion or scheduling.
Task execution times are unchanged. RAMCapacity is 1 GiByte nominal; the
1410 MiB budget exceeds capacity before requiring 20% reserve. Usable RAM
and runtime ReqSpec evidence remain pending.

Sources for accounting mechanisms, NOT for the numerical allowances:

- https://docs.kernel.org/filesystems/proc.html
- https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html
- https://github.com/SteveMacenski/slam_toolbox
- https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/core_servers/costmap_2d/
- https://github.com/embedded-mas/embedded-mas

See ANALYSIS_CORRECTIONS.md for corrected binary capacities, leaf bindings and analysis limitations.
