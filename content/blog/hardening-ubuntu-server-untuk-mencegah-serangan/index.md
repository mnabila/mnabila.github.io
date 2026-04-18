+++
draft = false
date = '2026-04-18'
title = 'Hardening Ubuntu Server Untuk Mencegah Serangan'
type = 'blog'
description = 'Panduan hardening Ubuntu Server untuk mencegah serangan umum seperti brute force SSH, crypto miner, dan malware'
image = ''
tags = ['security', 'ubuntu', 'server', 'hardening', 'linux']
+++

## Latar Belakang

Pernah ngecek server dan tiba-tiba CPU usage mentok 100% padahal tidak ada workload yang berjalan? Buka `top`, ada proses dengan nama random yang memakan seluruh resource. Cek koneksi outbound, ada traffic ke IP asing di port aneh sehingga bisa disimpulkan server ini sudah disusupi crypto miner.

Kalau deploy Ubuntu Server di VPS atau public cloud dengan konfigurasi default, server tersebut sudah langsung menjadi target operasi mereka. SSH exposed di port 22, firewall belum aktif, root login masih terbuka -- dalam hitungan jam, bot otomatis sudah mulai brute force. Tidak perlu jadi target spesifik; cukup punya IP publik saja sudah cukup untuk masuk radar scanner.

**Hardening** adalah proses memperkuat konfigurasi server agar attack surface sekecil mungkin. Bukan tool atau software tunggal, tapi kumpulan langkah-langkah konfigurasi yang secara keseluruhan membuat server jauh lebih sulit ditembus.

## Permasalahan

Secara default, Ubuntu Server di VPS/public cloud punya beberapa kelemahan yang sering diabaikan:

- **SSH terbuka di port 22 dengan password authentication** -- ini target utama brute force bot yang scan seluruh internet 24/7
- **Root login masih aktif** -- kalau password root lemah atau bocor, attacker langsung dapat akses penuh tanpa perlu privilege escalation
- **Firewall tidak aktif** -- semua port terbuka, service apapun yang berjalan langsung accessible dari luar
- **Tidak ada proteksi brute force** -- tidak ada mekanisme untuk memblokir IP yang gagal login berulang kali
- **Service default yang tidak dibutuhkan tetap berjalan** -- menambah attack surface tanpa manfaat
- **Tidak ada monitoring atau audit logging** -- tidak ada cara untuk tahu kalau ada proses mencurigakan atau koneksi outbound aneh

Yang dibutuhkan adalah baseline security yang bisa diterapkan segera setelah server pertama kali di-deploy, tanpa perlu setup yang terlalu rumit.

## Pendekatan Solusi

Ada beberapa layer hardening yang bisa diterapkan, masing-masing menangani aspek keamanan yang berbeda:

| Layer | Tujuan | Tools/Metode |
|-------|--------|-------------|
| **Access control** | Membatasi siapa yang bisa masuk | Non-root user, SSH key-only, custom port |
| **Network filtering** | Membatasi port/service yang terekspos | UFW (firewall) |
| **Brute force protection** | Memblokir percobaan login berulang | Fail2ban |
| **System maintenance** | Menutup vulnerability yang sudah diketahui | Unattended upgrades |
| **Malware detection** | Mendeteksi rootkit dan software mencurigakan | rkhunter, chkrootkit |
| **Audit dan monitoring** | Tracking aktivitas dan deteksi anomali | auditd, journalctl, proses monitoring |

Pendekatan yang dipakai di sini adalah **defense in depth** -- tidak bergantung pada satu layer saja, tapi kombinasi beberapa mekanisme yang saling melengkapi. Kalau satu layer bypass, layer berikutnya masih bisa menahan.

## Implementasi Teknis

### Membuat User Non-Root dan Disable Root Login

Langkah paling basic -- jangan pernah pakai root untuk operasi sehari-hari. Buat user biasa dengan akses sudo:

```
$ adduser admin
$ usermod -aG sudo admin
```

