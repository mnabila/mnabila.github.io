+++
draft = false
date = '2026-08-08'
title = 'Sinkronisasi Docker Volume antar Server dengan tar dan rsync di Ubuntu Server 22.04'
type = 'blog'
description = 'Cara memindahkan dan menyinkronkan named volume Docker antar dua server yang sama-sama berbasis container tanpa merusak permission dan ownership data'
image = ''
tags = ['docker', 'volume', 'rsync', 'ubuntu']
+++

## Latar Belakang

Saya menjalankan dua server yang keduanya berbasis docker sebagai infrastrukturnya. Server lama sudah waktunya dipensiunkan dan saya menyiapkan server baru dengan spesifikasi lebih besar sebagai penggantinya. Hampir semua service di sana hidup sebagai container, dan datanya tersimpan rapi di named volume.

Yang bikin repot adalah ketika akan memindahkan data itu ke server baru. Aplikasi yang stateless gampang dipindahkan tinggal tarik image dan `docker compose up`. Tapi volume berisi database, file upload user, dan konfigurasi yang tidak boleh hilang sedikitpun, jadi tidak bisa asal dimigrasikan.

Saya juga tidak bisa langsung matikan server lama begitu saja. Selama masa transisi keduanya harus tetap jalan bersamaan, sehingga selain migrasi sekali jalan saya butuh cara menyinkronkan volume secara berkala.

## Permasalahan

- **Named volume tidak bisa langsung di-copy**, isinya ada di `/var/lib/docker/volumes/<nama>/_data` yang butuh akses root dan tidak intuitif untuk dipindah manual.
- **Permission dan ownership gampang rusak**, kalau di-copy pakai tool biasa, UID/GID file bisa berubah dan bikin Postgres atau aplikasi tidak akan jalan.
- **Data tidak konsisten kalau container masih menulis**, menyalin volume database yang aktif berisiko menghasilkan snapshot yang korup.
- **Downtime kalau semua di-copy sekaligus**, volume besar butuh waktu transfer lama, dan service tidak bisa mati selama itu.
- **UID antar host bisa berbeda**, user `postgres` di server lama belum tentu punya UID yang sama di server baru, sehingga file jadi tidak terbaca.

## Pendekatan Solusi

Ada beberapa cara memindahkan isi volume antar server. Saya bandingkan dulu sebelum memilih.

| Pendekatan                                    | Kelebihan                                                                                     | Kekurangan                                                                  |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `docker cp` lalu `scp` arsip                  | Cara paling familiar dan tidak menyentuh internal Docker sama sekali                          | Harus bikin arsip dulu jadi butuh disk ekstra, dan langkahnya kepisah-pisah |
| Helper container `tar` streaming via SSH      | Portable karena cuma butuh image `alpine`, dan datanya langsung mengalir tanpa file perantara | Perlu setup SSH tanpa password antar host dulu                              |
| `rsync` langsung ke `/var/lib/docker/volumes` | Mudah untuk perubahan kecil karena cuma kirim delta                                           | Harus jadi root dan gampang salah path, rasanya seperti mengakali Docker    |
| Shared storage NFS                            | Kedua server lihat data yang sama secara real-time                                            | Nambah komponen baru yang bisa menjadi masalah baru, dan ada latency jaringan   |
| Continuous sync (Syncthing/lsyncd)            | Sekali diatur langsung sinkron otomatis terus-menerus                                         | Terlalu berat untuk sekadar pindahan satu kali                              |

Untuk **migrasi satu kali**, saya pakai helper container yang membungkus isi volume dengan `tar` lalu mengalirkannya langsung ke server tujuan lewat SSH. Pendekatan ini tidak menyentuh path internal Docker, tidak butuh disk sementara, dan portable karena hanya mengandalkan image `alpine`.

Untuk **sinkronisasi berkala** selama masa transisi, saya pakai `rsync` yang juga dijalankan lewat helper container. Alasannya, `rsync` cuma mengirim delta sehingga sinkronisasi kedua dan seterusnya jadi jauh lebih cepat.

