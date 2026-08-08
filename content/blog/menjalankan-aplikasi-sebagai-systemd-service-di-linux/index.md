+++
draft = false
date = '2026-08-08'
title = 'Menjalankan Aplikasi sebagai Systemd Service di Linux'
type = 'blog'
description = 'Cara menjalankan aplikasi custom sebagai systemd service supaya jalan otomatis saat boot, restart sendiri ketika crash, dan lognya tercatat rapi'
image = ''
tags = ['systemd', 'linux', 'daemon', 'deployment', 'service']
+++

## Latar Belakang

Tiap kali selesai build aplikasi backend, entah binary Go atau Rust, pertanyaannya selalu sama begitu mau deploy ke server. Bagaimana supaya aplikasi ini tetap jalan setelah saya logout dari SSH, tetap hidup kalau server reboot, dan nyala lagi sendiri kalau tiba-tiba crash. Jalanin binary langsung dari terminal jelas bukan opsi, karena begitu sesi SSH ditutup, prosesnya ikut mati bareng shell.

Kalau lagi buru-buru, saya kadang lempar ke PM2 kalau kebetulan sudah terpasang di server, atau bungkus pakai Docker kalau memang mau dicontainerisasi. Tapi keduanya bawa beban sendiri. PM2 butuh runtime Node.js cuma buat momong satu proses, sedangkan Docker kerasa berlebihan buat jalanin satu binary yang sebenarnya tinggal ditaruh di host. Yang saya butuhkan sebenarnya simpel: cara menjalankan satu binary secara native di server yang tetap hidup setelah saya logout, nyala lagi saat crash, dan lognya tercatat rapi, tanpa nambah dependency atau layer baru yang harus ikut dijaga.

## Permasalahan

Deploy binary tanpa proses manager yang bener bikin beberapa masalah kerasa langsung:

- **Proses mati saat logout**: binary yang dijalankan langsung di terminal nempel ke sesi SSH. Begitu koneksi putus, prosesnya ikut mati bareng shell
- **Mati total setelah reboot**: habis server restart, aplikasi tidak nyala sendiri. Harus login manual lalu jalankan lagi satu-satu
- **Tidak ada auto-restart**: pas aplikasi crash karena bug atau kehabisan memori, prosesnya berhenti gitu aja dan nggak ada yang menghidupkan lagi
- **Log berantakan**: output aplikasi cuma nongol di terminal atau di-redirect ke file manual, tanpa rotasi dan tanpa cara standar buat baca ulang
- **Ribet dikelola**: nggak ada cara seragam buat start, stop, restart, atau cek status. Semuanya manual pakai `ps` dan `kill`
- **Alternatif nambah dependency**: PM2 butuh runtime Node.js dan Docker nambah layer image plus registry, padahal yang mau saya jalanin cuma satu binary statis di host

## Pendekatan Solusi

Ada beberapa cara untuk menjalankan aplikasi sebagai background process yang persisten:

| Pendekatan          | Kelebihan                                                  | Kekurangan                                                       |
| ------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------- |
| **PM2**             | Enak untuk ekosistem Node.js, restart dan log built-in     | Butuh runtime Node.js meski aplikasinya bukan JS                 |
| **Docker**          | Isolasi penuh, dependency terbungkus, portable antar host  | Perlu build image, registry, dan satu layer runtime lagi         |
| **Supervisor**      | Fitur lengkap, config sederhana                            | Perlu install paket tambahan, satu daemon lagi yang harus dijaga |
| **Systemd service** | Sudah built-in, survive reboot, auto-restart, log terpusat | Sintaks unit file perlu dipelajari dulu                          |

Saya pilih **systemd service** karena beberapa alasan:

1. **Sudah ada di sistem**: nggak perlu install apapun. `systemd` itu init system default di Ubuntu, Debian, Arch, dan mayoritas distro modern, jadi tidak nambah dependency seperti PM2 atau layer image seperti Docker
2. **Fiturnya lengkap**: auto-restart, dependency ordering, resource limit, sampai log lewat `journald` semuanya sudah tersedia tanpa tool lain
3. **Pas buat satu binary di host**: aplikasi backend yang cuma perlu hidup terus dijalanin native, tanpa overhead containerisasi yang berlebihan
4. **Standar dan portable**: unit file yang sama bisa dipakai di hampir semua server Linux, jadi ilmunya kepakai lagi di project lain

## Implementasi Teknis

Contohnya, saya punya binary hasil build aplikasi backend bernama `myapp` yang listen di port 8080. Binary statis seperti ini paling pas dijalankan lewat systemd karena tinggal ditunjuk langsung di `ExecStart`, tanpa perlu runtime atau interpreter tambahan.

