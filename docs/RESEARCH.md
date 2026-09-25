# DVFS research checklist

## Target
OnePlus Pad 3 / Qualcomm SM8750 (Snapdragon 8 Elite).

## Do not assume
A conventional Linux OPP table may not be the runtime authority for CPU frequency on this platform. Qualcomm cpufreq hardware control, SCMI/DCVS, firmware performance levels, device-tree configuration, and thermal/power management may interact.

Therefore a patch adding a single `opp-4400000000` entry is **not** considered valid until the control path is confirmed in the exact OnePlus source tree.

## Investigation targets

- `drivers/cpufreq/`
- Qualcomm cpufreq hardware driver
- SCMI / performance-domain interfaces
- Qualcomm DCVS-related code
- SM8750 device-tree CPU nodes
- OPP tables and CPU performance levels
- thermal zones and cooling maps
- power/voltage regulator constraints
- vendor modules and device-tree overlays
- build configuration for the Pad 3 kernel

## Experimental principle

Change one control point at a time. Never invent voltage values. First establish that the kernel can request the new frequency and that the hardware/firmware actually accepts it.

Initial experimental target should be a small frequency step above the verified stock ceiling, only after the authoritative DVFS mechanism is identified.

## Evidence to record

For every experiment record:
- source commit
- exact modified files
- build configuration
- requested frequency
- observed frequency
- temperatures
- sustained-load stability
- crashes/reboots
- recovery procedure

No experimental patch should be described as working until it has been tested on real hardware.