Setelah memastikan user baru bisa login dan menjalankan `sudo`, disable root login di SSH. Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
```

Restart SSH:

```
$ sudo systemctl restart sshd
```

Dengan ini, meskipun attacker tahu password root, mereka tidak bisa login via SSH.

### Mengamankan SSH: Custom Port dan Key-Based Authentication

Ganti port SSH default untuk mengurangi noise dari automated scanner. Di `/etc/ssh/sshd_config`:

```
Port 2222
```

Lalu setup key-based authentication dan disable password login sepenuhnya. Di mesin lokal, generate SSH key kalau belum punya:

```
$ ssh-keygen -t ed25519 -C "email@domain.com"
```

Copy public key ke server:

```
$ ssh-copy-id -p 2222 admin@server-ip
```

Setelah memastikan bisa login dengan key, disable password authentication di `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```
$ sudo systemctl restart sshd
```

> **Note:** Pastikan sudah bisa login dengan SSH key sebelum disable password authentication. Kalau tidak, bisa terkunci dari server sendiri.

### Mengaktifkan Firewall dengan UFW

UFW (Uncomplicated Firewall) sudah terinstall di Ubuntu tapi tidak aktif secara default. Aktifkan dan hanya buka port yang dibutuhkan:

```
$ sudo ufw allow 2222/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443/tcp
$ sudo ufw enable
```

Cek status:

```
$ sudo ufw status verbose
```

Prinsipnya: **deny all, allow specific**. Hanya buka port yang memang dibutuhkan oleh service yang berjalan.

### Proteksi Brute Force dengan Fail2ban

Fail2ban memonitor log file dan secara otomatis memblokir IP yang menunjukkan perilaku mencurigakan, seperti gagal login berulang kali:

```
$ sudo apt install fail2ban
```

Buat konfigurasi lokal agar tidak tertimpa saat update:

```
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`, sesuaikan konfigurasi untuk SSH:

