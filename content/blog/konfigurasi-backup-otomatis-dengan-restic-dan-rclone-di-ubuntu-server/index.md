+++
draft = false
date = '2026-08-08'
title = 'Konfigurasi Backup Otomatis dengan Restic dan Rclone di Ubuntu Server'
type = 'blog'
description = 'Cara mengarsipkan log aplikasi ke storage terpisah sebelum dihapus logrotator, terenkripsi dan terjadwal otomatis dengan restic dan rclone, lengkap dengan retention policy dan verifikasi restore'
image = ''
tags = ['restic', 'rclone', 'ubuntu']
+++

## Latar Belakang

Log aplikasi gampang diabaikan sampai suatu hari benar-benar dibutuhkan. Aplikasi saya di `/opt/myapp/log` nulis log terus-menerus, dan biar disk tidak penuh, log-nya dirotate secara berkala. Rotasi ini otomatis ngehapus log lama setelah beberapa waktu. Masalahnya kerasa pas saya perlu nyelidikin kejadian yang sudah agak lama, misalnya nyari kapan sebuah error pertama muncul atau ngerekonstruksi sebuah insiden. Sering kali pas dicari, log-nya sudah keburu dihapus logrotator. Di beberapa post sebelumnya saya sudah bahas cara [menjadwalkan task dengan systemd timer](/blog/menjadwalkan-task-dengan-systemd-timer-sebagai-pengganti-cron-di-linux/) dan sinkronisasi Docker volume antar server, tapi keduanya belum nutup skenario ini: menyelamatkan log sebelum logrotator menghapusnya permanen.

Jalan keluarnya adalah ngarsipin log ke lokasi terpisah sebelum dirotate. Simpan salinannya di storage lain, jadi kalaupun log lokal sudah dibuang rotasi, arsipnya tetap ada buat diperiksa nanti. Tapi ngarsipin log bikin beberapa syarat baru. Log sering ngandung info sensitif seperti IP, path internal, atau token, jadi tidak boleh disimpan mentah di storage terpisah dan harus terenkripsi. Log itu repetitif dan numpuk cepat, jadi nyalin ulang semuanya tiap hari boros storage dan bandwidth, butuh deduplikasi dan incremental. Dan arsip yang tidak pernah diuji restore sama saja dengan tidak punya arsip. Ketiga syarat inilah yang harus dipenuhi sebelum saya percaya sama arsipnya.

## Permasalahan

Ada beberapa hal spesifik yang harus dipenuhi strategi arsip log ini:

- **Log dihapus rotasi**: log lama otomatis dibuang biar disk tidak penuh, jadi begitu perlu nyelidikin kejadian lama, datanya sering sudah tidak ada
- **Enkripsi wajib**: log bisa ngandung IP, path internal, sampai token. Naruh log mentah di storage terpisah itu berisiko, kredensial bocor berarti seluruh isi log ikut terbaca
- **Full backup boros**: nyalin ulang semua log tiap hari menghabiskan bandwidth dan storage, apalagi log numpuk cepat dan sebagian besar isinya tidak berubah
- **Arsip manual sering lupa**: mengandalkan ingatan buat ngarsipin log itu resep bencana. Prosesnya harus otomatis dan terjadwal
- **Restore yang tidak teruji**: banyak orang baru sadar arsipnya korup atau tidak lengkap pas benar-benar butuh restore, di momen paling genting

## Pendekatan Solusi

Ada beberapa tool backup yang umum dipakai di Linux:

| Tool           | Kelebihan                                               | Kekurangan                                                 |
| -------------- | ------------------------------------------------------- | ---------------------------------------------------------- |
| **rsync**      | Simpel, cepat, ada di mana-mana                         | Tidak ada enkripsi, snapshot, atau deduplikasi bawaan      |
| **BorgBackup** | Deduplikasi dan enkripsi bagus                          | Backend terbatas, butuh Borg terpasang di sisi remote      |
| **Duplicity**  | Enkripsi GPG, banyak backend                            | Incremental berbasis chain yang rapuh, restore bisa lambat |
| **Restic**     | Enkripsi default, deduplikasi, snapshot, banyak backend | Operasi prune bisa berat, perlu manajemen password         |

