+++
draft = false
date = '2026-04-05'
title = 'Konfigurasi Dnscrypt Proxy Dengan Adguard Di Archlinux'
type = 'blog'
description = 'Cara mengkonfigurasi dnscrypt-proxy di Archlinux menggunakan AdGuard DNS untuk enkripsi DNS sekaligus ad blocking.'
image = ''
tags = ['dnscrypt', 'dns', 'adguard', 'archlinux']
+++

## Latar Belakang

DNS query secara default dikirim dalam bentuk plain text -- artinya ISP, admin jaringan, atau siapapun yang berada di jalur koneksi bisa melihat domain apa saja yang kita akses. Selain masalah privasi, beberapa ISP juga melakukan DNS hijacking untuk mengarahkan traffic ke halaman iklan atau memblokir akses ke situs tertentu.

**dnscrypt-proxy** adalah tool yang mengenkripsi DNS query menggunakan protokol DNSCrypt atau DNS-over-HTTPS (DoH), sehingga query DNS tidak bisa disadap atau dimanipulasi. Dikombinasikan dengan **AdGuard DNS** sebagai upstream resolver, kita tidak hanya mendapat enkripsi tapi juga ad blocking dan tracker protection langsung di level DNS -- tanpa perlu extension browser tambahan.

## Permasalahan

Beberapa masalah yang dihadapi tanpa DNS terenkripsi:

- **ISP bisa melihat semua domain yang diakses** -- meskipun koneksi ke website sudah HTTPS, DNS query tetap plain text
- **DNS hijacking oleh ISP** -- beberapa ISP mengarahkan DNS query ke server mereka sendiri, bahkan kalau kita sudah set DNS ke Google atau Cloudflare
- **Iklan dan tracker di mana-mana** -- solusi seperti uBlock Origin hanya bekerja di browser, tidak mencakup aplikasi lain di sistem
- **DNS resolver bawaan tidak terenkripsi** -- default resolver di `/etc/resolv.conf` mengirim query tanpa enkripsi apapun

Yang dibutuhkan adalah satu solusi yang menangani enkripsi DNS sekaligus filtering -- dan bekerja untuk semua aplikasi di sistem, bukan hanya browser.

## Pendekatan Solusi

Ada beberapa kombinasi tool dan DNS provider yang bisa dipakai:

| Pendekatan | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **dnscrypt-proxy + AdGuard DNS** | Enkripsi + ad blocking built-in, satu tool | Bergantung pada server AdGuard |
| **dnscrypt-proxy + Cloudflare** | Enkripsi, server cepat | Tidak ada ad blocking bawaan |
| **systemd-resolved + DoT** | Bawaan systemd, minimal setup | Tidak ada ad blocking, opsi server terbatas |
| **Pi-hole + upstream DoH** | Full kontrol, custom blocklist | Butuh dedicated server/device |

Saya memilih **dnscrypt-proxy dengan AdGuard DNS** karena:

1. **Satu tool, dua fungsi** -- enkripsi dan ad blocking tanpa perlu setup tambahan
2. **Bekerja system-wide** -- semua aplikasi (browser, terminal, desktop app) melewati DNS yang sama
3. **AdGuard DNS punya server yang cepat** -- tersedia di banyak region termasuk Asia
4. **Konfigurasi minimal** -- cukup edit satu file TOML

## Implementasi Teknis

### Instalasi

dnscrypt-proxy tersedia di repository resmi Archlinux:

```
$ sudo pacman -S dnscrypt-proxy
```

### Konfigurasi

File konfigurasi utama ada di `/etc/dnscrypt-proxy/dnscrypt-proxy.toml`. Sebelum mengedit, backup dulu file aslinya:

```
$ sudo cp /etc/dnscrypt-proxy/dnscrypt-proxy.toml /etc/dnscrypt-proxy/dnscrypt-proxy.toml.bak
```

Edit file konfigurasi:

```
$ sudo vim /etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

Berikut parameter-parameter penting yang perlu dikonfigurasi:

#### Server Names

Set DNS server ke AdGuard DNS:

```toml
server_names = ['adguard-dns', 'adguard-dns-doh']
```

`adguard-dns` menggunakan protokol DNSCrypt, sedangkan `adguard-dns-doh` menggunakan DNS-over-HTTPS. Keduanya sudah include ad blocking dan tracker protection secara default.

Jika ingin menggunakan AdGuard DNS yang non-filtering (tanpa ad blocking), gunakan:

```toml
server_names = ['adguard-dns-unfiltered']
```

#### Listen Address

Pastikan dnscrypt-proxy listen di localhost:

```toml
listen_addresses = ['127.0.0.1:53']
```

#### Opsi Keamanan

Aktifkan beberapa opsi keamanan:

```toml
# Hanya gunakan server yang mendukung DNSSEC
require_dnssec = true

# Hanya gunakan server yang tidak melakukan logging
require_nolog = true