```ini
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

Konfigurasi di atas akan memblokir IP selama 1 jam setelah 3 kali gagal login dalam 10 menit. Aktifkan fail2ban:

```
$ sudo systemctl enable --now fail2ban
```

Cek status banned IP:

```
$ sudo fail2ban-client status sshd
```

### Automatic Security Updates

Vulnerability baru ditemukan setiap hari, dan patch biasanya dirilis cukup cepat. Aktifkan automatic security updates agar tidak perlu manual:

```
$ sudo apt install unattended-upgrades
$ sudo dpkg-reconfigure -plow unattended-upgrades
```

Ini akan otomatis menginstall security patches tanpa perlu intervensi manual. Untuk update manual yang lebih menyeluruh:

```
$ sudo apt update && sudo apt upgrade -y
```

### Disable Service yang Tidak Digunakan

Setiap service yang berjalan adalah potensi attack surface. Cek service apa saja yang aktif:

```
$ sudo systemctl list-units --type=service --state=running
```

Prinsipnya: kalau tidak tahu kenapa sebuah service berjalan, cari tahu dulu sebelum men-disable-nya. Tapi kalau memang tidak dibutuhkan, matikan.

### Deteksi Malware dengan rkhunter dan chkrootkit

Install tools untuk mendeteksi rootkit dan malware:

```
$ sudo apt install rkhunter chkrootkit
```

Jalankan scan:

```
$ sudo rkhunter --check --skip-keypress
$ sudo chkrootkit
```

Untuk scan otomatis mingguan, tambahkan cron job:

```
$ sudo crontab -e
```

```
0 3 * * 1 /usr/bin/rkhunter --check --skip-keypress --report-warnings-only | mail -s "rkhunter weekly scan" admin@domain.com
```

### Setup Audit dan Logging

Logging yang baik adalah kunci untuk investigasi kalau terjadi insiden. Install dan konfigurasi auditd:

```
$ sudo apt install auditd
$ sudo systemctl enable --now auditd
```

Tambahkan beberapa audit rule dasar untuk memonitor perubahan pada file konfigurasi kritis:

```
$ sudo auditctl -w /etc/passwd -p wa -k identity
$ sudo auditctl -w /etc/shadow -p wa -k identity
$ sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config
```

Untuk melihat log:

```
$ sudo journalctl -u sshd --since "1 hour ago"
$ sudo ausearch -k identity --interpret
```

### Monitoring Proses Mencurigakan

Biasakan untuk memeriksa proses yang berjalan dan koneksi network yang aktif:

```
$ top -b -n 1 | head -20
$ ss -tulnp
$ sudo netstat -tulnp
```

Kalau ada proses dengan nama random yang memakan CPU tinggi, atau koneksi outbound ke IP/port yang tidak dikenal -- itu red flag yang perlu diinvestigasi segera.

## Tantangan yang Dihadapi

Tantangan pertama adalah **mendeteksi crypto miner yang sudah terlanjur masuk**. Pernah mengalami situasi di mana server tiba-tiba lambat, CPU 100% terus-menerus. Saat dicek dengan `top`, terlihat proses asing dengan nama random. Investigasi lebih lanjut menunjukkan proses tersebut masuk melalui SSH brute force (password lemah + root login aktif), menginstall dirinya via crontab, dan menambahkan SSH key asing ke `authorized_keys` untuk persistent access. Langkah response-nya: kill proses, bersihkan crontab dan authorized_keys, ganti semua password, lalu terapkan hardening. Tapi kalau tidak yakin sejauh mana server ter-compromise, rebuild dari awal lebih aman daripada membersihkan satu per satu.

Tantangan kedua adalah **menentukan port mana yang perlu dibuka di firewall**. Kalau terlalu ketat, service yang legitimate bisa terganggu. Kalau terlalu longgar, sama saja tidak pakai firewall. Pendekatannya: mulai dari deny all, lalu buka port satu per satu sesuai kebutuhan -- SSH, HTTP/HTTPS, dan port spesifik yang dipakai aplikasi. Kalau ragu apakah suatu port dibutuhkan, coba tutup dulu dan lihat apakah ada service yang terdampak.

Satu hal lagi -- **false positive dari fail2ban** bisa jadi masalah. Kalau `maxretry` terlalu rendah atau `findtime` terlalu panjang, IP yang legitimate (termasuk IP sendiri) bisa ikut ter-ban. Untuk mengatasinya, tambahkan IP trusted ke whitelist di konfigurasi fail2ban menggunakan directive `ignoreip`.

## Insight dan Pembelajaran

Beberapa hal yang bisa diambil dari pengalaman hardening server:

- **Hardening bukan sekali setup lalu selesai** -- threat landscape terus berubah, vulnerability baru terus ditemukan. Minimal review konfigurasi secara berkala dan pastikan automatic updates aktif.
- **Server publik tanpa hardening hampir pasti akan disusupi** -- dari pengalaman, server dengan konfigurasi default di public cloud bisa mulai menerima brute force attempt dalam hitungan menit setelah deploy.
- **Key-based SSH authentication adalah non-negotiable** -- password authentication di server publik itu terlalu berisiko. Satu langkah ini saja sudah mengeliminasi mayoritas brute force attack.
- **Defense in depth lebih efektif daripada satu solusi** -- firewall saja tidak cukup, fail2ban saja tidak cukup. Kombinasi beberapa layer yang saling melengkapi memberikan proteksi yang jauh lebih baik.
- **Logging dan monitoring sering diabaikan tapi krusial** -- tanpa logging yang baik, kalau server disusupi, tidak ada cara untuk tahu apa yang terjadi, kapan, dan bagaimana. Ini menyulitkan incident response dan mencegah pembelajaran dari insiden.
- **Gunakan configuration management untuk konsistensi** -- kalau punya banyak server, gunakan tools seperti Ansible untuk menerapkan hardening secara konsisten. Manual setup satu per satu rawan terlewat dan tidak scalable.

## Penutup

Hardening Ubuntu Server bukan sesuatu yang rumit, tapi sering terabaikan karena dianggap "nanti saja". Padahal minimal baseline seperti secure SSH (key-only, custom port, no root), firewall aktif, fail2ban, dan automatic security updates sudah bisa mencegah mayoritas serangan umum. Semua langkah di atas tidak membutuhkan waktu lama untuk diimplementasikan, tapi dampaknya signifikan -- perbedaan antara server yang bertahan berbulan-bulan tanpa insiden dan server yang disusupi dalam hitungan jam setelah deploy.

## Referensi

- [Linux Server Hardening Security Tips - nixCraft](https://www.cyberciti.biz/tips/linux-security.html) -- Diakses pada 2026-04-18
