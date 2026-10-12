# Analysis corrections (2026-10-12)

- Bind the nine RAM budget holders directly to RAM. No aggregate parent budget.
- Nominal RAM uses GiByte (1024^3 bytes), not decimal GByte. Usable RAM still needs measurement.
- ESP32 inter-core data uses shared RAM; I2C carries only the IMU transaction.
- Shared-memory copies have a provisional 0..10 us latency per connection.
- Keep the 100 ms IMU-to-odometry requirement. Response_Time values use the current fixed-priority schedule: telemetry 0.5..1.1 ms, bridge 0.3..0.8 ms, odometry 0.2..1.4 ms. Recompute after changing the task set. These are analytical assumptions, not measured times.
- IMU response 5.03..15.03 ms already includes the conservative I2C wait. Do not count that transfer again in the activation-to-result flow.
- v6 power connections are directional to avoid counting supplies and loads twice.
- v6 OS virtual processors remain structural. Tasks bind directly to physical cores because the installed Flow Latency analysis interprets bound virtual processors as ARINC653 partitions. OS execution overhead is not separately modeled.

Regenerate v5 robot.impl, v6 robot.impl and v6 robot.peak_30min. Run Scheduling, Bound Resource Budgets, Bus Load, Flow Latency and (v6) Power again. For latency: asynchronous, partition end, deadline, empty queue, queuing enabled.

Expected RAM budgets: v5 normal 1410/1024 MiB (fails); v6 normal 1410/4096 MiB (65.58% nominal reserve); v6 peak 2360/4096 MiB (42.38% nominal reserve). These do not prove 20% usable RAM remains available for 30 minutes.

Existing reports were preserved and describe the previous models. New reports must be generated from new instances.

Validation: loaded AADL/ReqSpec with installed OSATE libraries; instantiated all system implementations without instantiation errors or warnings; checked 12 physical task bindings; used BoundResourceAnalysis memory selection to verify nine budget holders; ran FlowLatencyAnalysisSwitch with AS-PE-DL-EQ-EQL (approximately 83 ms maximum against 100 ms). Checked instantiated connection bindings and, for v6, directional power totals: battery 55.557 W, logic 17.5 W, motors 30 W, USB 3.5 W. OSATE UI reports were not overwritten.
