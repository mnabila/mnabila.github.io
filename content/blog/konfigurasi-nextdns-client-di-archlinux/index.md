+++
draft = false
date = '2020-11-08'
lastmod = '2026-03-15'
title = 'Konfigurasi Nextdns Client Di Archlinux'
type = 'blog'
description = 'Cara mengimplementasikan NextDNS sebagai DNS firewall untuk keamanan berselancar di internet pada Archlinux.'
image = ''
tags = ['nextdns', 'dns', 'blockads']
+++

## Latar Belakang

Sudah beberapa lama saya menggunakan **dnscrypt** sebagai DNS resolver di Archlinux. Fungsinya sederhana -- mengenkripsi DNS query agar tidak bisa disadap oleh ISP atau pihak ketiga. Namun seiring waktu, kebutuhan saya berkembang. Selain enkripsi, saya juga ingin fitur seperti ad blocking, tracker protection, dan monitoring query -- sesuatu yang dnscrypt tidak sediakan secara built-in.

Saat itulah saya menemukan **NextDNS** -- layanan DNS firewall yang menawarkan paket lengkap: DNS-over-HTTPS/TLS, blokir iklan, pencegahan pelacakan, dan dashboard monitoring. Kebetulan mereka juga menyediakan client resmi yang ringan.

## Permasalahan

Dnscrypt bekerja dengan baik untuk kebutuhan enkripsi DNS, tapi konfigurasinya cukup rumit jika ingin menambahkan fitur filtering. Butuh setup tambahan seperti mengintegrasikan dengan Pi-hole atau membuat blocklist sendiri. Saya mencari solusi yang lebih turnkey -- satu tool yang langsung menyediakan enkripsi sekaligus filtering tanpa perlu banyak konfigurasi tambahan.

## Pendekatan Solusi

Ada beberapa cara mengintegrasikan NextDNS ke sistem Archlinux:

- **nextdns-client** -- client resmi dari NextDNS
- **systemd-resolved** -- resolver DNS bawaan systemd
- **dnscrypt-proxy** -- proxy DNS terenkripsi

Saya memilih **nextdns-client** karena paling straightforward -- satu binary, satu service, minimal konfigurasi. Tidak perlu mengubah setup systemd-resolved yang bisa berdampak ke service lain.

## Implementasi Teknis

### Instalasi

NextDNS client tersedia di AUR (Arch User Repository):

```
$ yay -S nextdns
```

![instalasi](img/instalasi.png)

### Mengaktifkan Service

Setelah terinstall, aktifkan service-nya:

```
$ sudo systemctl start nextdns
```

Verifikasi service sudah berjalan:

```
$ sudo systemctl status nextdns
```

![status nextdns](img/nextdns-status.png)

### Konfigurasi NextDNS ID

Setiap akun NextDNS memiliki ID unik yang menentukan konfigurasi filtering mana yang akan digunakan:

1. Buka [my.nextdns.io](https://my.nextdns.io) dan salin NextDNS ID

   ![my.nextdns.io dashboard](img/mynextdns.png)

2. Jalankan perintah instalasi dengan ID tersebut:

   ```
   $ sudo nextdns install -config <nextdns-id> -setup-router
   ```

### Konfigurasi DNS Resolver

Agar semua DNS query melewati NextDNS, set DNS resolver ke `127.0.0.1`. Ada dua cara:

**Via NetworkManager:**

Masukkan `127.0.0.1` di bagian DNS server dan search domain pada network manager.

![network manager](img/networkmanager.png)

**Via resolv.conf:**

Langsung ubah file `/etc/resolv.conf` dan set alamat DNS-nya ke `127.0.0.1`.

### Verifikasi

Coba ping ke salah satu domain untuk memastikan konfigurasi sudah benar:

![test konfigurasi](img/test.png)

## Tantangan yang Dihadapi

Tantangan pertama adalah memastikan NextDNS client tidak berkonflik dengan DNS resolver yang sudah ada. Di Archlinux, `systemd-resolved` bisa aktif secara default dan menguasai `/etc/resolv.conf`. Perlu dipastikan bahwa resolv.conf mengarah ke `127.0.0.1` (NextDNS client) dan bukan ke `127.0.0.53` (systemd-resolved).

Selain itu, penggunaan flag `-setup-router` pada perintah install perlu dipahami implikasinya -- flag ini mengkonfigurasi NextDNS client untuk melayani DNS query dari seluruh perangkat di jaringan lokal, bukan hanya mesin lokal saja. Untuk kebutuhan personal, sebenarnya tidak wajib digunakan.

## Insight dan Pembelajaran

Migrasi dari dnscrypt ke NextDNS memberikan beberapa insight:

- **Kelebihan:**
  - Instalasi sangat mudah -- tidak perlu konfigurasi manual yang rumit
  - Resource yang dibutuhkan sangat ringan
  - Dashboard web untuk monitoring dan konfigurasi filtering sangat membantu
  - Bisa digunakan di berbagai perangkat dengan satu akun

- **Kekurangan:**
  - Ada limitasi 300 ribu query per bulan untuk paket gratis -- penggunaan normal di satu mesin biasanya cukup, tapi bisa habis cepat jika dipakai sebagai router DNS untuk banyak perangkat
  - Bergantung pada layanan pihak ketiga -- jika NextDNS down, DNS resolution terganggu

Trade-off utamanya ada pada kontrol vs kenyamanan. Dnscrypt memberikan kontrol penuh tapi butuh effort lebih untuk setup filtering. NextDNS memberikan kenyamanan turnkey tapi dengan ketergantungan pada layanan cloud.

## Penutup

NextDNS berhasil menggantikan dnscrypt di setup Archlinux saya dengan pengalaman yang lebih baik secara keseluruhan. Proses migrasi berjalan lancar tanpa downtime yang berarti. Untuk kebutuhan personal, paket gratis dengan 300 ribu query per bulan sudah lebih dari cukup. Ke depannya, jika ada kebutuhan untuk self-hosted DNS filtering, Pi-hole atau AdGuard Home bisa menjadi alternatif yang tidak bergantung pada layanan cloud.

## Referensi

- [NextDNS Wiki](https://github.com/nextdns/nextdns/wiki) -- Diakses pada 2020-11-08