Saya pilih menggunakan **restic** yang dipadukan dengan **rclone** dengan alasan:

1. **Terenkripsi secara default**: restic mengenkripsi semua data sebelum keluar dari server. Storage cloud cuma lihat blob terenkripsi, bukan isi file
2. **Deduplikasi dan snapshot**: restic memecah data jadi blok dan cuma menyimpan blok baru. Tiap backup adalah snapshot lengkap secara logis, tapi hemat secara fisik
3. **Rclone membuka semua backend**: restic mendukung beberapa backend secara native. Tapi lewat rclone, restic bisa nulis ke puluhan provider cloud, dari S3-compatible, Google Drive, sampai Backblaze B2, dengan satu antarmuka yang sama

## Implementasi Teknis

Sebagai contoh, saya akan melakukan backup direktori log `/opt/myapp/log` ke RustFS, object storage S3-compatible yang saya jalankan sendiri, lewat rclone. Pola yang sama berlaku buat direktori atau backend lain yang didukung rclone.

### Instalasi Restic dan Rclone

Restic dan rclone sudah tersedia di repository Ubuntu, tapi versinya kerap tertinggal beberapa rilis. Biar dapat fitur dan perbaikan terbaru, saya pasang dari sumber resminya.

```bash
# Install dari repository (paling simpel)
$ sudo apt update
$ sudo apt install restic rclone

# Verifikasi versi
$ restic version
$ rclone version
```

### Konfigurasi Remote Rclone

Rclone menyimpan konfigurasi remote di file terenkripsi. Jalankan wizard interaktif buat nambah remote baru:

```bash
$ rclone config
```

Wizard bakal nanya beberapa hal berurutan. Untuk RustFS yang S3-compatible, jawabannya kurang lebih:

- **name**: `backup-cloud`
- **Storage**: `s3`
- **provider**: `Other` (RustFS belum jadi named-provider di rclone, jadi diperlakukan sebagai S3 generik)
- **access_key_id** dan **secret_access_key**: kredensial dari RustFS
- **endpoint**: URL endpoint RustFS, misal `http://10.10.10.13:9000`

Setelah selesai, uji koneksi remote dengan melihat isi bucket:

```bash
# Lihat daftar remote yang terkonfigurasi
$ rclone listremotes

# Tes akses ke bucket
$ rclone ls backup-cloud:my-backup-bucket
```

### Menyiapkan Password dan Environment Restic

Restic butuh password buat mengenkripsi repository. Jangan pernah taruh password ini di dalam script atau riwayat shell. Simpan di file terpisah dengan permission ketat.

```bash
# Buat file password dengan permission hanya untuk root
$ sudo mkdir -p /etc/restic
$ openssl rand -base64 32 | sudo tee /etc/restic/password > /dev/null
$ sudo chmod 600 /etc/restic/password
```

Agar tidak perlu ngetik lokasi repository dan password berulang kali, saya taruh di environment file yang nanti dipakai systemd:

```bash
$ sudo nano /etc/restic/restic.env
```

Isi dengan variabel berikut:

```conf
RESTIC_REPOSITORY=rclone:backup-cloud:my-backup-bucket/server-01
RESTIC_PASSWORD_FILE=/etc/restic/password
```

Format `rclone:backup-cloud:my-backup-bucket/server-01` ini ngasih tahu restic buat pakai backend rclone dengan remote `backup-cloud` dan path `my-backup-bucket/server-01`.

### Inisialisasi Repository

Sebelum backup pertama, repository harus diinisialisasi dulu. Muat environment-nya biar perintah restic tahu lokasi dan password-nya.

```bash
$ set -a && source /etc/restic/restic.env && set +a
$ sudo -E restic init
```

Kalau berhasil, restic bakal konfirmasi repository sudah dibuat:

```
created restic repository 8f3a2b1c at rclone:backup-cloud:my-backup-bucket/server-01

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is irrecoverably lost.
```

### Backup Pertama

