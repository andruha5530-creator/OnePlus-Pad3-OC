# SM8750 CPU frequency control map

Source baseline: `OnePlusOSS/android_kernel_oneplus_sm8750`, branch `oneplus/sm8750_b_16.0.0_oneplus_13`.

## Confirmed control path

### 1. Qualcomm cpufreq hardware driver

Primary file:

`drivers/cpufreq/qcom-cpufreq-hw.c`

The driver:

- exposes the `qcom-cpufreq-hw` cpufreq driver;
- supports `qcom,cpufreq-hw`, `qcom,cpufreq-epss`, and `qcom,cpufreq-epss-pdmem`;
- reads a hardware frequency LUT;
- builds the cpufreq frequency table from that LUT;
- optionally creates/uses Linux OPP entries;
- writes a selected table index to the hardware performance-state register;
- supports a separate PDMEM mapping for domains with `perf_lock_support`;
- contains LMh/DCVS thermal-throttling handling.

Important functions:

- `qcom_cpufreq_hw_read_lut()` — constructs the available frequency table.
- `qcom_cpufreq_hw_target_index()` — writes the selected index to `reg_perf_state`.
- `qcom_cpufreq_hw_fast_switch()` — fast path that writes the selected index directly.
- `qcom_cpufreq_hw_cpu_init()` — reads the CPU `qcom,freq-domain`, maps the domain resource, reads the LUT, and initializes OPP state.

## 2. LUT is more important than a guessed OPP node

The driver reads frequency and voltage information from MMIO LUT registers:

- frequency LUT base: `reg_freq_lut`
- voltage LUT base: `reg_volt_lut`
- performance state: `reg_perf_state`

For the generic Qualcomm HW implementation in this source, the driver uses a LUT row size of 32 bytes. The EPS/EPSS variants use a different register layout.

This means an overclock patch must first determine which compatible/domain the Pad 3 actually binds to and which LUT is populated at runtime.

## 3. Device tree is the next critical layer

The driver obtains the CPU domain from:

`qcom,freq-domain`

and maps a corresponding platform memory resource.

Therefore the next investigation target is the SM8750/Sun/Pineapple/Pad 3 device-tree hierarchy that supplies:

- CPU nodes;
- `qcom,freq-domain`;
- cpufreq-hw compatible;
- MMIO resources;
- clocks named `xo` and `alternate`;
- optional PDMEM resources;
- thermal/LMh IRQs.

## 4. SCMI/DCVS is a second control layer

The same OnePlus source contains Qualcomm SCMI/DCVS components under:

`drivers/soc/qcom/dcvs/`

Notable files include:

- `qcom_scmi_client.c`
- `c1dcvs_scmi_v2.c`
- `dynpf_scmi.c`
- `dcvs_epss.c`
- `Kconfig`
- `Makefile`

The build/module lists also include `qcom_cpucp`, `qcom_scmi_vendor`, `qcom_scmi_client`, `dcvs_fp`, and `qcom-dcvs`.

This strongly suggests that frequency selection is not safely modeled as a simple static OPP-table edit.

## 5. Thermal limiting

The cpufreq driver has explicit LMh/DCVS handling. It can read the current hardware vote/domain state and convert a throttled frequency into thermal pressure.

Therefore a successful higher-frequency request does not automatically mean the device will sustain it.

## Current hypothesis

The likely path is:

CPU policy
→ `qcom-cpufreq-hw`
→ frequency-table/LUT index
→ hardware performance-state register
→ Qualcomm CPU/firmware/hardware control

with SCMI/DCVS and thermal mechanisms influencing the available/requested performance state.

## Next step

Before changing any frequency value, locate the exact Pad 3 device-tree node and identify:

1. its cpufreq-hw compatible;
2. CPU frequency-domain indices;
3. LUT/PDMEM resources;
4. the source of the LUT contents;
5. the highest stock LUT entry;
6. whether the final/boost entry is firmware-defined.

Only after these are known should we design the first experimental patch.
