# Local APIC

The local APIC is an entry of the [MADT](devse-documentation-en/x86_64/Devices/MADT), its type is 0

The number of local APIC entries in the MADT is equivalent to the number of CPUs, each cpu has its local APIC

the structure of the APIC room entrance is

| offset / size (in bytes) | name |
| ----- | ----- |
| 2/1 | ACPI identifier |
| 3/1 | APIC identifier |
| 4/4 | cpu flag |

