# Detailed findings

## 1. Scope and evidence levels

The initial goal was to restore lid-close suspend rather than hibernate and
investigate sleep battery consumption. Lid-close suspend was restored; the
underlying power-consumption question remains unresolved.

This report distinguishes:

- **Measured:** battery-counter differences and hardware-residency counters.
- **Observed:** live ACPI data, trace returns, and pinctrl registers.
- **Interpreted:** the relationship between firmware policy and power-state
  choices, and the discrepancy with independently maintained board sources.
- **Unproven:** the cause of all measured drain and the effect/safety of RTD3.

Only one machine, one BIOS version, and the listed SSD configuration were
tested. There is no controlled `RD3C=1` versus `RD3C=2` comparison.

## 2. System and topology

| Item | Value |
| --- | --- |
| System | Framework Laptop 13 (Intel Core Ultra Series 1) |
| Board | FRANMECP02 A2 |
| CPU | Intel Core Ultra 5 125H |
| BIOS | Insyde 03.06, dated 2025-08-08 |
| Kernel | Linux 7.2.15-1-omarchy |
| SSD | Micron 2200S, MTFDHBA256TCK-1AS1AABHA |
| SSD firmware | HPS0V23 |
| SSD PCI address | `0000:a9:00.0` |
| SSD upstream root port | `0000:00:1c.0` |
| Root port ACPI path | `\_SB.PC00.RP01` |
| SSD ACPI path | `\_SB.PC00.RP01.PXSX` |
| Wi-Fi | Intel AX210, `0000:aa:00.0`, under `0000:00:1c.5` / RP06 |
| Graphics | i915; `i915/mtl_dmc.bin` version 2.23 loaded |
| Available sleep mode | `s2idle` only |

## 3. Battery and hardware-residency measurements

| Test | Suspend duration | Hardware sleep | Battery observation |
| --- | --- | --- | --- |
| Short baseline | 618 s | 615.409358 s (99.58%) | Rounded battery percentage remained 60%; not a precise drain measurement |
| Approximately one hour | 3655 s | 3649.059284 s (99.84%) | Nearby energy readings: 36.0839 to 34.5049 Wh over 3744 s, approximately 1.52 W |
| HDMI Gen 3 module removed | 1785 s | 1781.005654 s (99.78%) | 2,203,000 to 2,154,000 uAh over 1819.994 s; approximately 1.50 W using 15.48 V design voltage |

Important limitations:

- Battery measurement windows include brief awake periods and suspend/resume
  transitions. These are **interval averages, not isolated steady-state sleep
  power measurements**.
- The HDMI-removed estimate uses charge loss multiplied by design voltage,
  not a direct energy-meter reading: 49 mAh × 15.48 V = 0.75852 Wh.
- The tests are not a statistically controlled expansion-card comparison.
  Removing the HDMI module made no clear difference in these measurements.
- No experiment isolated the SSD's or root port's individual consumption.
- Controlled tests had no unexpected full-system wakeups.

PMC counters repeatedly accumulated `S0i2.1` residency, with `S0i2.2` remaining
zero. In the focused decision capture, the relevant raw counter values were:

| Counter | Before | After | Difference |
| --- | ---: | ---: | ---: |
| pmc0 S0i2.1 | 6,587,769,755 | 6,671,345,459 | 83,575,704 |
| pmc1 S0i2.1 | 6,585,784,114 | 6,669,361,037 | 83,576,923 |
| pmc0 S0i2.2 | 0 | 0 | 0 |
| pmc1 S0i2.2 | 0 | 0 | 0 |

These are residency-counter readings from `substate_residencies` (microseconds),
not battery measurements. Absence of the deeper substate does not identify its
cause.

## 4. PCI suspend decision

[Selected trace events](evidence/suspend-decisions.txt) show:

| Device | Maximum passed to ACPI selector | ACPI selection | Observed PCI suspend state |
| --- | --- | --- | --- |
| SSD root port `00:1c.0` | 4 | 0 (D0) | D0; PCI PM reported `Skipped` |
| SSD `a9:00.0` | 4 | 3 (D3hot) | D3hot |
| Wi-Fi root port `00:1c.5` | 4 | 4 (D3cold) | D3hot |

