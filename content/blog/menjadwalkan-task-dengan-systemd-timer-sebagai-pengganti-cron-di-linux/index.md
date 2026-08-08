+++
draft = false
date = '2026-08-08'
title = 'Menjadwalkan Task dengan Systemd Timer sebagai Pengganti Cron di Linux'
type = 'blog'
description = 'Cara mengganti cron dengan systemd timer untuk penjadwalan task yang lebih andal, dengan log terpusat, dependency management, dan fitur catch-up yang tidak dimiliki cron'
image = ''
tags = ['systemd', 'linux', 'cron', 'automation', 'scheduling']
+++

## Latar Belakang

Setelah aplikasi berjalan rapi sebagai systemd service, kebutuhan berikutnya biasanya soal task berkala. Bukan proses yang hidup terus, tapi yang jalan sebentar lalu berhenti. Backup database tiap malam. Bersihkan log lama tiap minggu. Sinkronisasi data tiap jam. Health check tiap beberapa menit. Task seperti ini tidak butuh proses yang nyala terus. Cukup dieksekusi di waktu tertentu, selesai, berhenti.

Selama ini saya selalu sama. Buka `crontab -e`, tulis satu baris jadwal, selesai. Cron memang simpel dan ada di mana-mana. Tapi begitu task-nya mulai penting, misalnya backup produksi, keterbatasan cron mulai kerasa. Ada task yang gagal diam-diam, dan saya baru sadar berhari-hari kemudian. Server sempat mati pas jam backup, task-nya terlewat begitu saja tanpa ada yang mengulang. Script yang mulus di shell malah gagal pas dipanggil cron karena environment-nya beda. Di titik itu saya sadar butuh penjadwalan yang lebih bisa diandalkan: lognya jelas dan gampang ditelusuri, bisa catch-up kalau server sempat mati, dan sadar sama dependency antar service supaya task tidak jalan sebelum yang dibutuhkannya siap.

## Permasalahan

Beberapa keterbatasan cron yang langsung kerasa saat saya mengelola task terjadwal di server produksi:

- **Log tidak jelas**: output cron dikirim ke mail lokal yang hampir tidak pernah saya buka. Kalau task gagal, tidak ada notifikasi apa-apa. Kegagalannya baru ketahuan setelah dampaknya kerasa
- **Tidak ada catch-up saat server mati**: kalau server mati pas jadwal task tiba, cron tidak mengulang setelah server nyala. Task-nya hilang begitu saja
- **Sulit di-debug**: tidak ada cara langsung untuk lihat kapan task terakhir jalan, berapa lama durasinya, atau apa exit code-nya. Semua harus dikorek manual dari file log
- **Tidak ada dependency**: cron tidak peduli apakah service yang dibutuhkan sudah siap. Task backup bisa jalan sebelum database benar-benar naik selepas reboot
- **Environment minim**: cron jalan dengan environment yang sangat terbatas. seperti script jalan di shell tapi gagal pas dipanggil cron, karena `PATH` dan variabelnya berbeda

## Pendekatan Solusi

Ada beberapa opsi untuk menjalankan task terjadwal di Linux:

| Pendekatan          | Kelebihan                                                                    | Kekurangan                                                          |
| ------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Cron**            | Simpel, universal, cukup satu baris jadwal                                   | Log nyangkut di mail lokal, tidak ada catch-up, environment minim  |
| **Cron + anacron**  | Menambal catch-up untuk task harian yang terlewat                           | Cuma granularitas harian, dan tetap terpisah dari ekosistem systemd |
| **Systemd timer**   | Log terpusat di journald, catch-up lewat `Persistent`, sadar dependency, bisa resource limit | Butuh dua file (`.timer` dan `.service`) dengan sintaks lebih verbose |

Saya pilih **systemd timer** karena:

