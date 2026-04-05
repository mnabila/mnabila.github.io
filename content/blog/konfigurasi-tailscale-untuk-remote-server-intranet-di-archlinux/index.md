+++
draft = false
date = '2026-04-05'
title = 'Konfigurasi Tailscale Untuk Remote Server Intranet Di Archlinux'
type = 'blog'
description = 'Cara menggunakan Tailscale di Archlinux untuk mengakses server yang berada di jaringan intranet dari mana saja tanpa perlu VPN tradisional atau port forwarding.'
image = ''
tags = ['tailscale', 'vpn', 'wireguard', 'networking', 'archlinux']
+++

## Latar Belakang

Pernah dalam situasi di mana harus mengakses server kantor atau homelab yang ada di balik NAT/firewall, tapi tidak bisa karena sedang di luar jaringan? Biasanya solusinya adalah setup VPN tradisional -- OpenVPN, WireGuard manual, atau minta IT buka port forwarding. Semua opsi ini butuh effort yang tidak sedikit dan maintenance yang cukup merepotkan.

**Tailscale** hadir sebagai solusi yang jauh lebih simpel. Tailscale adalah mesh VPN berbasis **WireGuard** yang membuat semua device kita terhubung dalam satu jaringan privat (disebut **tailnet**), tanpa perlu konfigurasi firewall, port forwarding, atau setup server VPN. Cukup install di setiap device, login, dan semua device langsung bisa saling berkomunikasi -- bahkan kalau berada di balik NAT yang berbeda.

## Permasalahan

Skenarionya sederhana: ada beberapa server di jaringan intranet (misalnya homelab atau kantor) yang perlu diakses dari luar. Masalahnya:

- Server berada di balik NAT/firewall tanpa public IP
- Port forwarding bukan opsi karena tidak punya kontrol penuh atas router, atau terlalu berisiko dari sisi keamanan
- VPN tradisional seperti OpenVPN butuh dedicated server, sertifikat, dan konfigurasi yang cukup rumit
- WireGuard manual butuh exchange public key dan konfigurasi endpoint di setiap device

Yang dibutuhkan adalah cara untuk mengakses server intranet dari mana saja -- dari laptop saat di kafe, dari HP saat di perjalanan -- tanpa harus memikirkan topologi jaringan.

## Pendekatan Solusi

Tailscale menyelesaikan ini dengan beberapa pendekatan:

1. **Mesh network peer-to-peer** -- device berkomunikasi langsung tanpa melewati server pusat, sehingga latency rendah
2. **NAT traversal otomatis** -- Tailscale menggunakan teknik seperti STUN dan DERP relay untuk menembus NAT tanpa port forwarding
3. **WireGuard di balik layar** -- semua koneksi terenkripsi end-to-end menggunakan WireGuard
4. **Subnet router** -- satu device di intranet bisa menjadi gateway untuk mengakses seluruh subnet, tanpa perlu install Tailscale di setiap server

Ada dua pendekatan untuk mengakses server intranet:

| Pendekatan | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **Install Tailscale di setiap server** | Koneksi langsung peer-to-peer, setiap server punya IP Tailscale sendiri | Harus install dan maintain Tailscale di setiap server |
| **Subnet router** | Cukup install di satu device, seluruh subnet bisa diakses | Traffic melewati router node, performa bergantung pada device router |

Untuk setup yang simpel dengan beberapa server, install langsung di setiap server lebih straightforward. Untuk intranet dengan banyak device (termasuk yang tidak bisa install Tailscale seperti printer atau IoT), subnet router lebih praktis.

## Implementasi Teknis

### Instalasi

Tailscale tersedia di repository resmi Archlinux (extra):

```
$ sudo pacman -S tailscale
```

Aktifkan daemon Tailscale:

```
$ sudo systemctl enable --now tailscaled
```

### Login dan Menghubungkan Device

Jalankan perintah berikut untuk menghubungkan device ke tailnet:

```
$ sudo tailscale up
```

Akan muncul URL untuk autentikasi di browser. Buka URL tersebut, login dengan akun Tailscale (bisa pakai Google, GitHub, atau provider lain), dan device akan terhubung ke tailnet.

Cek status koneksi:

```
$ tailscale status
```

Output-nya kurang lebih seperti ini:

```
100.64.0.1    laptop          user@  linux   -
100.64.0.2    server-lab      user@  linux   -
100.64.0.3    server-staging  user@  linux   -
```

Setiap device mendapat IP unik di range `100.x.y.z` yang bisa digunakan untuk berkomunikasi.

### Setup di Sisi Server (Intranet)

Lakukan hal yang sama di server yang ingin diakses. SSH ke server, install Tailscale, lalu jalankan:

```
$ sudo pacman -S tailscale
$ sudo systemctl enable --now tailscaled
$ sudo tailscale up --ssh
```

Flag `--ssh` mengaktifkan **Tailscale SSH** -- fitur yang memungkinkan SSH langsung via Tailscale tanpa perlu manage SSH key secara manual. Tailscale menangani autentikasi berdasarkan identitas user di tailnet.

Sekarang dari laptop, kita bisa langsung SSH ke server:

```
$ ssh user@server-lab
```

Atau menggunakan IP Tailscale:

```
$ ssh user@100.64.0.2
```