Sekarang jalankan backup ke direktori yang mau diamankan:

```bash
$ sudo -E restic backup /opt/myapp/log
```

Restic bakal nampilin progres dan ringkasan di akhir:

```
Files:        1284 new,     0 changed,     0 unmodified
Dirs:          213 new,     0 changed,     0 unmodified
Added to the repository: 342.516 MiB (128.004 MiB stored)

processed 1284 files, 342.516 MiB in 0:47
snapshot a1b2c3d4 saved
```

Perhatikan baris `Added to the repository`. Dari 342 MiB data, cuma 128 MiB yang benar-benar tersimpan setelah kompresi dan deduplikasi. Di backup berikutnya angka ini bakal jauh lebih kecil, karena cuma blok yang berubah yang dikirim.

### Verifikasi dan Uji Restore

Backup yang belum diuji restore itu tidak bisa dipercaya. Pertama, lihat daftar snapshot yang ada:

```bash
$ sudo -E restic snapshots
```

```
ID        Time                 Host        Tags        Paths
------------------------------------------------------------------
a1b2c3d4  2026-08-08 10:15:03  server-01               /opt/myapp/log
------------------------------------------------------------------
1 snapshots
```

Lalu uji restore ke direktori sementara, jangan langsung nimpa data asli:

```bash
$ sudo -E restic restore a1b2c3d4 --target /tmp/restore-test
```

Bandingkan hasilnya dengan sumber buat mastiin integritasnya. Selain itu, restic bisa memverifikasi struktur internal repository:

```bash
# Verifikasi metadata repository
$ sudo -E restic check

# Verifikasi sekaligus membaca sebagian data (lebih menyeluruh)
$ sudo -E restic check --read-data-subset 10%
```

### Retention Policy

Tanpa pembatasan, jumlah snapshot bakal terus nambah. Restic misahin dua operasi. `forget` menandai snapshot lama buat dibuang, sedangkan `prune` benar-benar menghapus blok yang sudah tidak direferensikan.

```bash
# Simpan 7 harian, 4 mingguan, 6 bulanan, lalu hapus sisanya
$ sudo -E restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --prune
```

Kebijakan di atas nyimpen snapshot harian selama seminggu terakhir, mingguan selama sebulan, dan bulanan selama setengah tahun. Sisanya dibuang dalam satu perintah.

### Otomasi dengan Systemd Timer

Semua langkah di atas masih manual. Biar otomatis, saya bungkus dalam systemd service dan timer, pakai pola yang sama seperti di post systemd timer sebelumnya. Pertama, service unit yang menjalankan backup sekaligus retention:

```bash
$ sudo nano /etc/systemd/system/restic-backup.service
```

```ini
[Unit]
Description=Restic Backup to Cloud
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/restic/restic.env
ExecStart=/usr/bin/restic backup /opt/myapp/log
ExecStartPost=/usr/bin/restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

Lalu timer yang menjalankan service itu tiap hari jam 03:00:

```bash
$ sudo nano /etc/systemd/system/restic-backup.timer
```

```ini
[Unit]
Description=Run restic backup daily at 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=600

[Install]
WantedBy=timers.target
```

Aktifkan timer dan uji service-nya manual sekali sebelum mengandalkan jadwal otomatis:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now restic-backup.timer

# Uji manual, lalu cek lognya
$ sudo systemctl start restic-backup.service
$ sudo journalctl -u restic-backup.service -f
```

## Tantangan yang Dihadapi

Tantangan pertama dan paling bikin deg-degan adalah **manajemen password repository**. Kehilangan password berarti kehilangan seluruh backup secara permanen, tidak ada mekanisme recovery. Awalnya saya cuma menyimpan password itu di file di server yang sama. Baru ketahuan kemudian, kalau server hilang total, password-nya ikut hilang dan backup di cloud jadi tumpukan blob yang tidak bisa dibuka. Solusinya, saya simpan salinan password di password manager yang terpisah dari server. Backup terenkripsi tidak ada gunanya kalau kuncinya ikut lenyap bareng data aslinya.