Kunci dari kedua pendekatan ini sama: jangan pernah menyentuh `/var/lib/docker/volumes` secara langsung. Selalu mount volume ke dalam container helper, biar Docker yang urus path-nya.

## Implementasi Teknis

### Persiapan Akses SSH antar Server

Streaming antar host butuh SSH tanpa password. Dari server lama (`10.10.10.10`), buat key lalu kirim ke server baru (`10.10.10.11`).

```bash
$ ssh-keygen -t ed25519 -C "volume-sync" -f ~/.ssh/volsync
$ ssh-copy-id -i ~/.ssh/volsync.pub deploy@10.10.10.11
```

Uji koneksinya biar tahu tidak ada prompt yang mengganggu saat streaming.

```bash
$ ssh -i ~/.ssh/volsync deploy@10.10.10.11 'echo ok'
ok
```

Setelah ini kedua server sudah bisa saling kirim data tanpa interaksi manual.

### Identifikasi Volume yang Akan Dipindah

Lihat dulu volume apa saja yang ada dan mana yang menyimpan data penting.

```bash
$ docker volume ls
DRIVER    VOLUME NAME
local     app_pgdata
local     app_uploads
```

Untuk memastikan ukuran datanya, inspect salah satu volume dan cek isinya lewat container sementara.

```bash
$ docker run --rm -v app_pgdata:/data alpine du -sh /data
1.2G    /data
```

Sekarang saya tahu volume `app_pgdata` dan `app_uploads` yang perlu ikut pindah.

### Hentikan Container yang Menulis ke Volume

Untuk volume database, konsistensi lebih penting daripada uptime sesaat. Hentikan container yang memakai volume tersebut sebelum menyalin.

```bash
$ docker compose stop app postgres
```

> **Penting:** Menyalin volume database yang masih aktif berisiko menghasilkan data korup. Selalu hentikan container database dulu, atau gunakan `pg_dump` untuk volume Postgres kalau downtime tidak bisa ditolerir.

### Streaming Volume Langsung antar Server

Ini inti prosesnya. Jalankan helper container di server lama untuk membungkus isi volume dengan `tar`, lalu pipe hasilnya lewat SSH ke helper container di server baru yang mengekstraknya.

```bash
$ docker run --rm -v app_pgdata:/data alpine \
    tar czf - --numeric-owner -C /data . \
  | ssh -i ~/.ssh/volsync deploy@10.10.10.11 \
    'docker run --rm -i -v app_pgdata:/data alpine \
       tar xzf - --numeric-owner -C /data'
```

Flag `--numeric-owner` di sini yang menyelamatkan permission. Tanpa itu, `tar` akan menerjemahkan UID ke nama user berdasarkan `/etc/passwd` masing-masing host, dan ownership file bisa kacau di server tujuan.

Volume `app_pgdata` di server baru harus sudah ada sebelum extract. Kalau belum, buat dulu dengan `docker volume create app_pgdata` di server tujuan.

### Verifikasi Hasil Transfer

Bandingkan ukuran dan jumlah file di kedua sisi untuk memastikan tidak ada yang tertinggal.

```bash
$ ssh -i ~/.ssh/volsync deploy@10.10.10.11 \
    'docker run --rm -v app_pgdata:/data alpine sh -c "du -sh /data && ls /data | wc -l"'
1.2G    /data
28
```

Angka yang cocok dengan server asal menandakan transfer utuh. Setelah itu, jalankan container di server baru dan cek log-nya untuk memastikan aplikasi membaca data dengan benar.

### Sinkronisasi Incremental dengan rsync

Untuk volume yang masih terus berubah selama transisi, misalnya `app_uploads`, transfer penuh setiap kali boros. Pakai `rsync` yang hanya mengirim delta. Mount volume ke helper container, lalu jalankan `rsync` melalui SSH.

```bash
$ docker run --rm -v app_uploads:/data \
    -v ~/.ssh:/root/.ssh:ro \
    instrumentisto/rsync-ssh \
    rsync -az --numeric-ids --delete \
      -e "ssh -i /root/.ssh/volsync" \
      /data/ deploy@10.10.10.11:/mnt/uploads-staging/
```

