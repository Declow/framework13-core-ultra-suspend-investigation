# Evidence collection and publication scope

## Collection

All observations were collected on the configuration listed in the report.

1. Restored lid-close suspend and verified logind's active actions.
2. Read battery energy/charge counters before and after user-initiated sleep.
3. Compared suspend duration with `last_hw_sleep` and Intel PMC residency
   counters; inspected logs for unexpected wakeups.
4. Removed the HDMI Gen 3 expansion module for a further battery-only test.
5. Collected temporary PCI PM debug messages and suspend/resume callback events.
6. Used temporary function-entry/return probes to observe ACPI state selection.
   Entry and return events were associated within the same callback/task;
   unrelated interleaved activity was not treated as part of that call.
7. Copied ACPI tables read-only and decompiled them with ACPICA `iasl`.
8. Used a small read-only kernel module to evaluate selected existing integer
   objects and GPIO packages via `acpi_evaluate_integer` /
   `acpi_evaluate_object`.
9. In a second read-only inspection, evaluated `\_SB.GNUM` with the observed
   descriptors and a comparison descriptor for B06. Inspection of GNUM showed
   that it decodes descriptor-table data without writing GPIOs.
10. Compared the resulting GPIO numbers against selected live pinctrl entries
    and independently maintained platform sources.

The read-only module was restricted to the exact model and BIOS and built
against the running kernel's matching headers. It did not load ACPI tables,
call power methods, write PCI configuration, or toggle GPIOs. It was
unloaded immediately after each inspection.

Loading unsigned out-of-tree code marks the kernel as tainted, even when the
module only inspects data. This marker remains until reboot; unloading alone
does not clear it.

Temporary trace instances, function probes, diagnostic latch settings, and
dynamic-debug settings were cleaned up. The focused capture's cleanup
reported no errors.

## What was not done

- No `RD3C=2` test or other storage-policy override.
- No direct GPIO change or forced PCI power-state change.
- No BIOS flash or firmware-setting modification.
- No NVMe feature, ASPM, or boot-parameter change.
- No automatic module loading or persistent experimental installation.
- No claim that measured battery draw is entirely caused by the SSD/root port.
- No independent multi-machine or alternate-SSD reproduction.

A proposed ACPI helper was compiled locally for inspection, but never
loaded. It and all executable artifacts are deliberately excluded from this
findings repository.

## Evidence formatting

- `acpi-probe.txt` preserves the selected ACPI probe output without host details.
- `pinctrl.txt` contains only the three relevant pin entries from each view.
- `suspend-decisions.txt` contains selected actual trace event text, grouped
  by the device's noirq suspend callback. Kernel pointer arguments are replaced
  with `<pointer omitted>`; task/CPU columns and unrelated events are omitted.
  Trace-clock timestamps and function offsets are retained.
- The report's battery and PMC tables transcribe relevant numeric readings.

Full ACPI tables and journals remain outside this repository. This intentionally
limits independent review of all firmware branches; the report's ACPI-flow
descriptions are interpretations of the local decompilation, not a substitute
for maintainer review of the complete firmware.

No serial numbers, filesystem identifiers, usernames, hostnames, network
configuration, or authentication data are included.

## Original decompilation locations

These line ranges identify the original local decompilation for a future
maintainer review. They are not links to files included in this repository;
table numbering and line positions can differ with another dump or disassembler.

| Local table | Lines | Content |
| --- | --- | --- |
| DSDT | 33459–33524 | RP01 initialization and STD3 copy |
| SSDT19 | 694–733 | Reset/power packages and root-port `_S0W` |
| SSDT19 | 865–903 | RTD3 resource gating |
| SSDT19 | 923–1025 | PON / POFF sequencing |
| SSDT19 | 1053–1101 | Storage-child power methods |
| DSDT | 13306–13524 | GPCM GPIO descriptor table |
| DSDT | 13654–13725 | GPIO descriptor selection and decoding |
| DSDT | 13865–13876 | SGOV GPIO output write |
| DSDT | 86018–86043 | PIN helpers |