1. **Log terpusat**: output task masuk ke `journald`, jadi bisa ditelusuri lewat `journalctl` sama seperti service lain. Tidak ada lagi log yang nyangkut di mail lokal
2. **Catch-up otomatis**: dengan `Persistent=true`, task yang terlewat saat server mati langsung dijalankan begitu server nyala lagi. Ini krusial buat backup
3. **Reuse konsep yang sudah dipahami**: timer cuma memicu sebuah service unit, jadi semua yang sudah saya pelajari tentang `User`, `EnvironmentFile`, dan resource limit langsung kepakai. Kalau belum familiar dengan unit file systemd, ada baiknya baca dulu tulisan saya tentang [menjalankan aplikasi sebagai systemd service](/blog/menjalankan-aplikasi-sebagai-systemd-service-di-linux/)

## Implementasi Teknis

Sebagai contoh, saya bikin task backup database PostgreSQL yang jalan tiap hari jam 02:00 pagi. Pola yang sama kepakai untuk task terjadwal apapun.

Konsep kuncinya: systemd memisahkan **apa yang dikerjakan** (didefinisikan di `.service`) dari **kapan dikerjakan** (didefinisikan di `.timer`). Kedua file punya nama dasar yang sama supaya systemd otomatis memasangkannya.

### Membuat Service Unit

Pertama, buat service unit yang berisi task-nya. Beda dengan service aplikasi yang jalan terus, task terjadwal pakai `Type=oneshot`. Tipe ini bilang ke systemd bahwa proses bakal jalan sampai selesai lalu berhenti. Itu memang perilaku yang diharapkan, bukan tanda gagal.

```bash
$ sudo nano /etc/systemd/system/backup-db.service
```

Isi dengan konfigurasi berikut:

```ini
[Unit]
Description=Daily PostgreSQL Backup
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=oneshot
User=postgres
ExecStart=/opt/scripts/backup-db.sh
```

Perhatikan service ini **tidak punya section `[Install]`**. Itu disengaja. Service ini tidak jalan saat boot dan tidak di-enable langsung. Yang memicunya timer. Direktif `After=postgresql.service` juga memastikan backup tidak jalan sebelum database siap, sesuatu yang tidak bisa dilakukan cron.

### Membuat Timer Unit

Selanjutnya, buat timer unit dengan nama dasar yang sama (`backup-db`), hanya berbeda ekstensi.

```bash
$ sudo nano /etc/systemd/system/backup-db.timer
```

Isi dengan konfigurasi berikut:

```ini
[Unit]
Description=Run PostgreSQL backup daily at 02:00

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

Penjelasan tiap direktif di section `[Timer]`:

| Direktif             | Fungsi                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------- |
| `OnCalendar`         | jadwal absolut kapan timer memicu service, memakai format kalender systemd                    |
| `Persistent=true`    | menjalankan task yang terlewat begitu server nyala kembali, keunggulan utama atas cron        |
| `RandomizedDelaySec` | menambah jeda acak (di sini hingga 300 detik) untuk mencegah banyak timer jalan serentak      |

Section `[Install]` di sini pakai `WantedBy=timers.target`, bukan `multi-user.target` seperti service biasa. Target ini tempat ngumpulnya semua timer yang aktif saat boot.

### Memahami Format OnCalendar

Format `OnCalendar` lebih ekspresif dibanding sintaks cron. Berikut beberapa pola yang sering dipakai:

| Ekspresi                | Arti                                        |
| ----------------------- | ------------------------------------------- |
| `daily`                 | setiap hari jam 00:00                        |
| `weekly`                | setiap Senin jam 00:00                       |
| `*-*-* 02:00:00`        | setiap hari jam 02:00                        |
| `Mon *-*-* 09:00:00`    | setiap Senin jam 09:00                       |
| `*-*-01 00:00:00`       | setiap tanggal 1 tiap bulan                  |
| `*:0/15`                | setiap 15 menit                              |
| `Mon..Fri *-*-* 18:00`  | setiap hari kerja jam 18:00                  |

Untuk memastikan ekspresi benar dan lihat kapan ia memicu berikutnya, pakai `systemd-analyze`:

```bash
$ systemd-analyze calendar "*-*-* 02:00:00"
```

Outputnya menampilkan normalisasi ekspresi dan waktu trigger berikutnya:

```
  Original form: *-*-* 02:00:00
