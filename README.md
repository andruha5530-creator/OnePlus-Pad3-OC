# OnePlus Pad 3 OC

Research and kernel-modification workspace for investigating CPU frequency control on the OnePlus Pad 3 (Snapdragon 8 Elite / SM8750).

## Status

- Repository initialized.
- No device-flashing patch is claimed to be working yet.
- First goal: identify the actual SM8750 CPU DVFS control path used by the OnePlus kernel/device tree/Qualcomm firmware.
- Any frequency increase must be validated incrementally with thermal, stability, and recovery checks.

## Planned work

1. Map the kernel source and device-tree CPU frequency/DVFS implementation.
2. Identify OPP/performance-level definitions, cpufreq driver hooks, SCMI/DCVS interfaces, and thermal/power limits.
3. Determine which values are actually authoritative at runtime.
4. Build a minimal experimental kernel change without changing voltage assumptions.
5. Validate boot and runtime behavior before considering higher frequencies.
6. Keep stock/recovery paths documented.

## Safety

Overclocking can cause boot loops, crashes, excessive heat, accelerated wear, data loss, or a device that requires recovery/reflashing. Do not flash experimental images without a known-good recovery path and backups of the relevant stock partitions.

## Sources

Official OnePlus kernel repositories for SM8750 should be treated as the primary source for implementation details.
