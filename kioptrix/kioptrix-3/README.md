# Kioptrix Level 3 (1.2)

**Zorluk:** Orta
**Hedef İşletim Sistemi:** Ubuntu (kernel 2.6.x), "Ligoat Security" web sitesi
**Sonuç:** Root alındı ✅

Saldırı zinciri:

```
Nmap → galeri SQL injection → credentials → SSH → sudo ht → sudoers düzenleme → root
```

---

## 1. Keşif (Enumeration)

```bash
nmap -sV -O <hedef-ip>
```

Açık portlar:

| Port | Servis | Versiyon |
| ---- | ------ | -------- |
| 22 | SSH | OpenSSH 4.7p1 |
| 80 | HTTP | Apache 2.2.8 + PHP 5.2.4 + Suhosin-Patch |

Web sitesi başlığı: "Ligoat Security - Got Goat? Security". Site LotusCMS ile çalışıyordu.

`/etc/hosts` dosyasına `kioptrix3.com` girdisini eklemem gerekti (site domain adıyla çağırıyordu).

Dizin taraması (ffuf) ile bulunanlar:

```
modules       [301]
gallery       [301]   <-- SQL injection buradaydı
data          [403]
core          [301]
phpmyadmin    [301]
```

## 2. Yanlış Yaptığım / Başarısız Olan Vektörler

Bu box'ta doğru yolu bulmadan önce birçok yanlış yol denedim. Bunları not almak, bir dahaki sefere zaman kazandırıyor:

| Denenen Vektör | Neden Başarısız Oldu |
| -------------- | -------------------- |
| phpMyAdmin `setup.php` config exploiti (Metasploit `phpmyadmin_config`, CVE-2009-1151) | Hedefteki phpMyAdmin manuel kurulmuştu, sihirbaz tabanlı `/config/` ve `scripts/setup.php` altyapısı yoktu |
| Metasploit modülünde `URI` alanına tam URL girmek | Path yerine tam URL girince modül yanlış istek atıyordu — sadece path girilmeliydi |
| phpMyAdmin FILE yetkisiyle dosya yazma | MySQL kullanıcısı sadece USAGE yetkisine sahipti, FILE yoktu |
| MySQL yaSSL exploiti | Yanlış port / SSL kurulu değildi |
| PHP CGI exploiti (CVE-2012-1823) | `/cgi-bin/` altında çalıştırılabilir binary yoktu |
| LotusCMS RCE (63 payload denedim) | Suhosin-Patch sistem komutlarını engelledi |

**Genel ders:** phpMyAdmin'i bir *exploit hedefi* olarak görmek yerine, credential elde edip *giriş yapılacak bir hedef* olarak görmek gerekiyordu. Metasploit modülü çalışmadığında körlemesine denemeye devam etmek yerine, "bu box'ın niyet ettiği yol ne?" diye sormak daha verimliydi.

## 3. Başarılı Vektör: Galeri SQL Injection

`gallery.php` sayfasındaki parametre (`id` / sort) SQL injection'a açıktı:

```
http://kioptrix3.com/gallery/gallery.php?id=1
```

SQLmap ile otomatik olarak sömürdüm:

- `gallery` veritabanındaki `dev_accounts` tablosu bulundu
- İki kullanıcının credentials'ı çekildi, MD5 hash'leri kırıldı:

```
dreg       → Mast3r
loneferret → starwars
```

phpMyAdmin üzerinden de veritabanını manuel doğruladım (MySQL 5.0.51a).

## 4. SSH Erişimi

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa loneferret@<hedef-ip>
# Şifre: starwars
```

> Not: Eski SSH sunucusu olduğu için modern OpenSSH client'ı bağlanmayı reddediyor; `-oHostKeyAlgorithms=+ssh-rsa` ile eski algoritmayı elle etkinleştirmek gerekti.

## 5. Privilege Escalation — sudo + ht editörü

Giriş yaptıktan sonra ilk iş:

```bash
sudo -l
```

Çıktı:

```
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
```

Bu şunu söylüyordu:
- `loneferret`, `su` komutunu **çalıştıramaz** (`!` = yasak)
- `loneferret`, `ht` editörünü root yetkisiyle ve **şifresiz** çalıştırabilir

`ht`, terminal tabanlı bir text/hex editörü. Root yetkisiyle çalıştığı için sistemdeki herhangi bir dosyayı düzenleyebilir hale geliyor — yani doğrudan root shell kadar tehlikeli.

Saldırı: `/etc/sudoers` dosyasını `ht` ile açıp `!/usr/bin/su` satırındaki `!` işaretini kaldırdım, böylece `su` kullanılabilir hale geldi.

```bash
export TERM=xterm       # ht xterm-256color'ı tanımıyor, bu şart
sudo ht /etc/sudoers
```

Editör içinde: ok tuşlarıyla gez, düzenle, **F2** kaydet, **F10** çık.

Sonra:

```bash
sudo su
# ROOT ✅
```

## 6. Öğrenilen Araçlar

- **sqlmap** — SQL injection otomasyonu
- **ht** — terminal tabanlı text/hex editörü
- **searchsploit** — exploit veritabanı arama

## 7. Öğrenilen Kavramlar

- MySQL yetki sistemi (USAGE, FILE, GRANT arasındaki fark)
- Suhosin-Patch'in PHP exploitlerini nasıl engellediği
- `sudo -l` ile privilege escalation vektörü tespiti (root yetkili yazılabilir bir binary = root)
- Eski SSH algoritma uyumluluğu sorunu
- Bir aracı "exploit hedefi" mi yoksa "erişim hedefi" mi olarak görmek gerektiği ayrımı

## 8. Bir Dahaki Sefere / Notlar

- Bu box'ta çok fazla başarısız exploit denedim. Web tarafında en bariz giriş noktası (galeri parametresi) dururken önce Metasploit modüllerine sarılmak zaman kaybıydı — önce manuel web enumeration yapılmalıydı.
- `/etc/sudoers` düzenlemesi risklidir: yanlış düzenlenirse sistem tüm sudo komutlarını reddeder. Gerçek sistemde `visudo` sözdizimi kontrolü yapar; CTF'te bozulursa snapshot'a dönmek gerekir.
- Alternatif privesc: aynı root-write primitive ile `/etc/passwd` düzenlemek ya da kernel tarafında DirtyCow — ama `ht` yolu bu box'ın niyet edilen çözümü.
