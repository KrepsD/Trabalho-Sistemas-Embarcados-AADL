# Platform models adapted from ProVANT

MobileRobot_v6 is a full copy of v5. Re-instantiate `robot_system::robot.impl`: copied instances/reports are old v5 results. Import v6 with **File > Import > General > Existing Projects into Workspace**, leaving **Copy projects into workspace** unchecked when using this existing workspace folder.

## Changes and provenance

| File | Change | ProVANT source in `Library` |
|---|---|---|
| `raspberry_hardware.aadl` | D435i camera, USB host, power distribution and four OS scheduling domains | `devices/devices-camera_usb.aadl` |
| `robot_usb.aadl` | USB bus with payload capacity and transfer-time assumptions | `buses/buses-usb.aadl` |
| `platform_os.aadl` | Virtual schedulers for Zephyr and chonOS | `os.aadl`: Root_Scheduler and POSIX_Scheduler |
| `robot_power.aadl` | Battery, 5 V / 6 V regulators and electrical buses | `devices/devices-battery_14v_4ah.aadl`, `devices/devices-regulator_tsi10n.aadl`, `buses/buses-misc.aadl` |
| `electricity_properties.aadl` | Original property definitions copied unchanged | `property_set/electricity_properties.aadl` |
| `esp32_hardware.aadl` | Two OS scheduling domains, board supply and motor supply | Adaptations to the existing board |
| `robo_data.aadl` | Depth image payload size | D435i scenario calculation |
| `robot_system.aadl` | USB/power connections and thread-to-OS bindings | Integration |
| Requirements/goals/stakeholder files | Unique V6 identifiers and new Raspberry interface contracts | Existing v5 requirements |

Original library roles are retained. The camera's picture port keeps the existing `state` name and its optional `proc_evt` output remains unused. USB remains a bus, OS remains a virtual processor, and battery/regulators remain electrical devices connected through bus access. No library functionality is represented as executable firmware. The obsolete REAL annex check and unrelated OS variants were not imported.

## Camera and USB

- Host is **Raspberry Pi 4 Model B, 4 GB**, BCM2711 / Cortex-A72, with a 1.8 GHz frequency assumption and the camera connected to USB 3. The 20 MB/s effective USB allowance is retained conservatively.
- D435i scenario: depth Z16, 640 x 480, 15 frames/s. RGB, published IR, point cloud and camera IMU remain disabled assumptions.
- One frame: `640 * 480 * 2 = 614400 bytes`.
- Depth payload: `614400 * 15 = 9216000 bytes/s = 73.728 Mbit/s`.
- Connection allocation: **85 Mbit/s**, reserving margin for ancillary metadata. CameraInfo is host metadata, not a second raw camera image stream.
- Bus budget: **100 Mbit/s**; assumed effective payload capacity: **160 Mbit/s (20 MB/s)**. This is not the USB 2.0 physical line rate or a measured Raspberry throughput.
- At 50 ns/byte and 0.125–1 ms fixed delay: **30.845–31.720 ms/frame**. This simple estimate excludes additional host contention, capture/exposure, driver and ROS2 processing. USB host contention is not explicitly modeled.
- Queue depth and actual supported profile must still be checked on the device. The camera model does not configure ROS2 or librealsense.

The existing logical camera-to-SLAM path is preserved. A driver/depth-to-LaserScan conversion component and the perception-to-Nav2 obstacle branch are **not implemented by this change**. SLAM Toolbox cannot use a raw depth image directly. These remain the next software modeling step. No additional perception RAM is charged: the existing 210 MiB allowance already covers the driver/conversion estimate; the frame Data_Size describes traffic, not a second application RAM budget.

## OS and scheduling

Each existing thread now binds to a per-core `virtual processor`, which binds to the same physical core used in v5. Four chonOS domains represent one OS over four cores, not four OS installations. The same applies to the two Zephyr domains.

The fixed-priority policy is retained from v5. It assumes pinned, preemptive tasks and does not prove equivalence to normal Linux/ROS2 executors or Zephyr SMP, cooperative threads and time slicing. Periods, execution times, deadlines, priorities and RAM budgets are unchanged. No OS execution overhead has been invented. The existing OS RAM allowance is counted once.