### Menyiapkan User dan Lokasi Aplikasi

Jalanin aplikasi sebagai `root` itu kebiasaan buruk. Kalau aplikasinya punya celah keamanan, penyerang langsung dapat akses root. Jadi saya bikin user sistem khusus tanpa shell login buat jalanin aplikasinya.

```bash
# Buat user sistem tanpa home dan tanpa shell login
$ sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp

# Letakkan binary di lokasi standar
$ sudo mkdir -p /opt/myapp
$ sudo cp myapp /opt/myapp/
$ sudo chown -R myapp:myapp /opt/myapp
$ sudo chmod +x /opt/myapp/myapp
```

### Menulis Unit File

Unit file adalah jantung dari konfigurasi service. File ini diletakkan di `/etc/systemd/system/` dengan ekstensi `.service`.

```bash
$ sudo nano /etc/systemd/system/myapp.service
```

Isi dengan konfigurasi berikut:

```ini
[Unit]
Description=My Application Backend Service
Documentation=https://github.com/mnabila/myapp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/myapp --port 8080
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Setiap section punya peran masing-masing:

- **[Unit]** mendeskripsikan service dan dependency-nya. `After=network-online.target` dan `Wants=network-online.target` memastikan service baru dijalankan setelah jaringan benar-benar siap, penting untuk aplikasi yang butuh bind ke port atau koneksi ke database
- **[Service]** mengatur bagaimana proses dijalankan. Ini bagian yang paling sering disesuaikan
- **[Install]** menentukan kapan service diaktifkan saat `enable`. `WantedBy=multi-user.target` berarti service ikut nyala saat sistem mencapai mode multi-user normal, yaitu kondisi boot standar sebuah server

Beberapa direktif penting di `[Service]`:

| Direktif           | Fungsi                                                                        |
| ------------------ | ----------------------------------------------------------------------------- |
| `Type=simple`      | systemd menganggap proses langsung jalan di foreground (default, paling umum) |
| `User` / `Group`   | menjalankan proses sebagai user non-root demi keamanan                        |
| `WorkingDirectory` | direktori kerja proses, penting kalau aplikasi memakai path relatif           |
| `ExecStart`        | perintah lengkap untuk menjalankan aplikasi, wajib memakai path absolut       |
| `Restart`          | aturan restart. `on-failure` hanya restart saat exit code bukan 0             |
| `RestartSec`       | jeda sebelum systemd mencoba restart ulang                                    |

### Mengaktifkan dan Menjalankan Service

Setiap kali menambah atau mengubah unit file, systemd perlu diberi tahu untuk membaca ulang konfigurasi.

```bash
# Baca ulang konfigurasi systemd
$ sudo systemctl daemon-reload

# Aktifkan supaya nyala otomatis saat boot
$ sudo systemctl enable myapp

# Jalankan sekarang
$ sudo systemctl start myapp
```

Perintah `enable` dan `start` bisa digabung dengan `sudo systemctl enable --now myapp`.

### Verifikasi Status

Cek apakah service berjalan dengan benar:

```bash
$ sudo systemctl status myapp
```

Kalau berhasil, outputnya akan menampilkan status aktif seperti ini:

```
● myapp.service - My Application Backend Service
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-08-08 14:00:03 WIB; 12s ago
   Main PID: 4213 (myapp)
      Tasks: 7 (limit: 4915)
     Memory: 12.4M
        CPU: 45ms
     CGroup: /system.slice/myapp.service
             └─4213 /opt/myapp/myapp --port 8080
```

Perhatikan baris `Active: active (running)` dan `enabled` pada baris `Loaded`. Keduanya menandakan service sedang berjalan sekaligus akan otomatis nyala saat boot.

### Membaca Log Aplikasi

Karena systemd mengarahkan output aplikasi ke `journald`, semua log bisa dibaca dengan `journalctl` tanpa perlu setup file log manual.

```bash
# Lihat semua log service
$ sudo journalctl -u myapp

# Ikuti log secara real time seperti tail -f
$ sudo journalctl -u myapp -f

# Lihat log sejak boot terakhir saja
$ sudo journalctl -u myapp -b

# Batasi ke 100 baris terakhir
$ sudo journalctl -u myapp -n 100
```

### Mengelola Konfigurasi Lewat Environment

Menaruh konfigurasi sensitif seperti kredensial database langsung di `ExecStart` bukan ide bagus. Systemd mendukung file environment terpisah lewat `EnvironmentFile`.

```bash
$ sudo nano /etc/myapp/myapp.env
```

Isi dengan pasangan key value:

```conf
DATABASE_URL=postgres://user:secret@10.10.10.10:5432/mydb
LOG_LEVEL=info
APP_PORT=8080
```

Lalu referensikan di unit file dan amankan permission-nya supaya hanya user aplikasi yang bisa membaca:

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
ExecStart=/opt/myapp/myapp
```

