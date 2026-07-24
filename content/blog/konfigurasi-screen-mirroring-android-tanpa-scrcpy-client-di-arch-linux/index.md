+++
draft = false
date = '2026-07-10'
title = 'Konfigurasi Screen Mirroring Android tanpa scrcpy Client di Arch Linux'
type = 'blog'
description = 'Cara mengonsumsi stream scrcpy server langsung dengan ffmpeg dan mpv untuk screen mirroring yang lebih ringan tanpa scrcpy client'
image = ''
tags = ['scrcpy', 'ffmpeg', 'mpv', 'android', 'archlinux']
+++

## Latar Belakang

Saya sering pakai **scrcpy** untuk screen mirroring Android ke Linux. Buat monitoring device, demo, atau sekadar mirroring layar ke monitor yang lebih besar, scrcpy jadi tool yang paling praktis.

Masalahnya, scrcpy client cukup rakus CPU. Untuk sesi screen mirroring singkat mungkin tidak terasa, tapi kalau dibiarkan jalan terus untuk live monitoring, CPU usage-nya bisa cukup mengganggu. Padahal yang saya butuhkan cuma menampilkan layar Android, tanpa perlu fitur input control atau recording.

Ternyata scrcpy itu terdiri dari dua komponen terpisah: **server** yang jalan di Android dan **client** yang jalan di PC. Server menangkap layar dan encode ke H.264, lalu mengirim stream lewat socket. Client menerima stream, decode, dan render ke window. Yang berat itu di sisi client, karena proses decoding-nya menggunakan software decoder.

## Permasalahan

- **CPU usage scrcpy client tinggi**: scrcpy client melakukan software decoding H.264 yang bisa memakan 15-30% CPU tergantung resolusi dan framerate. Untuk live monitoring yang jalan terus-menerus, ini cukup boros
- **Hardware acceleration tidak optimal di scrcpy client**: scrcpy memang support beberapa renderer, tapi konfigurasinya tidak selalu smooth tergantung GPU dan driver yang dipakai
- **Fitur yang tidak dibutuhkan**: untuk live monitoring, fitur input control, clipboard sync, dan file transfer dari scrcpy client tidak diperlukan. Overhead-nya percuma
- **mpv sudah punya hardware decoding yang mature**: **mpv** dengan **VAAPI** atau **VDPAU** bisa decode H.264 dengan CPU usage minimal. Tinggal cari cara pipe stream dari scrcpy server ke mpv

## Pendekatan Solusi

| Pendekatan | Kelebihan | Kekurangan |
| --- | --- | --- |
| scrcpy client (default) | Satu command, lengkap dengan input control | CPU tinggi karena software decoding |
| scrcpy + `--v4l2-sink` | Bisa pakai viewer eksternal | Masih butuh scrcpy client, perlu setup `v4l2loopback` |
| `adb screenrecord` + pipe | Paling simpel | Limit 3 menit per sesi, tidak bisa continuous |
| scrcpy-server + ffmpeg + mpv | CPU rendah dengan hardware decoding, ringan | Tidak ada input control, butuh setup manual |

Saya pilih **scrcpy-server + ffmpeg + mpv** karena tujuan utamanya pure monitoring tanpa input. Dengan mpv yang sudah support VAAPI, hardware decoding H.264 bisa dimanfaatkan langsung dan CPU usage turun drastis.

Kuncinya ada di flag `send_frame_meta=false` pada scrcpy server. Flag ini menghilangkan metadata per-frame (PTS timestamp + packet size) dari stream output, sehingga yang keluar adalah raw H.264 NAL units yang bisa langsung di-parse oleh **FFmpeg**.

## Implementasi Teknis

### Instalasi Dependency

Di Arch Linux, semua package yang dibutuhkan ada di official repository:

```bash
$ sudo pacman -S scrcpy ffmpeg mpv
```

Package `scrcpy` sudah include server JAR di `/usr/share/scrcpy/scrcpy-server`. Tidak perlu download terpisah.

Pastikan juga `adb` sudah terinstall dan device Android terhubung:

