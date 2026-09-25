# SM8850 vs SM8750: EPSS/LUT and Pad 3 frequency candidates

## Source basis

Compared:
- OnePlus 15 SM8850 OOS 16 kernel: `travismills82/oneplus15-sm8850-oos-kernel`, `main`
- OnePlus SM8750 kernel: `OnePlusOSS/android_kernel_oneplus_sm8750`, branch `oneplus/sm8750_b_16.0.0_oneplus_13`
- Qualcomm Snapdragon 8 Elite Gen 5 public specification

## EPSS/LUT result

Both kernels use the same fundamental Qualcomm EPSS CPUFreq hardware model:

| Item | SM8750 | SM8850 |
|---|---|---|
| Driver | qcom-cpufreq-hw | qcom-cpufreq-hw |
| EPSS compatible | qcom,cpufreq-epss | qcom,cpufreq-epss |
| LUT max entries | 40 | 40 |
| Frequency LUT | 0x100 | 0x100 |
| Voltage LUT | 0x200 | 0x200 |
| Performance-state register | 0x320 | 0x320 |
| LUT row size | 4 bytes | 4 bytes |
| DCVS control | 0xb0 | 0xb0 |
| Domain state | 0x20 | 0x20 |
| Interrupt clear | 0x308 | 0x308 |

The important difference is not a different EPSS register format. The SM8750 driver has additional support around PDMEM/perf-lock and OPlus telemetry/thermal integration. The SM8850 OnePlus 15 source has a cleaner/commonized implementation, but its EPSS register/LUT layout is the same.

## What the LUT actually contains

The kernel does **not** contain a portable list such as `4.32 GHz -> voltage X`.

At boot, `qcom_cpufreq_hw_read_lut()` reads the hardware EPSS LUT:
- frequency information from `base + 0x100 + index * 4`
- voltage information from `base + 0x200 + index * 4`
- the selected state is written to `base + 0x320`

The driver converts the LUT fields into runtime cpufreq/OPP entries. Therefore, copying a Gen 5 frequency number into a DT OPP table does not by itself create a new EPSS hardware state.

## Pad 3 candidate frequencies

Pad 3 stock target used for this research:
- prime domain: 4.32 GHz
- performance domains: 3.53 GHz

The following are **candidate test points**, not verified SM8750-safe frequencies and not a claim that these states exist in the Pad 3 EPSS LUT.

| Level | Prime | Performance | Delta prime | Purpose |
|---|---:|---:|---:|---|
| Stock | 4.32 | 3.53 | 0% | Baseline |
| A | 4.40 | 3.56 | +1.9% | First low-risk probe |
| B | 4.45 | 3.58 | +3.0% | Small step |
| C | 4.50 | 3.60 | +4.2% | Moderate probe |
| D | 4.55 | 3.61 | +5.3% | Intermediate |
| E | 4.60 | 3.63 | +6.5% | Gen-5-class clock target |
| F | 4.65 | 3.63 | +7.6% | Experimental |
| G | 4.70 | 3.63 | +8.8% | High experimental |
| H | 4.74 | 3.63 | +9.7% | Gen 5 advertised ceiling; not a Pad 3 target to assume safe |

## Recommended research order

1. Dump the actual Pad 3 EPSS LUT and voltage LUT from the running device.
2. Decode all 40 rows per frequency domain.
3. Identify the highest real non-turbo state and its voltage.
4. Compare that real SM8750 LUT against the SM8850 LUT from a device using SM8850.
5. Only then choose a software modification.
6. Keep thermal/LMh limits unchanged for the first experiment.

## Key conclusion

SM8850 is useful as a **reference for the EPSS mechanism**, but not as a drop-in frequency/voltage table for SM8750. The register interface is strongly compatible at the driver level, while the actual silicon-specific LUT and voltage data remain hardware/firmware controlled.

The most useful next step is therefore an **on-device Pad 3 LUT dumper**, not a blind 4.74 GHz DT patch.
