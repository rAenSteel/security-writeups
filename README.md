# Security Write-ups

Kendi kurduğum penetrasyon testi lab'ında (KVM/libvirt üzerinde Kali attacker + hedef VM'ler) çözdüğüm makinelerin write-up'ları.

Amaç: sadece "flag'i aldım" değil, **hangi yolu neden seçtim, neyi yanlış yaptım, ne öğrendim** kaydını tutmak. Başarısız denemeler de bilinçli olarak yazıldı — çünkü asıl öğrenme orada.

## Lab Ortamı

- **Host:** CachyOS (Arch tabanlı)
- **Sanallaştırma:** KVM / QEMU / libvirt
- **Attacker:** Kali Linux (izole `virbr` network)
- **Hedefler:** Kioptrix serisi, Metasploitable2 (izole, internete kapalı)

## İçindekiler

### Kioptrix Serisi

| Makine | OS | Ana Vektör | Privesc |
| ------ | -- | ---------- | ------- |
| [Kioptrix Level 1](kioptrix/kioptrix-1/) | Red Hat (2.4.x) | mod_ssl OpenFuck (CVE-2002-0082) | exploit içinde |
| [Kioptrix Level 2](kioptrix/kioptrix-2/) | CentOS 4.5 | SQLi + command injection | kernel sock_sendpage (CVE-2009-2698) |
| [Kioptrix Level 3](kioptrix/kioptrix-3/) | Ubuntu (2.6.x) | galeri SQL injection | sudo + ht editörü |

## Not

Tüm testler kendi izole lab ortamımda, sahibi olduğum/izinli VM'ler üzerinde yapılmıştır. Buradaki içerik eğitim amaçlıdır.