## Electrical allocation — provisional, not measured

The user selected the initial **2S, 7.4 V nominal** scenario. Pack capacity, current ratings and regulator models are not selected hardware.

| Segment | Allocation | Meaning |
|---|---:|---|
| Battery rail | 6.0–8.4 V, assumed 10 A; 60 W guaranteed-design envelope at 6 V | Hypothesis, not a battery specification |
| 5 V regulator | 4 A / 20 W output; 22.223 W input | Assumed 90% efficiency; full output reserved |
| 6 V regulator | 5 A / 30 W output; 33.334 W input | Assumed 90% efficiency; full output reserved |
| Raspberry input | 15 W | Board supply envelope includes downstream USB |
| ESP32 board and sensors | 2.5 W | Provisional board allowance |
| Raspberry USB supply | 6 W shared | 5 V / 1.2 A total USB envelope |
| D435i | 3.5 W | Planning allowance; measure selected profile |
| Drivers and motors together | 30 W | Provisional envelope; check startup/stall against actual motors |

At full allocation, battery demand is **55.557 W of 60 W**. The 5 V branch reserves **17.5 W of 20 W**, and the motor branch **30 W of 30 W**: the 5 V branch retains 2.5 W allocation margin; the motor branch retains none. This is a sizing scenario, not a declaration that the chosen hardware is adequate. The USB branch reserves **3.5 W of 6 W**. Do not sum budgets across voltage levels: downstream loads are already included in their upstream allocations.

The 6 V regulator must retain regulation near minimum battery voltage (buck-boost, or a higher cutoff with a suitable buck regulator). Battery capacity in Ah, usable energy, discharge curve, cable losses and transient currents are not known; no 30-minute electrical autonomy claim is made. `battery_sensor` remains the acquisition abstraction; `power_battery` is the physical power source.

## Checks to run in OSATE

1. Import v6, refresh and instantiate `robot.impl`.
2. Budget > Analyze Bus Load: inspect `perception_usb` and the camera connection.
3. Scheduling analysis: confirm all 12 threads remain assigned to their original six physical cores through the virtual schedulers.
4. Budget > Analyze Power Requirements: inspect each electrical rail separately; the values are design allocations above.
5. RAM analysis: no duplicate camera/OS RAM allocation was added.

Electrical and USB properties are analysis inputs, not automatic physical guarantees. Copied v5 reports are not evidence for v6.

## Reference documentation

- [RealSense USB 2 support and reduced camera profiles](https://www.realsenseai.com/news-insights/usb2-support-for-intel-realsense-technology/)
- [Raspberry Pi power and peripheral documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)
- [OSATE scheduling implementation](https://github.com/osate/osate2/blob/master/analyses/org.osate.analysis.resource.management/src/org/osate/analysis/resource/management/handlers/Schedule.java)

## Validation performed

- Installed OSATE libraries: model validation returned 0 errors; instance creation returned 0 errors and 0 warnings.
- The installed InstanceModelUtil.isBoundToProcessor (used by scheduling analysis) resolved all 12 threads to exactly one original physical core each, across six cores.
- V5 source files match the pre-change backup byte-for-byte.
- GUI scheduling, USB load, power and RAM reports were not rerun in the editor; run them on the new instance.

## Pi 4 / 4 GB comparison

- Official hardware reference: https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/
- V5 stays Pi 3 B+ / 1 GB. V6 now has 4096 MiB nominal modeled capacity and the same 1410 MiB estimated demand: 2686 MiB nominal headroom (65.58%).
- The 20 percent reserve requires at least 1762.5 MiB usable RAM. Actual usable MemTotal and the 30-minute runtime requirement still require deployment evidence; nominal capacity is not a measurement.
- No execution times were reduced. A higher clock property alone does not establish higher schedulability.
- The 5 V regulator allocation was raised to 4 A because it supplies both the Pi (15 W including USB) and ESP32 (2.5 W). This is a hypothetical regulator rating, not a selected part.
