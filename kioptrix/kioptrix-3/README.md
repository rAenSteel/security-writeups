# Kioptrix Level 3 (1.2)

**Difficulty:** Medium
**Target OS:** Ubuntu (kernel 2.6.x), "Ligoat Security" website
**Result:** Root ✅

Attack chain:

```
Nmap → gallery SQL injection → credentials → SSH → sudo ht → edit sudoers → root
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
| 22 | SSH | OpenSSH 4.7p1 |
| 80 | HTTP | Apache 2.2.8 + PHP 5.2.4 + Suhosin-Patch |

Site title: "Ligoat Security - Got Goat? Security". The site ran on LotusCMS.

I had to add a `kioptrix3.com` entry to `/etc/hosts`, because the site loads its resources by domain name.

Directory scan (ffuf) results:

```
modules       [301]
gallery       [301]   <-- the SQL injection was here
data          [403]
core          [301]
phpmyadmin    [301]
```

## 2. What I got wrong / vectors that failed

I tried many wrong paths on this box before finding the right one. Writing them down saves time next time:

| Vector tried | Why it failed |
| ------------ | ------------- |
| phpMyAdmin `setup.php` config exploit (Metasploit `phpmyadmin_config`, CVE-2009-1151) | phpMyAdmin on the target was installed manually; the wizard-based `/config/` and `scripts/setup.php` were not there |
| Entering a full URL in the Metasploit module's `URI` option | With a full URL instead of a path the module sent the wrong request; the option takes only the path |
| Writing files through phpMyAdmin with the FILE privilege | The MySQL user only had USAGE, not FILE |
| MySQL yaSSL exploit | Wrong port / SSL was not set up |
| PHP-CGI exploit (CVE-2012-1823) | There was no executable binary under `/cgi-bin/` |
| LotusCMS RCE (I tried 63 payloads) | Suhosin-Patch blocked system commands |

**General lesson:** I should have treated phpMyAdmin as a place to log in with stolen credentials, not as an exploit target. When a Metasploit module did not work, asking "what path does this box intend?" would have been more productive than continuing to try things blindly.

## 3. Working vector: gallery SQL injection

A parameter on the `gallery.php` page (`id` / sort) was vulnerable to SQL injection:

```
http://kioptrix3.com/gallery/gallery.php?id=1
```

I exploited it automatically with sqlmap:

- Found the `dev_accounts` table in the `gallery` database
- Dumped two users' credentials and cracked their MD5 hashes:

```
dreg       → Mast3r
loneferret → starwars
```

I also verified the database manually through phpMyAdmin (MySQL 5.0.51a).

## 4. SSH access

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa loneferret@<target-ip>
# Password: starwars
```

> Note: The SSH server is old, so a modern OpenSSH client refuses to connect; I had to re-enable the old algorithm with `-oHostKeyAlgorithms=+ssh-rsa`.

## 5. Privilege escalation: sudo + the ht editor

First thing after logging in:

```bash
sudo -l
```

Output:

```
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
```

This meant:
- `loneferret` **cannot** run `su` (`!` = forbidden)
- `loneferret` can run the `ht` editor as root, **without a password**

`ht` is a terminal-based text/hex editor. Running as root, it can edit any file on the system, which makes it as dangerous as a root shell.

Attack: I opened `/etc/sudoers` with `ht` and removed the `!` in the `!/usr/bin/su` line, which made `su` usable.

```bash
export TERM=xterm       # ht does not recognize xterm-256color, so this is required
sudo ht /etc/sudoers
```

Inside the editor: move with the arrow keys, edit, save with **F2**, quit with **F10**.

Then:

```bash
sudo su
# ROOT ✅
```

## 6. Tools learned

- **sqlmap**: SQL injection automation
- **ht**: terminal-based text/hex editor
- **searchsploit**: exploit database search

## 7. Concepts learned

- The MySQL privilege system (the difference between USAGE, FILE and GRANT)
- How Suhosin-Patch blocks PHP exploits
- Finding a privilege escalation vector with `sudo -l` (a writable binary run as root = root)
- Compatibility problems with old SSH algorithms
- Deciding whether a tool is an "exploit target" or an "access target"

## 8. Next time / notes

- I tried far too many failed exploits on this box. With the most obvious web entry point (the gallery parameter) right there, reaching for Metasploit modules first was a waste of time; manual web enumeration should come first.
- Editing `/etc/sudoers` is risky: a broken file makes the system reject every sudo command. On a real system `visudo` checks the syntax; in a CTF you have to roll back to a snapshot if it breaks.
- Alternative privesc: use the same root-write primitive to edit `/etc/passwd`, or go after the kernel with DirtyCow. The `ht` path is the intended solution for this box.