Normalized form: *-*-* 02:00:00
    Next elapse: Sat 2026-08-09 02:00:00 WIB
       (in UTC): Fri 2026-08-08 19:00:00 UTC
       From now: 15h left
```

### Mengaktifkan Timer

Setelah kedua file siap, baca ulang konfigurasi systemd lalu aktifkan timer-nya. Yang di-enable adalah `.timer`, **bukan** `.service`.

```bash
# Baca ulang konfigurasi systemd
$ sudo systemctl daemon-reload

# Aktifkan dan langsung jalankan timer
$ sudo systemctl enable --now backup-db.timer
```

Kesalahan umum di sini: yang di-enable malah service-nya, bukan timer-nya. Kalau itu terjadi, service coba jalan saat boot, bukan sesuai jadwal.

### Verifikasi dan Monitoring

Perintah paling berguna untuk memantau semua timer adalah `list-timers`:

```bash
$ systemctl list-timers
```

Outputnya menampilkan jadwal berikutnya, sisa waktu, kapan terakhir jalan, dan berapa lama sejak itu:

```
NEXT                        LEFT      LAST                        PASSED   UNIT             ACTIVATES
Sat 2026-08-09 02:00:00 WIB 15h left  Fri 2026-08-08 02:00:00 WIB 9h ago   backup-db.timer  backup-db.service
```

Untuk memastikan logika task-nya benar tanpa nunggu jadwal, service bisa dipicu manual. Ini cara terbaik menguji script backup, lepas dari soal penjadwalan:

```bash
# Jalankan task sekarang juga, tanpa menunggu timer
$ sudo systemctl start backup-db.service

# Cek hasil eksekusinya lewat log
$ sudo journalctl -u backup-db.service -n 50
```

Karena output masuk ke journald, semua riwayat eksekusi bisa ditelusuri, termasuk yang gagal:

```bash
# Ikuti log secara real time
$ sudo journalctl -u backup-db.service -f

