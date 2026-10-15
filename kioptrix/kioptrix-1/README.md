# Kioptrix Level 1

**Difficulty:** Easy
**Target OS:** Red Hat Linux (kernel 2.4.x)
**Result:** Root ✅

Attack chain:

```
Nmap → outdated mod_ssl/OpenSSL found → OpenFuck exploit → apache shell → local privesc → root
```

---

## 1. Enumeration

I ran a full port scan, then a service and version scan on the open ports:
1.
```bash
nmap -T4 -p- <target ip>
```
2.
```bash
nmap -T4 -A -p<open ports> <target ip>
```

Notable ports ([raw output](nmap.md)):

| Port | Service | Version |
| ---- | ------- | ------- |
| 22 | SSH | OpenSSH 2.9p2 (protocol 1.99) |
| 80 | HTTP | Apache 1.3.20 + mod_ssl 2.8.4 + OpenSSL 0.9.6b |
| 111 | rpcbind | RPC #100000 |
| 139 | netbios-ssn | Samba (workgroup MYGROUP) |
| 443 | HTTPS | Apache 1.3.20 + mod_ssl 2.8.4 (SSLv2 supported) |

The system is very old: Apache 1.3.20, OpenSSL 0.9.6b, kernel 2.4.x. These versions date from around 2001–2002, which makes the box a prime target for known public exploits.

Two possible paths stood out:
- **mod_ssl 2.8.4 / OpenSSL 0.9.6b** → a known buffer overflow
- **Samba** → old versions have other vulnerabilities (I tried the first path and it worked)

## 2. Vulnerability research

I searched for Apache/mod_ssl with `searchsploit`:

```bash
searchsploit apache mod_ssl
```

The results included the classic **OpenFuck / OpenFuckV2** exploits:

```
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow      | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow    | unix/remote/764.c / 47080.c
```

This is **CVE-2002-0082**, a heap overflow in mod_ssl. The target's mod_ssl 2.8.4 falls inside the vulnerable range `< 2.8.7`.

## 3. Exploitation (initial access + root)

The OpenFuck exploit gave me remote code execution through mod_ssl as the apache user. After connecting, the exploit also runs a local privilege escalation, so it took me straight to root.

> Note: This exploit fails to compile with a modern GCC; it needs the old `-lcrypto` linking and some header fixes. It also asks for a "magic number" that matches the target's exact Apache version.

## 4. Concepts learned

- How service version banners (Apache 1.3.20, mod_ssl 2.8.4) map directly to CVEs
- Matching version ranges such as `< 2.8.7` with `searchsploit`
- Compilation problems of old C exploits on modern systems
- How the mod_ssl / OpenSSL buffer overflow works (CVE-2002-0082)

## 5. Next time / notes

- I never tried the Samba path on this box. It is worth studying separately as an alternative vector (for example trans2open).
- Instead of running OpenFuck blindly, picking the magic number for the target platform (Red Hat + Apache 1.3.20) in advance cuts down the number of attempts.
