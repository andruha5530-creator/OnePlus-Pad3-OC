# SM8750 CPU frequency control map

Source baseline: `OnePlusOSS/android_kernel_oneplus_sm8750`, branch `oneplus/sm8750_b_16.0.0_oneplus_13`.

Pad 3 identification: LineageOS device metadata identifies OnePlus Pad 3 as codename `erhai`, model `OPD2415`, platform `SM8750 / sun`, with 8 Oryon cores and stock CPU frequencies of 2 x 4.32 GHz + 6 x 3.53 GHz. This is used here only to identify the target device; it is not itself proof that a higher frequency is safe or possible.

## 1. Qualcomm cpufreq hardware driver

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

## 2. Important discovery: SM8750 uses EPSS-style hardware domains

In the Qualcomm SM8750 device-tree source, the common `pineapple.dtsi` defines:

```dts
cpufreq_hw: qcom,cpufreq-hw {
    compatible = "qcom,cpufreq-epss";
    reg = <0x17D91000 0x1000>,
          <0x17D92000 0x1000>,
          <0x17D93000 0x1000>,
          <0x17D94000 0x1000>;
    reg-names = "freq-domain0",
                "freq-domain1",
                "freq-domain2",
                "freq-domain3";
    clocks = <&rpmhcc RPMH_CXO_CLK>, <&gcc GCC_GPLL0>;
    clock-names = "xo", "alternate";
    interrupts = <...>;
    #freq-domain-cells = <1>;
};
```

This is a major correction to the initial simple-OPP hypothesis: the relevant control point is an EPSS hardware frequency domain, not a normal `opp-4320000000` table that can simply be extended in DTS.

The common SM8750 source also maps CPU cores to these domains through `qcom,freq-domain`. The generic Qualcomm `pineapple.dtsi` example has four domains and uses domain IDs 0, 1, 2 and 3.

## 3. LUT is more important than a guessed OPP node

The driver reads frequency/voltage information from MMIO LUT registers.

For the EPSS implementation, the register layout differs from the generic `qcom,cpufreq-hw` implementation. The exact LUT contents therefore have to be inspected from the running Pad 3 hardware or from a matching compiled DT/firmware image.

The key question is no longer simply “where is 4.32 GHz written in DTS?”, but:

**What EPSS performance-state table is actually exposed by the Pad 3 firmware/hardware, and what is its highest valid entry?**

## 4. CPU → frequency-domain mapping

The common SM8750 Qualcomm device tree shows the expected mechanism:

```
CPU node
  |
  +-- qcom,freq-domain = <&cpufreq_hw DOMAIN>
  |
  +-- cpufreq_hw
       |
       +-- freq-domainN MMIO resource
       +-- EPSS hardware LUT
       +-- performance-state index
```

The exact Pad 3 `erhai` device tree is still the missing link in the public OnePlus source tree. The public source currently exposes the SoC-level `pineapple`/SM8750 definitions, while the device-specific tree is assembled separately.

## 5. SCMI/DCVS is a second control layer

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

This means a kernel-only frequency-table modification may be insufficient if firmware rejects or clamps performance states.

## 6. Thermal limiting

The cpufreq driver has explicit LMh/DCVS handling. A higher requested performance state does not imply that the tablet will sustain that state under load.

For experimentation we therefore need to keep the stock thermal/LMh path intact initially.

## 7. What we should NOT patch yet

Do **not** add a fake `opp-4400000000` or `opp-4500000000` node at this stage.

Do **not** invent voltage values.

Do **not** modify thermal limits together with the first frequency experiment.

The first modification should be diagnostic or table-extension work only after the real EPSS/LUT mechanism is identified.

## 8. Current control map

```
Pad 3 / erhai CPU policy
        |
        v
qcom-cpufreq-hw
        |
        v
qcom,cpufreq-epss
        |
        v
EPSS freq-domain MMIO
        |
        v
hardware frequency/performance-state LUT
        |
        v
performance-state index
        |
        v
Qualcomm CPU/firmware

       ^             ^
       |             |
    SCMI/DCVS     LMh/thermal
```

## 9. Next source-to-modification step

The next task is to obtain the **Pad 3-specific compiled DTB/DTBO or exact `erhai` device-tree source** and answer:

1. Which CPU nodes are assigned to EPSS domains 0/1/2/3?
2. Which domain corresponds to the 4.32 GHz prime/performance core?
3. Which `reg-names = "freq-domainN"` resource is used by that domain?
4. What EPSS LUT entries are actually exposed?
5. Where do the LUT frequencies originate — DT, firmware, or hardware registers populated by firmware?
6. Whether the maximum 4.32 GHz entry can be replaced/extended without a firmware-side limit.

Only then should we prepare the first experimental patch.

## Safety / recovery

Before flashing any modified kernel/DTB:

- preserve the original `boot`, `vendor_boot`, `dtbo`, and relevant firmware images;
- keep a known-good boot image available;
- test a kernel that only logs/inspects the frequency table before attempting an overclock;
- change one control point at a time;
- do not claim an overclock is functional until the modified image has actually booted and the requested frequency has been observed on hardware.