# Hanya gunakan server yang tidak melakukan filtering (override jika pakai AdGuard)
require_nofilter = false
```

`require_nofilter` di-set `false` karena kita memang ingin filtering dari AdGuard DNS. Kalau di-set `true`, server AdGuard yang melakukan filtering akan diexclude.

#### IPv6

Jika tidak menggunakan IPv6, bisa disable untuk menghindari query yang tidak perlu:

```toml
ipv6_servers = false
block_ipv6 = true
```

#### Sources

Pastikan bagian `[sources]` mengacu ke public resolver list agar dnscrypt-proxy tahu daftar server yang tersedia:

```toml
[sources]
  [sources.'public-resolvers']
  urls = ['https://raw.githubusercontent.com/DNSCrypt/dnscrypt-resolvers/master/v3/public-resolvers.md', 'https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md']
  cache_file = 'public-resolvers.md'
  minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
```

### Disable systemd-resolved

Jika `systemd-resolved` aktif, perlu dinonaktifkan karena akan berkonflik dengan dnscrypt-proxy di port 53:

```
$ sudo systemctl disable --now systemd-resolved
```

### Konfigurasi DNS di NetworkManager

Agar semua DNS query melewati dnscrypt-proxy, set DNS server ke `127.0.0.1` lewat NetworkManager. Buka pengaturan koneksi yang digunakan, masuk ke bagian IPv4 Settings, lalu ubah DNS server ke `127.0.0.1`.

Dengan cara ini, NetworkManager yang mengurus `/etc/resolv.conf` secara otomatis -- tidak perlu edit manual atau proteksi file dengan `chattr`.

### Aktifkan Service

```
$ sudo systemctl enable --now dnscrypt-proxy
```

Verifikasi service berjalan:

```
$ sudo systemctl status dnscrypt-proxy
```

### Verifikasi

Test apakah DNS resolution bekerja:

```
$ dig mnabila.com
```

Pastikan di bagian `SERVER` menunjukkan `127.0.0.1#53`:

```
;; SERVER: 127.0.0.1#53(127.0.0.1)
```

Cek apakah dnscrypt-proxy menggunakan server AdGuard:

```
$ dnscrypt-proxy -resolve mnabila.com
```

Output akan menampilkan informasi resolver yang digunakan, termasuk nama server dan protokol (DNSCrypt atau DoH).

Test ad blocking -- coba resolve domain iklan:

```
$ dig ads.google.com
```

Jika AdGuard DNS aktif dan berfungsi, domain iklan akan di-resolve ke `0.0.0.0` atau tidak mengembalikan IP sama sekali.

## Tantangan yang Dihadapi

Tantangan pertama adalah **port 53 yang sudah dipakai**. Jika `systemd-resolved` masih aktif, dnscrypt-proxy tidak bisa bind ke port 53 dan service akan gagal start. Pastikan `systemd-resolved` sudah dinonaktifkan sebelum menjalankan dnscrypt-proxy.

Tantangan kedua adalah memastikan **DNS di NetworkManager sudah mengarah ke `127.0.0.1`** untuk setiap koneksi yang digunakan. Kalau punya beberapa profil koneksi (WiFi rumah, WiFi kantor, tethering), masing-masing perlu di-set DNS-nya. Jika tidak, koneksi tersebut akan kembali menggunakan DNS dari DHCP (biasanya DNS ISP).

## Insight dan Pembelajaran

Beberapa insight setelah menggunakan dnscrypt-proxy dengan AdGuard DNS:

- **Ad blocking di level DNS itu efektif untuk system-wide** -- tidak hanya browser, tapi juga aplikasi desktop, terminal app, dan bahkan IoT device kalau kita setup sebagai DNS server untuk jaringan lokal.
- **`require_nofilter = false` adalah setting yang mudah terlewat** -- secara default nilainya `true`, yang berarti server dengan filtering (termasuk AdGuard) akan diexclude. Kalau lupa set ke `false`, dnscrypt-proxy tidak akan menggunakan AdGuard DNS meskipun sudah di-set di `server_names`.
- **DNSSEC memberikan layer keamanan tambahan** -- memvalidasi bahwa response DNS benar-benar dari server yang dituju dan belum dimanipulasi di perjalanan. Worth it untuk diaktifkan.
- **Performa tidak terasa berbeda** -- kekhawatiran awal bahwa enkripsi DNS akan memperlambat browsing ternyata tidak terbukti. Latency tambahan dari enkripsi hampir tidak terasa di penggunaan sehari-hari.

## Penutup

dnscrypt-proxy dengan AdGuard DNS adalah kombinasi yang solid untuk mengamankan DNS di Archlinux. Satu tool menangani dua kebutuhan sekaligus: enkripsi DNS query agar tidak bisa disadap ISP, dan ad blocking di level DNS yang bekerja untuk seluruh sistem. Konfigurasinya cukup straightforward -- edit satu file TOML, set resolv.conf, dan aktifkan service. Untuk yang peduli dengan privasi dan ingin mengurangi iklan tanpa banyak setup, ini solusi yang minimal effort tapi high impact.

## Referensi

- [Arch Wiki - Dnscrypt-proxy](https://wiki.mnabila.com/title/Dnscrypt-proxy) -- Diakses pada 2026-04-05
- [dnscrypt-proxy GitHub](https://github.com/DNSCrypt/dnscrypt-proxy) -- Diakses pada 2026-04-05
- [dnscrypt-proxy Wiki - Configuration](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Configuration) -- Diakses pada 2026-04-05
- [AdGuard DNS](https://adguard-dns.io/) -- Diakses pada 2026-04-05