The requested state and achieved state must not be conflated. In particular,
the Wi-Fi root port's ACPI selection of 4 does not mean it reached D3cold.

For the SSD root port, both `acpi_pm_device_sleep_state()` and
`acpi_pci_choose_state()` returned 0 inside its noirq suspend callback.
The callback completed with `err=0`.

This establishes that **ACPI policy selects D0**, rather than demonstrating
that the NVMe suspend callback failed.

However:

- PCI D0 does not, by itself, establish that the PCIe link remains active.
- Latched PMC PCIe-port power-gating requirements reported satisfied.
- This selection alone does not prove why `S0i2.2` is absent or explain all drain.

## 5. Storage firmware policy

Firmware tables were copied read-only and decompiled with ACPICA `iasl`
20251212. The relevant behavior is summarized below; these are descriptions,
not an override to install.

1. RP01 initialization copies `STD3` into `RP01.RD3C` for an NVMe/AHCI child.
2. `RP01._S0W` returns 4 only when `RD3C == 2`; otherwise it returns 0.
3. `RP01.GPRS` blocks the generic RTD3 resource path when `RD3C != 2`.
   It also checks platform enablement and VMD conditions.
4. The child's `_S0W` returns 4 for `RD3C == 2`, 0 for `RD3C == 0`,
   and 3 otherwise.
5. The storage `_DSD` advertises `StorageD3Enable` when `STD3` is nonzero.
6. The child shares the root port's power resource through `_PR0` / `_PR3`.

The [live probe](evidence/acpi-probe.txt) confirmed:

```text
STD3 = 1
RP01.RD3C = 1
PGRT = 1
RP01.PRMV = 0
```

This matches the trace: D0 for the root port and D3hot for its storage child.
There is no vendor explanation here establishing why this policy was chosen.

## 6. GPIO mismatch found during enablement checks

Before attempting `RD3C=2`, the generic RTD3 power/reset path was inspected.

The relevant live packages are:

```text
RP01.RSTG = { 0x141082, 0 }  // copied from S1RG / S1RP
RP01.PWRG = { 0x141085, 1 }  // copied from S1PG / S1PP
```

The decoded `PON` / `POFF` path calls `\PIN.ON` / `\PIN.OFF` with these
packages, as well as platform clock and link-sequencing methods. `PIN` passes
the descriptor directly into `\_SB.SGOV`.

The installed firmware decodes the descriptor with:

```text
community array index = (descriptor >> 10) & 7
group index           = (descriptor >> 7) & 7
pad index             = descriptor & 0x7f
```

`PCHS=6` selects the `GPCM` table. For both descriptors, the community array
index is 4 and the group index is 1. In that table this identifies physical
community 5, GPP_D.

The firmware's own read-only `GNUM` method and the kernel's pinctrl data agree:

| Role | Descriptor | Firmware GNUM | Pin name |
| --- | --- | ---: | --- |
| Generic storage reset | `0x141082` | 386 | GPP_D02 |
| Generic storage power | `0x141085` | 389 | GPP_D05 |
| Comparison descriptor for B06 | `0x141006` | 358 | GPP_B06 |

**The comparison descriptor was decoded only; it was never written to a
power/reset package or used to toggle a pin.**

Independent coreboot Marigold sources identify:

- `GPP_B06` as `RTD3_SSD_PLT_RST#`.
- `GPP_D02` and `GPP_D05` as not connected.
- NVMe RTD3 reset as active-low `GPP_B06`, with `skip_on_off_support=true`.

Live [pinctrl entries](evidence/pinctrl.txt) provide a further cross-check:

| Pin | PADCFG_DW0 | Relevant interpretation |
| --- | --- | --- |
| GPP_B06 | `0x84000201` | GPIO output enabled, output value high |
| GPP_D02 | `0x04000300` | GPIO TX-disable and RX-disable both set |
| GPP_D05 | `0x04000300` | GPIO TX-disable and RX-disable both set |