```bash
$ adb devices
```

```
List of devices attached
XXXXXXXXX	device
```

### Push scrcpy Server ke Device

scrcpy server perlu di-push ke device Android sebelum bisa dijalankan:

```bash
$ adb push /usr/share/scrcpy/scrcpy-server /data/local/tmp/scrcpy-server.jar
```

File ini akan tersimpan di `/data/local/tmp/` pada device. Lokasi ini dipilih karena accessible tanpa root.

### Jalankan scrcpy Server

Start server dengan flag yang menghilangkan semua metadata dari stream output:

```bash
$ adb shell CLASSPATH=/data/local/tmp/scrcpy-server.jar \
    app_process / com.genymobile.scrcpy.Server 2.7 \
    tunnel_forward=true \
    video=true \
    audio=false \
    control=false \
    send_device_meta=false \
    send_dummy_byte=false \
    send_codec_meta=false \
    send_frame_meta=false \
    video_codec=h264 \
    max_size=1920
```

Penjelasan flag-flag penting:

| Flag | Fungsi |
| --- | --- |
| `tunnel_forward=true` | Server listen di local socket, client yang connect |
| `audio=false` | Matikan audio stream |
| `control=false` | Matikan input control |
| `send_device_meta=false` | Jangan kirim device info di awal connection |
| `send_dummy_byte=false` | Jangan kirim dummy byte di awal |
| `send_codec_meta=false` | Jangan kirim codec metadata |
| `send_frame_meta=false` | Jangan kirim PTS dan size per frame, output jadi raw H.264 |
| `max_size=1920` | Batasi resolusi maksimal untuk mengurangi bandwidth |

> **Penting:** Versi server (`2.7` di contoh) harus sesuai dengan versi scrcpy yang terinstall. Cek dengan `scrcpy --version`.

### Setup Port Forwarding

Forward port dari device ke localhost supaya bisa diakses dari PC:

```bash
$ adb forward tcp:27183 localabstract:scrcpy
```

Port `27183` adalah port default scrcpy. Setelah ini, stream bisa diakses di `tcp://localhost:27183`.

### Connect dengan FFmpeg dan mpv

Pipe stream dari socket ke mpv melalui ffmpeg:

```bash
$ ffmpeg -probesize 32 -analyzeduration 0 \
    -f h264 -i tcp://localhost:27183 \
    -f mpegts -codec copy - | \
    mpv --no-cache --untimed --profile=low-latency -
```

Penjelasan parameter:

| Parameter | Fungsi |
| --- | --- |
| `-probesize 32` | Minimal probing, percepat start |
| `-analyzeduration 0` | Skip stream analysis, langsung play |
| `-f h264` | Input format raw H.264 |
| `-codec copy` | Tidak re-encode, langsung copy stream |
| `-f mpegts` | Container format untuk piping ke mpv |
| `--profile=low-latency` | Profile mpv untuk minimal buffering |
| `--untimed` | mpv tidak tunggu timing, play secepat data masuk |
| `--no-cache` | Matikan cache untuk latency minimal |

mpv akan otomatis menggunakan hardware decoding (VAAPI/VDPAU) kalau tersedia. Ini yang bikin CPU usage jauh lebih rendah dibanding scrcpy client.

### Script Wrapper

Supaya tidak perlu jalankan command satu per satu, bungkus semuanya dalam satu script:

