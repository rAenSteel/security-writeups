# Security Write-ups

Write-ups of machines I solved in my own penetration testing lab (a Kali attacker and target VMs on KVM/libvirt).

The goal is not only to record that I got the flag, but **which path I chose and why, what I got wrong and what I learned**. Failed attempts are written down on purpose, because that is where most of the learning happened.

## Lab environment

- **Host:** CachyOS (Arch-based)
- **Virtualization:** KVM / QEMU / libvirt
- **Attacker:** Kali Linux (isolated `virbr` network)
- **Targets:** Kioptrix series, Metasploitable 2 (isolated, no internet access)

## Contents

### Kioptrix series

| Machine | OS | Initial access | Privilege escalation |
| ------- | -- | -------------- | -------------------- |
| [Kioptrix Level 1](kioptrix/kioptrix-1/) | Red Hat (kernel 2.4.x) | mod_ssl "OpenFuck" remote exploit (CVE-2002-0082) | Built into the exploit |
| [Kioptrix Level 2](kioptrix/kioptrix-2/) | CentOS 4.5 | SQL injection login bypass + OS command injection | Kernel `ip_append_data()` exploit, exploit-db 9542 (CVE-2009-2698) |
| [Kioptrix Level 3](kioptrix/kioptrix-3/) | Ubuntu (kernel 2.6.x) | SQL injection in the photo gallery | `sudo` rights on the `ht` editor |

## Note

All testing was done in my own isolated lab, against VMs I own or am permitted to test. The content here is for educational purposes.
