# ESP32 provisional RAM budget

Design allowances, not measurements or vendor benchmarks. Same firmware budget in v5, v6 normal and v6 RAM-peak; the latter changes Raspberry allowances only. Wi-Fi/Bluetooth and PSRAM are not included.

| Holder | KiB | Contents |
|---|---:|---|
| Zephyr | 128 | Kernel objects, runtime data, system/interrupt stacks |
| Drivers | 32 | Driver state, excluding shared communication buffers |
| Shared communication | 16 | UART rings, frames, queues and port copies, once |
| Heap | 32 | Reserved dynamic allocation pool, even if not fully used |
| Motor control | 6 | Stack 4 + private data 2 |
| IMU acquisition/conversion/filter | 8 | Stack 4 + private data 4 |
| Battery | 3 | Stack 2 + private data 1 |
| Telemetry | 6 | Stack 4 + private data 2 |
| Command RX | 6 | Stack 4 + private data 2 |
| Command guard | 3 | Stack 2 + private data 1 |
| Total | 240 | 245760 bytes |

A provisional data-RAM capacity of 320 KiB (327680 bytes) leaves 80 KiB / 25% unallocated. This pool is a planning assumption, not a claim that Zephyr exposes exactly this amount. The existing 520 KByte physical memory declaration is retained and is distinct from this smaller data pool. The analysis checks both physical Memory_Size and SEI RAMCapacity: use the RAMCapacity comparison for this budget.

The 16 KiB shared allocation includes both driver-facing rings and application-facing queues/copies; do not add them again to driver or thread budgets. Heap storage is separate from static allocations. Six thread budgets include their private stacks. Kernel allowance includes only system/interrupt stacks, not those six stacks. No parent firmware budget or RAMActual is assigned.

Replace assumptions using the selected Zephyr build's linker map, RAM section totals, configured application/system stacks, driver configuration, heap reservation and measured stack/heap high-water marks. Account separately for code in internal RAM and hardware reservations when defining available data capacity. Increasing radio features or enabling other libraries requires a new estimate.

Reinstantiate and run Analyze Resource Allocations (Bound). Expect 245.760 decimal KB budget and 327.680 decimal KB RAM capacity. The report may reuse budgets in its Actual column; this is not measured usage. ReqSpec is unchanged: this is resource feasibility analysis, not a new formal RAM-reserve requirement. No CPU timings, queues or runtime allocation settings are changed by these properties.