The GPIO TX-disable, RX-disable, and TX-state fields are bits 8, 9, and 0
respectively in the cited Linux Intel pinctrl implementation.

**Interpretation:** the generic sequence is not validated merely because its
descriptors are nonzero. Its mapping conflicts with the independent board
configuration, and current pin registers corroborate the decoding.

**Limit:** independent coreboot configuration is not an authoritative validation
of the installed Insyde BIOS or every board revision. This is a discrepancy
for maintainers to review, not proof of damage, an intentionally disabled
feature, or a safe replacement pin assignment.

Consequently, no `RD3C` override was loaded and no GPIO was changed.

## 7. Other observations

- SSD APST was supported and enabled. Its configured table selected PS3
  after 100 ms, then PS4 from PS3 after 2000 ms. PS4 advertises 0.005 W
  maximum; this is not a measurement of drive power during suspend.
- The NVMe boot log reported the platform quirk selecting simple suspend.
- No PCI suspend callback failed in the captures.
- `spd5118` devices returned `-6` during resume; this did not prevent
  the observed low-power sleep residency.
- Earlier NVMe timeouts/controller reset followed a hibernate resume.
  They do not establish the cause of these suspend-drain measurements.

## 8. Comparison with the Laptop 12 workaround

The related [Framework Laptop 12 report](https://community.frame.work/t/85433)
and [supporting repository](https://github.com/Ronmi/fw12-s0ix-debug) describe
changing a runtime `RD3C` value from 1 to 2.

That report used an i5-1334U / Kingston NV3 configuration and had no S0ix
residency before the change. This Laptop 13 already reaches `S0i2.1` and uses
a different root port and power/reset path. Results cannot be transferred
without validation.

No exact-change adverse-outcome report was identified in the reviewed sources.
The author's warnings are precautionary, not documented damage incidents.
Microsoft's D3cold requirements nevertheless document that a device unable
to tolerate or recover from D3cold can remain unavailable until restart or
another sleep transition. That is a general compatibility risk, not a
demonstrated failure on this SSD.

## 9. Questions for firmware / platform maintainers

1. Is `STD3=1` intentional on this board and BIOS, and what does it protect?
2. Are S1RG/S1PG reference-board defaults, or are D02/D05 expected to be used?
3. Is B06 the correct SSD reset pin for board FRANMECP02 revision A2?
4. Is switching the SSD's main supply supported, or should storage RTD3 use
   a different power/reset sequence on this platform?
5. Is root-port D0 expected while the platform reaches `S0i2.1`?
6. What additional measurement could establish whether this policy limits
   deeper sleep or materially contributes to the observed battery drain?
7. Is there a supported setting or board-specific fix for this configuration?

## 10. References

External source snapshots are pinned where possible:

- [Coreboot Marigold GPIO configuration, B06 and D-group entries](https://github.com/coreboot/coreboot/blob/215d43291e90eb72578dd353bc96ccd52b3f280b/src/mainboard/framework/marigold/gpio.c#L156-L187).
- [Coreboot Marigold NVMe RTD3 configuration](https://github.com/coreboot/coreboot/blob/215d43291e90eb72578dd353bc96ccd52b3f280b/src/mainboard/framework/marigold/devicetree.cb#L305-L328).
- [Linux Meteor Lake pin numbering and communities](https://github.com/torvalds/linux/blob/995832b2cebe6969d1b42635db698803ee31294d/drivers/pinctrl/intel/pinctrl-meteorlake.c).
- [Linux Intel GPIO register fields](https://github.com/torvalds/linux/blob/995832b2cebe6969d1b42635db698803ee31294d/drivers/pinctrl/intel/pinctrl-intel.c#L70-L85).
- [Related Laptop 12 report](https://community.frame.work/t/high-power-drain-when-suspended-due-to-acpi-table-with-workaround/85433).
- [Laptop 12 proof of concept and its warnings](https://github.com/Ronmi/fw12-s0ix-debug).
- [Microsoft: firmware requirements for D3cold](https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/firmware-requirements-for-d3cold).
- [Microsoft: NVMe power-management guidance](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/power-management-for-storage-hardware-devices-nvme).
