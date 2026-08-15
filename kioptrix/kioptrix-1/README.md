# Kioptrix Level 1

**Zorluk:** Kolay
**Hedef İşletim Sistemi:** Red Hat Linux (kernel 2.4.x)
**Sonuç:** Root alındı ✅

Saldırı zinciri:

```
Nmap → eski mod_ssl/OpenSSL tespiti → OpenFuck exploiti → apache shell → local privesc → root
```

---

## 1. Keşif (Enumeration)

Nmap ile tam port + servis versiyon taraması yaptım:
1.
```bash
nmap -T4 -p- <hedef ip>
```
2
```bash
nmap -T4 -A -p<acik portlar> <hedef ip>
```

Öne çıkan portlar:

| Port | Servis | Versiyon |
| ---- | ------ | -------- |
| 22 | SSH | OpenSSH 2.9p2 (protocol 1.99) |
| 80 | HTTP | Apache 1.3.20 + mod_ssl 2.8.4 + OpenSSL 0.9.6b |
| 111 | rpcbind | RPC #100000 |
| 139 | netbios-ssn | Samba (workgroup MYGROUP) |
| 443 | HTTPS | Apache 1.3.20 + mod_ssl 2.8.4 (SSLv2 destekli) |

Sistem son derece eski: Apache 1.3.20, OpenSSL 0.9.6b, kernel 2.4.x. Bu versiyonlar 2001-2002 civarına ait, yani bilinen public exploitler için birinci sınıf hedef.

İki potansiyel yol göze çarptı:
- **mod_ssl 2.8.4 / OpenSSL 0.9.6b** → bilinen bir buffer overflow zafiyeti
- **Samba** → eski sürümlerde başka zafiyetler olabilir (ilk yolu denedim, çalıştı)

## 2. Zafiyet Araştırması

`searchsploit` ile Apache/mod_ssl aradım:

```bash
searchsploit apache mod_ssl
```

Sonuçta klasik **OpenFuck / OpenFuckV2** exploitleri çıktı:

```
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow      | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow    | unix/remote/764.c / 47080.c
```

Bu, **CVE-2002-0082** — mod_ssl'deki bir heap overflow. Hedefteki mod_ssl 2.8.4, açık aralık olan `< 2.8.7`'nin içinde, yani uygun.

## 3. Exploit (Initial Access + Root)

OpenFuck exploitini kullanarak mod_ssl üzerinden apache kullanıcısı olarak uzaktan kod çalıştırma sağladım. Exploit'in kendisi bağlantı kurduktan sonra local privilege escalation da içeriyordu, böylece doğrudan root'a ulaştım.

> Not: Bu exploit modern GCC ile derlenirken hata verir; eski `-lcrypto` bağlama ve bazı header düzeltmeleri gerekir. Ayrıca exploit hedefin tam Apache sürümüne karşılık gelen bir "magic number" seçimi ister.

## 4. Öğrenilen Kavramlar

- Servis versiyon banner'larının (Apache 1.3.20, mod_ssl 2.8.4) doğrudan CVE eşlemesine nasıl gittiği
- `searchsploit` ile versiyon aralığı eşleştirme (`< 2.8.7` gibi)
- Eski C exploitlerinin modern sistemde derlenme sorunları
- mod_ssl / OpenSSL buffer overflow mantığı (CVE-2002-0082)

## 5. Bir Dahaki Sefere / Notlar

- Bu box'ta Samba yolunu hiç denemedim — alternatif bir vektör olarak (trans2open gibi) ayrıca çalışılabilir.
- OpenFuck exploitini "kör" çalıştırmak yerine, hedef mimarisine (Red Hat + Apache 1.3.20) uygun magic number'ı önceden belirlemek deneme sayısını azaltıyor.
