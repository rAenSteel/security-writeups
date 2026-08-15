# Kioptrix Level 3 — Nmap Ham Çıktı

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
| ssh-hostkey:
|   1024 30:e3:f6:dc:2e:22:5d:17:ac:46:02:39:ad:71:cb:49 (DSA)
|_  2048 9a:82:e6:96:e4:7e:d6:a6:d7:45:44:cb:19:aa:ec:dd (RSA)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
|_http-title: Ligoat Security - Got Goat? Security ...

Running: Linux 2.6.X
OS details: Linux 2.6.9 - 2.6.33
```

## searchsploit Sonuçları (referans)

OpenSSH 4.7 ve Apache 2.2 için baktığım exploitler — hiçbiri işe yaramadı,
asıl yol web SQL injection'dı:

```
searchsploit openssh 4.7    -> sadece username enumeration, işe yaramaz
searchsploit apache 2.2     -> çoğu DoS veya alakasız versiyonlar
```

## phpMyAdmin Manuel Doğrulama

```
version()       5.0.51a-3ubuntu5.4
current_user()  @localhost
@@datadir       /var/lib/mysql/
```

## Dizin Taraması (ffuf)

```
modules       [Status: 301]
gallery       [Status: 301]   <-- SQLi
data          [Status: 403]
core          [Status: 301]
phpmyadmin    [Status: 301]
server-status [Status: 403]
```
