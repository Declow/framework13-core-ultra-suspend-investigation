# Framework Laptop 13 Core Ultra: suspend-power investigation

Investigation of approximately **1.5 W interval-average battery drain** around
`s2idle` on a Framework Laptop 13 (Intel Core Ultra Series 1), BIOS 03.06.

**Status: diagnostic findings, not a validated fix. No RTD3 override was applied.**

## Main findings

1. **The system enters hardware low-power sleep.** Controlled tests recorded
   approximately 99.8% hardware sleep residency without unexpected full wakeups.
   `S0i2.1` accumulates residency; `S0i2.2` remains zero.
2. **ACPI explicitly selects D0 for the SSD's PCIe root port.** A focused trace
   records `acpi_pm_device_sleep_state(maximum=4)` returning `0` for
   `0000:00:1c.0`. The SSD itself is assigned D3hot.
3. **The live storage policy is `STD3=1`, `RP01.RD3C=1`.** In the decoded
   firmware, `RD3C=2` would permit D3cold and activate the generic RTD3 sequence.
4. **The generic sequence has a GPIO mismatch that needs review.** Its live
   reset/power descriptors decode to **GPP_D02 / GPP_D05**, whereas independent
   coreboot Marigold sources identify **GPP_B06** as SSD reset and mark those
   D-group pins not connected. Live pinctrl data corroborates the pin decoding.

**Not established:** that the root port's D0 policy causes the measured drain,
that D3cold would enable `S0i2.2`, or that any override is safe on this board.
This is not evidence of an SSD hardware defect or proof of a confirmed vendor bug.

## Contents

- [Detailed report](REPORT.md): hardware, measurements, firmware logic, GPIO
  analysis, limitations, and questions for firmware maintainers.
- [ACPI probe results](evidence/acpi-probe.txt): selected live values.
- [GPIO evidence](evidence/pinctrl.txt): selected live pinctrl entries.
- [Suspend decision trace](evidence/suspend-decisions.txt): selected events,
  grouped by callback, with kernel pointers and task identifiers omitted.
- [Methodology](METHODOLOGY.md): how the evidence was collected and its limits.

## Hardware

| Component | Tested configuration |
| --- | --- |
| Laptop | Framework Laptop 13 (Intel Core Ultra Series 1) |
| Board | FRANMECP02, revision A2 |
| CPU | Intel Core Ultra 5 125H |
| BIOS | 03.06, dated 2025-08-08 |
| Kernel | 7.2.15-1-omarchy |
| SSD | Micron 2200S, MTFDHBA256TCK-1AS1AABHA, firmware HPS0V23 |
| Sleep mode | `s2idle`; no `deep` option exposed |
| Investigation date | 2026-11-13 |

## Safety and scope

This repository contains documentation and sanitized diagnostic excerpts,
**not an executable fix**. Do not force the mounted system disk's PCIe bridge
into D3 or copy another model's ACPI/GPIO configuration.

The [related Laptop 12 workaround](https://github.com/Ronmi/fw12-s0ix-debug)
targets different hardware. Its successful results do not validate it here.

No BIOS update, persistent ACPI override, boot-parameter change, NVMe feature
change, or RTD3 enablement was performed. Read-only inspection modules were
loaded temporarily and then unloaded; they mark the tested boot as using
unsigned, out-of-tree modules.

Full ACPI tables, complete journals, serial numbers, filesystem identifiers,
hostnames, usernames, network details, and raw kernel pointers are not included.
Investigation and documentation were AI-assisted; interpretations require
independent review.