```bash
#!/bin/bash

SCRCPY_SERVER="/usr/share/scrcpy/scrcpy-server"
SCRCPY_VERSION=$(scrcpy --version 2>&1 | head -1 | grep -oP '\d+\.\d+')
PORT=27183

cleanup() {
    adb forward --remove tcp:$PORT 2>/dev/null
    kill $SERVER_PID 2>/dev/null
    exit 0
}

trap cleanup EXIT INT TERM

# Push server
adb push "$SCRCPY_SERVER" /data/local/tmp/scrcpy-server.jar

# Start server
adb shell CLASSPATH=/data/local/tmp/scrcpy-server.jar \
    app_process / com.genymobile.scrcpy.Server "$SCRCPY_VERSION" \
    tunnel_forward=true \
    video=true \
    audio=false \
    control=false \
    send_device_meta=false \
    send_dummy_byte=false \
    send_codec_meta=false \
    send_frame_meta=false \
    video_codec=h264 \
    max_size=1920 &
SERVER_PID=$!

sleep 2

# Forward port
adb forward tcp:$PORT localabstract:scrcpy

# Connect and play
ffmpeg -probesize 32 -analyzeduration 0 \
    -f h264 -i tcp://localhost:$PORT \
    -f mpegts -codec copy - | \
    mpv --no-cache --untimed --profile=low-latency -
```

Simpan sebagai `mirror.sh`, lalu jalankan:

```bash
$ chmod +x mirror.sh
$ ./mirror.sh
```

Script ini juga handle cleanup otomatis saat di-interrupt dengan Ctrl+C. Port forwarding dihapus dan server process di-kill.

## Tantangan yang Dihadapi

Tantangan pertama yang cukup membingungkan adalah **timing connection**. Server butuh waktu beberapa detik untuk start di device. Kalau ffmpeg mencoba connect sebelum server siap, connection refused dan harus restart semuanya. Solusinya cukup simpel: tambahkan `sleep 2` setelah start server. Tidak elegan, tapi reliable.

Tantangan kedua adalah soal **versi scrcpy server**. Flag-flag seperti `send_frame_meta` dan `send_codec_meta` baru ada di scrcpy versi 2.x ke atas. Kalau pakai versi lama, flag ini tidak dikenali dan server gagal start tanpa error message yang jelas. Pastikan versi yang di-pass ke server command sama persis dengan versi binary scrcpy yang terinstall.

Yang juga perlu diperhatikan: kalau device masuk sleep mode atau layar mati, stream berhenti dan ffmpeg/mpv ikut exit. Untuk live monitoring yang benar-benar continuous, perlu tambahkan `stay_awake=true` dan `power_off_on_close=false` di flag server, plus logic restart di script wrapper kalau connection terputus.

## Insight dan Pembelajaran

- **scrcpy server dan client itu independen**: server hanya capture + encode, client handle decode + render. Memahami pemisahan ini membuka kemungkinan untuk ganti client dengan tool yang lebih sesuai kebutuhan
- **`send_frame_meta=false` adalah kunci**: tanpa flag ini, output stream punya header per-frame (PTS + packet size) yang bikin ffmpeg tidak bisa parse langsung sebagai raw H.264
- **Hardware decoding di mpv itu game changer**: CPU usage turun dari 15-30% (scrcpy client) ke 2-5% (mpv dengan VAAPI). Perbedaannya sangat signifikan untuk sesi monitoring yang berjalan lama
- **Trade-off yang jelas**: pendekatan ini mengorbankan fitur input control dan kemudahan setup demi efisiensi resource. Cocok untuk monitoring, tidak cocok kalau butuh interaksi dengan device

## Penutup

Mengonsumsi scrcpy server langsung dengan ffmpeg dan mpv adalah solusi yang efektif untuk screen mirroring ringan di Linux. Pendekatan ini memanfaatkan hardware decoding mpv yang sudah mature dan menghilangkan overhead fitur scrcpy client yang tidak dibutuhkan untuk skenario monitoring. Kalau tujuannya cuma menampilkan layar Android tanpa perlu kontrol input, cara ini jauh lebih hemat resource.

## Referensi

- [scrcpy - GitHub Repository](https://github.com/Genymobile/scrcpy), diakses pada 2026-07-10
- [scrcpy Developer Documentation](https://github.com/Genymobile/scrcpy/blob/master/doc/develop.md), diakses pada 2026-07-10
- [mpv Manual - Hardware Decoding](https://mpv.io/manual/master/#options-hwdec), diakses pada 2026-07-10
- [FFmpeg H.264 Raw Demuxer](https://ffmpeg.org/ffmpeg-formats.html#h264), diakses pada 2026-07-10