Masalah kedua datang dari **operasi prune yang berat dan ngunci repository**. Prune membaca metadata seluruh repository, ngitung ulang blok yang masih terpakai, lalu menghapus yang tidak. Di repository besar, proses ini makan waktu lama dan ngasih lock, jadi backup lain tidak bisa jalan barengan. Saya sempat kena backup harian yang gagal karena bentrok dengan prune yang masih jalan. Solusinya, prune tidak saya jalankan tiap hari. Backup harian cuma melakukan `forget` tanpa `--prune`, dan prune yang sesungguhnya dijadwalkan terpisah seminggu sekali di jam yang benar-benar sepi.

Tantangan ketiga soal **bandwidth pas backup pertama**. Snapshot awal harus ngunggah seluruh data. Di koneksi terbatas, ini bisa memakan bandwidth server sampai ganggu layanan lain. Rclone punya opsi throttling yang bisa diteruskan ke restic lewat `--limit-upload`. Dengan ngebatasi laju unggah, backup pertama memang jadi lebih lama, tapi server tetap responsif melayani trafik normal. Setelah snapshot awal beres, backup berikutnya jauh lebih ringan karena cuma ngirim perubahan.

Terakhir, saya belajar pentingnya **misahin verifikasi dari backup**. Perintah `restic check` yang membaca data secara menyeluruh itu cukup mahal dan tidak perlu jalan tiap backup. Saya jadwalin terpisah, misalnya `check --read-data-subset` seminggu sekali, jadi integritas repository tetap terpantau tanpa mbebanin backup harian. Backup yang tercatat sukses bukan jaminan datanya bisa dibaca. Verifikasi berkala inilah yang ngasih keyakinan itu.

## Insight dan Pembelajaran

- **Arsip ngasih log memori jangka panjang**: rotasi lokal ngejaga disk tetap lega, tapi cuma arsip di lokasi terpisah yang bikin log lama tetap bisa diakses pas dibutuhkan. Keduanya saling melengkapi, bukan gantiin satu sama lain
- **Password itu bagian dari backup, bukan sekadar detail**: simpan password repository di tempat yang terpisah dari server. Enkripsi tanpa akses ke kunci sama saja dengan kehilangan data
- **Pisahkan operasi mahal dari operasi rutin**: backup harian harus ringan dan cepat. Operasi berat seperti `prune` dan `check --read-data` dijadwalkan terpisah di jam sepi biar tidak bentrok
- **Deduplikasi bikin hemat storage**: restic cuma menyimpan blok yang benar-benar baru, jadi menahan puluhan snapshot jauh lebih hemat storage dari yang dibayangkan. Efeknya, retention policy yang panjang jadi realistis buat dipakai
- **Backup baru terbukti setelah berhasil direstore**: sesekali sempatkan restore sungguhan ke direktori terpisah, lalu bandingkan hasilnya dengan sumber aslinya.
- **Rclone bikin pilihan backend jadi banyak**: begitu rclone dipasang sebagai jembatan, strategi backup tidak terkunci di satu provider. Mau pindah dari satu cloud ke cloud lain tinggal ganti remotenya, sisanya jalan seperti biasa

## Penutup

Kombinasi restic dan rclone ngasih fondasi backup yang lengkap. Terenkripsi sejak di server, hemat berkat deduplikasi, fleksibel karena bisa nulis ke hampir semua cloud, dan otomatis lewat systemd timer. Sisanya tinggal disiplin di dua kebiasaan: misahin operasi berat dari backup rutin, dan menguji restore secara berkala. Arsip log baru benar-benar bernilai pas log aslinya sudah dihapus rotasi. Di momen itu, semua persiapan tadi terbayar lunas.

## Referensi

- [Restic Documentation](https://restic.readthedocs.io/en/stable/), diakses pada 2026-08-08
- [Preparing a new repository - Restic](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html), diakses pada 2026-08-08
- [Removing backup snapshots - Restic](https://restic.readthedocs.io/en/stable/060_forget.html), diakses pada 2026-08-08
- [Rclone Documentation](https://rclone.org/docs/), diakses pada 2026-08-08