### Menggunakan Auth Key untuk Server Headless

Untuk server yang tidak punya browser (headless), gunakan **auth key** supaya tidak perlu buka URL untuk login:

1. Generate auth key di [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Jalankan di server:

```
$ sudo tailscale up --auth-key=tskey-auth-xxxxx
```

Untuk server yang harus selalu terhubung, gunakan auth key dengan opsi **reusable** dan **pre-approved** agar reconnect otomatis tanpa perlu approval ulang.

### Setup Subnet Router (Opsional)

Jika ingin mengakses seluruh subnet intranet tanpa install Tailscale di setiap device, jadikan satu mesin sebagai subnet router.

Pertama, aktifkan IP forwarding di mesin yang akan jadi router:

```
$ echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
$ echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
$ sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Lalu advertise subnet yang ingin diekspos:

```
$ sudo tailscale set --advertise-routes=192.168.1.0/24
```

Ganti `192.168.1.0/24` dengan subnet intranet yang sesuai. Setelah itu, approve route di [Tailscale Admin Console](https://login.tailscale.com/admin/machines) -- klik device yang menjadi router, lalu approve subnet route-nya.

Di device client yang ingin mengakses subnet tersebut, aktifkan penerimaan route:

```
$ sudo tailscale set --accept-routes
```

Sekarang device client bisa mengakses semua IP di subnet `192.168.1.0/24` melalui Tailscale, seolah-olah berada di jaringan yang sama.

### MagicDNS

Tailscale secara default mengaktifkan **MagicDNS** -- fitur yang memungkinkan kita mengakses device berdasarkan hostname, bukan IP. Jadi daripada menghafal `100.64.0.2`, cukup gunakan:

```
$ ssh user@server-lab
$ ping server-staging
```

Hostname diambil dari nama mesin saat pertama kali terhubung ke tailnet, dan bisa diubah lewat Admin Console.

## Tantangan yang Dihadapi

Tantangan pertama adalah soal DNS resolution. Di Archlinux, jika menggunakan `systemd-resolved`, Tailscale biasanya bisa mengkonfigurasi DNS secara otomatis. Tapi jika menggunakan setup DNS custom (misalnya langsung edit `/etc/resolv.conf`), MagicDNS mungkin tidak bekerja out of the box. Perlu memastikan bahwa `tailscaled` punya akses untuk mengatur DNS, atau set `--accept-dns=false` dan handle routing DNS secara manual.

Tantangan kedua terkait subnet router -- traffic yang melewati subnet router menggunakan **SNAT (Source NAT)** secara default. Artinya, server di intranet melihat semua traffic seolah datang dari IP mesin router, bukan dari IP Tailscale device asal. Ini bisa jadi masalah untuk logging atau access control di sisi server. Jika butuh source IP asli, bisa disable SNAT:

```
$ sudo tailscale set --snat-subnet-routes=false
```

Tapi perlu diingat, ini membutuhkan routing tambahan di sisi intranet agar reply packet bisa kembali ke mesin router.

## Insight dan Pembelajaran

Setelah menggunakan Tailscale untuk remote server intranet, beberapa insight:

- **Setup-nya absurdly simple** -- dari install sampai bisa SSH ke server intranet, literally butuh waktu kurang dari 5 menit per device. Bandingkan dengan setup OpenVPN atau WireGuard manual yang bisa makan waktu berjam-jam.
- **Tailscale SSH mengurangi overhead manage SSH key** -- tidak perlu lagi copy-paste public key ke setiap server. Autentikasi berbasis identitas Tailscale jauh lebih praktis.
- **Subnet router adalah fitur underrated** -- satu device sebagai gateway sudah cukup untuk mengakses seluruh intranet. Sangat berguna untuk akses ke device yang tidak bisa install Tailscale.
- **MagicDNS bikin hidup lebih mudah** -- tidak perlu lagi maintain daftar IP. Cukup gunakan hostname dan Tailscale yang handle resolution-nya.
- **Perlu perhatikan key expiry** -- secara default, key Tailscale di setiap device punya masa berlaku. Jika expired, device terputus dari tailnet. Untuk server kritikal, pastikan enable key expiry disable atau set reminder untuk re-authenticate sebelum expired.

## Penutup

Tailscale mengubah cara kita mengakses server di intranet. Tidak perlu lagi ribet setup VPN server, buka port di firewall, atau manage sertifikat. Cukup install, login, dan semua device langsung terhubung dalam jaringan privat yang aman. Untuk yang sering butuh akses remote ke homelab atau server kantor dari mana saja, Tailscale adalah solusi yang worth it untuk dicoba. Free tier-nya sudah cukup generous untuk penggunaan personal -- hingga 100 device dalam satu tailnet.

## Referensi

- [Tailscale Documentation](https://tailscale.com/kb) -- Diakses pada 2026-04-05
- [Tailscale - What is Tailscale](https://tailscale.com/kb/1151/what-is-tailscale) -- Diakses pada 2026-04-05
- [Tailscale - Subnet Routers](https://tailscale.com/kb/1019/subnets) -- Diakses pada 2026-04-05
- [Arch Wiki - Tailscale](https://wiki.archlinux.org/title/Tailscale) -- Diakses pada 2026-04-05