```bash
$ sudo chown myapp:myapp /etc/myapp/myapp.env
$ sudo chmod 640 /etc/myapp/myapp.env
```

Setelah mengubah unit file, jalankan lagi `daemon-reload` diikuti `restart`:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart myapp
```

## Tantangan yang Dihadapi

Tantangan pertama datang dari **salah pilih `Type`**. Awalnya saya pakai `Type=forking` untuk aplikasi yang sebenarnya jalan di foreground, jadi systemd nunggu fork yang tidak pernah terjadi lalu menganggap service gagal start. Aturannya simpel: aplikasi yang jalan di foreground pakai `Type=simple`, dan `Type=forking` hanya untuk yang sengaja fork ke background.

Masalah kedua soal **path relatif**. Aplikasi saya baca `./config.yaml`, dan tanpa `WorkingDirectory` systemd menjalankan proses dari direktori root `/` sehingga file-nya tidak ketemu. Solusinya, set `WorkingDirectory` secara eksplisit atau ubah aplikasi biar pakai path absolut.

Tantangan ketiga adalah **restart loop yang liar** saat aplikasi crash karena config salah, di mana `Restart=on-failure` bikin systemd nyoba hidupin ulang terus tiap beberapa detik. Untungnya ada pengaman bawaan yang menghentikan percobaan dan nandain service-nya `failed` kalau gagal lebih dari 5 kali dalam 10 detik. Saya juga naikkan `RestartSec` jadi 5 detik biar restart-nya tidak terlalu agresif.

Terakhir, **log hilang setelah reboot**. Di beberapa distro `journald` menyimpan log di memori (`/run/log/journal`), jadi log lenyap tiap restart. Cukup bikin direktori `/var/log/journal` lalu jalankan `sudo systemctl restart systemd-journald`, dan log jadi persisten lintas reboot.

## Insight dan Pembelajaran

- **Systemd sudah cukup buat mayoritas kebutuhan process management**: sebelum buru-buru pasang PM2 atau bungkus aplikasi pakai Docker, sadari systemd sudah ngasih auto-restart, dependency ordering, resource limit, dan log terpusat gratis tanpa dependency tambahan
- **Jangan jalanin aplikasi sebagai root tanpa alasan**: bikin user sistem khusus dengan `--shell /usr/sbin/nologin`. Kalau aplikasi kebobolan, dampaknya mentok di satu user, bukan seluruh sistem
- **`daemon-reload` wajib tiap unit file berubah**: banyak yang bingung kenapa perubahannya nggak ngefek, ternyata lupa nyuruh systemd baca ulang konfigurasi
- **`WorkingDirectory` dan path absolut nyegah bug misterius**: beda direktori kerja antara jalanin manual dan lewat systemd itu sumber bug yang sering kelewat
- **Pisahin konfigurasi pakai `EnvironmentFile`**: naruh kredensial di unit file yang bisa dibaca semua orang itu lubang keamanan. File environment terpisah dengan permission `640` jauh lebih aman
- **Pelajari `journalctl` beneran**: bisa filter log berdasarkan unit, waktu, dan boot session bikin debugging jauh lebih cepat dibanding ngubek file log manual

## Penutup

Menjalankan aplikasi sebagai systemd service ngasih saya cara jalanin binary yang native ke sistem, tanpa perlu runtime PM2 atau layer image Docker. Cukup satu unit file, aplikasi custom langsung diperlakukan sama seperti service bawaan sistem. Aplikasi nyala otomatis saat boot, restart sendiri saat crash, lognya rapi, dan perintah pengelolaannya seragam. Kalau butuh lebih jauh, systemd masih menyimpan banyak fitur seperti resource limit lewat `MemoryMax` dan `CPUQuota`, sandboxing pakai `ProtectSystem` dan `PrivateTmp`, sampai systemd timer sebagai pengganti cron yang lebih fleksibel. Nguasain dasar unit file adalah pintu masuk ke semua itu.

## Referensi

- [systemd.service - Service unit configuration](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html), diakses pada 2026-08-08
- [systemd.unit - Unit configuration](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html), diakses pada 2026-08-08
- [journalctl - Query the systemd journal](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html), diakses pada 2026-08-08
- [systemd for Administrators - ArchWiki](https://wiki.archlinux.org/title/Systemd), diakses pada 2026-08-08