Perlu diperhatikan, `rsync` di sini menulis ke direktori host `/mnt/uploads-staging`, bukan langsung ke volume tujuan. Dari sana baru saya sinkronkan ke volume target di server baru. Flag `--numeric-ids` menjaga ownership tetap berbasis UID numerik, sejalan dengan `--numeric-owner` pada `tar`.

Jalankan perintah ini berkala, misalnya lewat cron, dan hanya perubahan sejak sinkronisasi terakhir yang akan dikirim.

## Tantangan yang Dihadapi

Masalah pertama yang tidak langsung kelihatan adalah soal ownership. Di percobaan awal saya lupa memakai `--numeric-owner`, dan hasilnya Postgres di server baru menolak start dengan error permission pada direktori data. Ternyata `tar` menerjemahkan UID pemilik file ke nama user berdasarkan `/etc/passwd` host asal, lalu menerjemahkan balik ke UID yang berbeda di host tujuan karena urutan user-nya tidak sama. Memakai UID numerik langsung di kedua sisi menghilangkan masalah ini sepenuhnya.

Masalah kedua adalah konsistensi data. Saya sempat berpikir bisa menyalin volume Postgres tanpa menghentikan container biar tidak ada downtime. Hasilnya database di server baru sempat menolak jalan karena WAL yang setengah tertulis. Sejak itu aturan saya sederhana, volume database selalu di-copy dalam keadaan container mati, atau pakai `pg_dump` kalau memang tidak boleh ada downtime sama sekali. Untuk volume yang isinya file statis seperti upload, menyalin dalam keadaan hidup masih aman.

Yang terakhir soal flag `--delete` pada `rsync`. Flag ini menghapus file di tujuan yang sudah tidak ada di sumber, dan itu memang yang saya mau untuk mirror yang bersih. Tapi kalau arah sync sampai terbalik, satu perintah bisa menghapus data produksi. Saya selalu memastikan arah sumber dan tujuan benar, dan menjalankan `rsync` dengan flag `--dry-run` dulu sebelum eksekusi sungguhan pada data penting.

## Insight dan Pembelajaran

- **Jangan pernah sentuh `/var/lib/docker/volumes` langsung**, selalu mount volume ke helper container biar Docker yang mengurus path dan lifecycle-nya.
- **UID numerik adalah teman**, `--numeric-owner` pada `tar` dan `--numeric-ids` pada `rsync` mencegah ownership berantakan saat user antar host tidak identik.
- **Streaming lewat pipe menghemat disk**, mengalirkan hasil `tar` langsung ke SSH tanpa file arsip perantara menghindari kebutuhan ruang penyimpanan dua kali lipat.
- **Konsistensi menentukan metode**, volume database butuh container mati atau dump logis, sedangkan volume file statis aman disalin dalam keadaan hidup.
- **rsync untuk perubahan, tar untuk pindahan pertama**, transfer penuh sekali di awal, lalu delta berkala jauh lebih murah untuk data yang masih aktif berubah.

## Penutup

Kunci memindahkan Docker volume antar server bukan pada tool yang canggih, melainkan pada disiplin menjaga ownership tetap numerik dan konsistensi data saat menyalin. Helper container `tar` yang di-stream lewat SSH menyelesaikan migrasi awal tanpa file perantara, sementara `rsync` menangani sinkronisasi berkala dengan biaya delta yang ringan. Selama path internal Docker tidak disentuh langsung, proses ini aman diulang berkali-kali sampai transisi antar server benar-benar selesai.

## Referensi

- [Docker Documentation: Back up, restore, or migrate data volumes](https://docs.docker.com/storage/volumes/#back-up-restore-or-migrate-data-volumes), Diakses pada 2026-08-08
- [GNU tar Manual: --numeric-owner](https://www.gnu.org/software/tar/manual/html_node/Attributes.html), Diakses pada 2026-08-08
- [rsync Manual Page](https://download.samba.org/pub/rsync/rsync.1), Diakses pada 2026-08-08