# Lihat eksekusi dalam 3 hari terakhir
$ sudo journalctl -u backup-db.service --since "3 days ago"
```

### Alternatif: Timer Relatif

Selain jadwal kalender absolut, timer juga bisa pakai jadwal relatif. Ini berguna untuk task berinterval. Alih-alih "jam 02:00", timer bisa diatur "6 jam sejak terakhir jalan".

```ini
[Timer]
OnBootSec=15min
OnUnitActiveSec=6h
```

`OnBootSec=15min` menjalankan task 15 menit setelah boot. `OnUnitActiveSec=6h` menjalankannya lagi tiap 6 jam sejak eksekusi terakhir. Pola ini cocok untuk task yang penting jaraknya konsisten, bukan waktu absolutnya, misalnya sinkronisasi data atau pembersihan cache. Untuk backup yang harus jalan di jam sepi, `OnCalendar` tetap lebih pas.

## Tantangan yang Dihadapi

Tantangan pertama datang dari **service yang gagal senyap padahal timer aktif**. Awalnya saya kira semua beres karena `systemctl list-timers` menampilkan timer aktif dengan jadwal berikutnya. Ternyata service-nya gagal tiap kali dipicu karena script backup ada bug. Pelajarannya, status timer yang sehat tidak menjamin task-nya sukses. Timer dan service itu dua hal terpisah. Sejak itu saya selalu uji service manual dengan `systemctl start` dulu sampai benar-benar sukses, baru andalkan timer untuk penjadwalannya.

Masalah kedua justru datang dari fitur andalannya sendiri, **`Persistent=true` yang memicu backup ganda**. Server saya mati cukup lama karena maintenance. Begitu nyala lagi, systemd langsung menjalankan backup yang terlewat. Barengan dengan itu jadwal normal jam 02:00 juga tiba. Alhasil dua proses backup jalan hampir bersamaan dan rebutan resource database. Untuk task mahal seperti backup, saya menambahkan lock file di dalam script supaya tidak ada dua instance yang jalan sekaligus. `Persistent` sangat berguna, tapi harus diimbangi kesadaran bahwa task bisa jalan di waktu yang tidak terduga.

Tantangan ketiga soal **environment yang beda**, masalah yang sama persis dengan cron. Script backup saya memanggil `pg_dump` tanpa path absolut dan mengandalkan `PATH` dari shell. Pas dijalankan lewat systemd, `PATH`-nya minimal jadi perintahnya tidak ketemu. Solusinya, saya pakai path absolut di dalam script, atau definisikan environment yang dibutuhkan lewat `EnvironmentFile` di service unit. Jangan pernah berasumsi environment systemd sama dengan environment interaktif.

Terakhir, saya sadar pentingnya **`RandomizedDelaySec` untuk menghindari thundering herd**. Waktu beberapa server saya diatur menjalankan backup dan sinkronisasi tepat jam 02:00, semuanya membebani storage dan jaringan di saat yang sama. Dengan `RandomizedDelaySec=300`, tiap timer memicu di titik acak dalam rentang 5 menit, jadi bebannya tersebar. Untuk satu server ini mungkin tidak kerasa. Tapi begitu jumlah server nambah, jeda acak ini menyelamatkan sistem dari lonjakan beban serentak.

## Insight dan Pembelajaran

- **Pisahkan "apa yang dikerjakan" dari "kapan dikerjakan"**: `.service` mendefinisikan aksi, `.timer` mendefinisikan jadwal. Pemisahan ini bikin task bisa dipicu manual buat testing tanpa ganggu jadwalnya, sesuatu yang mustahil dilakukan dengan satu baris cron
- **`Persistent=true` adalah keunggulan nyata atas cron**: task yang terlewat karena server mati tetap dijalankan. Untuk backup dan task kritikal, fitur ini saja sudah cukup jadi alasan pindah dari cron
- **`systemctl list-timers` adalah dashboard penjadwalan**: satu perintah menampilkan semua jadwal, kapan berikutnya, dan kapan terakhir jalan. Tidak perlu lagi nebak-nebak isi crontab yang tersebar
- **Selalu uji service secara terpisah dulu**: picu `.service` manual sampai sukses sebelum andalkan `.timer`. Langkah ini memisahkan masalah logika task dari masalah penjadwalan, jadi debugging lebih cepat
- **Environment systemd bukan environment shell**: selalu pakai path absolut atau `EnvironmentFile`. Asumsi keduanya sama adalah sumber bug yang paling sering muncul
- **Cron masih relevan untuk hal simpel**: tidak semua penjadwalan perlu dipindah. Untuk task sepele di mesin pribadi, cron sudah cukup. Systemd timer menang saat butuh log, catch-up, atau dependency

## Penutup

Systemd timer bukan sekadar pengganti cron. Ini upgrade dalam hal keandalan dan observability. Dengan memisahkan definisi task dari jadwalnya, menambahkan catch-up otomatis, dan mengalirkan semua log ke journald, penjadwalan task berubah dari yang tadinya "set and forget lalu berdoa" jadi sesuatu yang bisa dipantau dan di-debug dengan percaya diri. Untuk task terjadwal yang sensitif, langkah lanjutannya menggabungkan timer dengan fitur pengaman service seperti resource limit lewat `MemoryMax` dan sandboxing lewat `ProtectSystem`, supaya task jalan terisolasi dan tidak bisa merusak sistem meski ada yang tidak beres.

Sebagai rangkuman cepat, tiga perintah inti untuk mengelola timer:

```bash
$ sudo systemctl enable --now backup-db.timer   # aktifkan timer
$ systemctl list-timers                          # lihat semua jadwal
$ sudo systemctl start backup-db.service         # picu task manual untuk testing
```

Cron menjalankan task lalu melupakannya. Systemd timer menjalankan task, mencatatnya, dan mengingatkan kapan ia terakhir bekerja.

## Referensi

- [systemd.timer - Timer unit configuration](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html), diakses pada 2026-08-08
- [systemd.time - Time and date specifications](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html), diakses pada 2026-08-08
- [systemd/Timers - ArchWiki](https://wiki.archlinux.org/title/Systemd/Timers), diakses pada 2026-08-08
- [systemd-analyze - Analyze system manager](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html), diakses pada 2026-08-08
