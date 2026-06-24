+++
draft = false
date = '2026-05-30'
title = 'Konfigurasi v4l2loopback dan scrcpy untuk Virtual Webcam Android di Archlinux'
type = 'blog'
description = 'Cara mengubah layar Android menjadi device /dev/video di Linux menggunakan v4l2loopback dan scrcpy, berguna untuk computer vision, OBS, atau streaming tanpa capture card fisik.'
image = ''
tags = ['v4l2loopback', 'scrcpy', 'android', 'webcam', 'archlinux']
+++

## Latar Belakang

Saya sedang mengerjakan project computer vision yang butuh feed video dari kamera Android. Masalahnya, aplikasi seperti **OpenCV**, **OBS**, atau bahkan browser yang butuh akses webcam hanya bisa membaca device `/dev/video*`, mereka tidak bisa langsung mengambil stream dari layar Android yang terhubung via USB.

Opsi paling simple adalah beli capture card. Tapi untuk eksperimen dan prototyping, rasanya overkill mengeluarkan uang untuk hardware tambahan kalau bisa diselesaikan dengan software. Di Linux, ternyata ada cara untuk membuat virtual webcam device menggunakan **v4l2loopback**, sebuah kernel module yang membuat device `/dev/video*` virtual. Kombinasikan dengan **scrcpy** yang bisa mirror layar Android via ADB, dan hasilnya adalah layar Android masuk sebagai virtual webcam yang bisa dibaca oleh aplikasi apapun.

## Permasalahan

- **Android screen tidak dikenali sebagai webcam**: aplikasi seperti OpenCV, browser, atau OBS hanya bisa membaca device `/dev/video*`. Stream layar Android via USB tidak otomatis muncul sebagai video device di Linux
- **Butuh solusi tanpa hardware tambahan**: capture card fisik bisa menyelesaikan masalah, tapi tidak praktis untuk prototyping atau eksperimen cepat
- **Kompatibilitas dengan berbagai aplikasi**: solusi harus bisa dikenali oleh OpenCV, OBS, browser (WebRTC), dan software machine vision lainnya sebagai webcam native
- **Latency harus rendah**: untuk kebutuhan object detection realtime, delay yang terlalu besar membuat hasilnya tidak usable

## Pendekatan Solusi

Ada beberapa cara untuk mendapatkan video feed dari Android ke Linux:

| Pendekatan | Kelebihan | Kekurangan |
| --- | --- | --- |
| **Capture card fisik** | Plug and play, low latency, langsung jadi `/dev/video*` | Butuh hardware tambahan, biaya ekstra |
| **IP Webcam (via WiFi)** | Wireless, mudah setup | Latency tinggi, tergantung jaringan, butuh app di Android |
| **scrcpy + v4l2loopback** | Tanpa hardware, low latency via USB, native `/dev/video*` | Butuh setup kernel module, USB wired |
| **adb screenrecord + pipe** | Minimal dependency | Latency tinggi, frame rate terbatas, tidak stabil |
| **MJPEG stream + ffmpeg** | Fleksibel | Setup kompleks, overhead encoding/decoding |

Saya memilih **scrcpy + v4l2loopback** karena:

1. **Zero cost**: tidak butuh hardware tambahan, cukup kabel USB yang sudah ada
2. **Low latency**: scrcpy menggunakan koneksi USB langsung via ADB, latency-nya dalam kisaran 35-70ms
3. **Native V4L2 support**: scrcpy punya flag `--v4l2-sink` yang langsung menulis ke device v4l2loopback tanpa perlu pipeline ffmpeg terpisah
4. **Universal compatibility**: device `/dev/video*` bisa dibaca oleh OpenCV, OBS, browser, atau aplikasi apapun yang mendukung V4L2

> **Note:** V4L2 (Video for Linux 2) adalah API standar di kernel Linux untuk menangani video capture device. Semua aplikasi yang mendukung webcam di Linux menggunakan V4L2 untuk membaca dari `/dev/video*`.

## Implementasi Teknis

### Cek Dukungan V4L2 di Host

Sebelum mulai, pastikan kernel mendukung V4L2. Cek apakah sudah ada device video yang terdeteksi:

```
$ ls /dev/video*
```

Jika belum ada device apapun, itu normal, device virtual akan muncul setelah module v4l2loopback di-load. Yang perlu dicek adalah apakah kernel headers tersedia untuk kompilasi DKMS:

```
$ uname -r
$ ls /usr/lib/modules/$(uname -r)/build
```

Jika directory `build` ada, kernel headers sudah terinstall dan DKMS bisa mengompilasi module.

### Instalasi Paket

Install semua paket yang dibutuhkan:

```
$ sudo pacman -S v4l2loopback-dkms v4l-utils scrcpy android-tools
```

Penjelasan masing-masing paket:

| Paket | Fungsi |
| --- | --- |
| **v4l2loopback-dkms** | Kernel module untuk membuat virtual video device, versi DKMS agar otomatis recompile saat kernel update |
| **v4l-utils** | Utility untuk mengelola dan menginspeksi V4L2 device (`v4l2-ctl`, `v4l2-compliance`) |
| **scrcpy** | Mirror layar Android via ADB dengan opsi output ke V4L2 sink |
| **android-tools** | Menyediakan `adb` untuk koneksi USB ke Android device |

> **Penting:** Gunakan `v4l2loopback-dkms`, bukan `v4l2loopback` biasa. Versi DKMS (Dynamic Kernel Module Support) akan otomatis recompile module setiap kali kernel di-update. Tanpa DKMS, module akan hilang setelah kernel update dan harus di-install ulang secara manual.

### Load Kernel Module

Load module `v4l2loopback` terlebih dahulu:

```
$ sudo modprobe v4l2loopback exclusive_caps=1
```

Parameter `exclusive_caps=1` membuat device terlihat sebagai capture-only device, diperlukan agar browser dan aplikasi WebRTC bisa mengenali device ini sebagai webcam.

> **Tip:** Parameter `exclusive_caps=1` wajib diaktifkan jika ingin menggunakan virtual webcam di browser (Chrome, Firefox) untuk Google Meet, Zoom web, atau aplikasi WebRTC lainnya. Tanpa parameter ini, browser tidak akan menampilkan device di daftar webcam.

### Buat Virtual Device dengan v4l2loopback-ctl

Daripada menentukan nomor device secara hardcode via parameter `video_nr`, gunakan `v4l2loopback-ctl` untuk membuat device secara dinamis:

```bash
$ DEVICE=$(sudo v4l2loopback-ctl add -n "Android")
$ echo $DEVICE
```

Output yang diharapkan:

```
/dev/video0
```

Nomor device yang muncul tergantung device video yang sudah ada di sistem, bisa `/dev/video0`, `/dev/video1`, atau nomor lainnya. Dengan menyimpannya ke variabel `$DEVICE`, semua command selanjutnya tidak perlu di-hardcode ke nomor tertentu.

Penjelasan flag `v4l2loopback-ctl add`:

| Flag | Fungsi |
| --- | --- |
| `-n "Android"` | Label yang muncul di aplikasi seperti OBS atau browser saat memilih webcam |

Cek detail device menggunakan `v4l2-ctl`:

```bash
$ v4l2-ctl --device=$DEVICE --info
```

Output akan menampilkan informasi device termasuk label "Android" yang sudah di-set.

Untuk menghapus device saat sudah tidak dibutuhkan:

```bash
$ sudo v4l2loopback-ctl delete $DEVICE
```

### Aktifkan USB Debugging di Android

Sebelum menjalankan scrcpy, pastikan **USB Debugging** sudah aktif di Android device:

1. Buka **Settings** > **About Phone** > tap **Build Number** 7 kali untuk mengaktifkan Developer Options
2. Kembali ke **Settings** > **Developer Options** > aktifkan **USB Debugging**
3. Sambungkan Android ke PC via kabel USB
4. Jika muncul dialog "Allow USB Debugging", pilih **Allow**

Verifikasi device terdeteksi oleh ADB:

```
$ adb devices
```

Output yang diharapkan:

```
List of devices attached
XXXXXXXX	device
```

Jika status menunjukkan `unauthorized`, cek kembali dialog permission di Android.

### Jalankan scrcpy dengan V4L2 Sink

Jalankan scrcpy dengan output ke virtual webcam device:

```bash
$ scrcpy --v4l2-sink=$DEVICE --no-video-playback
```

Penjelasan flag:

- `--v4l2-sink=$DEVICE`: mengirim video stream ke device v4l2loopback yang sudah dibuat
- `--no-video-playback`: menonaktifkan window preview scrcpy, karena output hanya dibutuhkan di virtual webcam

Jika ingin tetap melihat preview di layar sekaligus mengirim ke virtual webcam, hilangkan flag `--no-video-playback`:

```bash
$ scrcpy --v4l2-sink=$DEVICE
```

Untuk mengatur resolusi dan bitrate agar lebih ringan:

```bash
$ scrcpy --v4l2-sink=$DEVICE --no-video-playback --max-size=1280 --video-bit-rate=4M
```

Parameter `--max-size=1280` membatasi dimensi terpanjang ke 1280 pixel, dan `--video-bit-rate=4M` mengatur bitrate ke 4 Mbps, cukup untuk kebanyakan kebutuhan computer vision tanpa membebani USB bandwidth.

### Verifikasi Stream

Setelah scrcpy berjalan, verifikasi stream menggunakan `ffplay`:

```bash
$ ffplay $DEVICE
```

Jika ffplay menampilkan layar Android, berarti pipeline sudah berjalan dengan benar. Selain ffplay, bisa juga diverifikasi menggunakan **OBS**, tambahkan source **Video Capture Device** dan pilih device "Android" dari dropdown.

Untuk verifikasi via OpenCV dengan Python, gunakan path device yang didapat dari `v4l2loopback-ctl`:

```python
import cv2

cap = cv2.VideoCapture("/dev/video0")  # sesuaikan dengan output v4l2loopback-ctl

while True:
    ret, frame = cap.read()
    if not ret:
        break
    cv2.imshow("Android Virtual Webcam", frame)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

Script ini membuka device virtual webcam dan menampilkan frame secara realtime, kalau layar Android muncul di window OpenCV, berarti device siap digunakan untuk pipeline computer vision.

### Auto-load Module Saat Boot

Agar module `v4l2loopback` otomatis ter-load setiap kali boot, buat file konfigurasi:

```bash
$ sudo tee /etc/modules-load.d/v4l2loopback.conf <<< "v4l2loopback"
```

File ini memberitahu systemd untuk me-load module `v4l2loopback` saat boot.

Kemudian buat file konfigurasi untuk parameter module:

```bash
$ sudo tee /etc/modprobe.d/v4l2loopback.conf <<< "options v4l2loopback exclusive_caps=1"
```

File ini memastikan parameter `exclusive_caps` selalu digunakan saat module di-load. Setelah reboot, module akan langsung tersedia dan device bisa dibuat secara dinamis menggunakan `v4l2loopback-ctl add -n "Android"` tanpa perlu menjalankan `modprobe` secara manual.

## Tantangan yang Dihadapi

**Device `/dev/video*` tidak muncul setelah `modprobe`.** Ini terjadi saat pertama kali saya mencoba install `v4l2loopback` tanpa varian DKMS. Module berhasil di-install, tapi setelah kernel update, module tidak bisa di-load karena versi kernel tidak cocok. Solusinya adalah selalu menggunakan `v4l2loopback-dkms`. DKMS akan otomatis recompile module setiap kali kernel berubah. Untuk memastikan DKMS sudah bekerja, cek status module:

```
$ dkms status
```

Output seharusnya menampilkan `v4l2loopback` dengan status `installed` untuk kernel yang sedang berjalan.

**Browser tidak mendeteksi virtual webcam.** Saya menghabiskan waktu cukup lama debugging kenapa Google Meet tidak menampilkan device "Android" di daftar webcam, padahal ffplay dan OBS bisa membacanya. Ternyata masalahnya ada di parameter `exclusive_caps`. Browser dan aplikasi WebRTC membutuhkan device yang melaporkan diri secara eksklusif sebagai capture device. Tanpa `exclusive_caps=1`, device melaporkan diri sebagai capture sekaligus output, dan browser mengabaikannya. Setelah menambahkan parameter ini dan me-reload module, browser langsung mendeteksi device.

**Latency saat processing OpenCV.** Latency scrcpy via USB sekitar 35-70ms, yang untuk kebanyakan kebutuhan OpenCV sudah cukup. Tapi kalau resolusi terlalu tinggi, total latency pipeline (screen capture + encoding + v4l2 + processing) bisa membengkak. Solusinya, saya menurunkan resolusi ke 720p dengan `--max-size=720`. Hasilnya jauh lebih responsif tanpa mengorbankan kualitas yang dibutuhkan untuk image processing.

## Insight dan Pembelajaran

- **DKMS adalah keharusan untuk kernel module pihak ketiga**: tanpa DKMS, setiap kernel update berpotensi memecahkan module yang sudah ter-install. Ini berlaku tidak hanya untuk v4l2loopback, tapi semua out-of-tree kernel module
- **`exclusive_caps=1` bukan opsional**: parameter ini kritis untuk kompatibilitas dengan browser dan aplikasi WebRTC. Tanpanya, device hanya bisa dibaca oleh aplikasi yang langsung mengakses V4L2 API seperti ffplay atau OpenCV, tapi tidak oleh browser
- **Resolusi dan bitrate sangat mempengaruhi latency**: untuk use case yang sensitif terhadap latency seperti object detection, menurunkan resolusi dan bitrate memberikan dampak signifikan. Perbedaan antara 1080p dan 720p bisa menghemat 30-50ms di total pipeline
- **scrcpy `--v4l2-sink` lebih efisien dari pipeline manual**: dibanding setup manual menggunakan ffmpeg untuk pipe stream ke v4l2loopback, built-in support scrcpy mengurangi satu layer proses dan menghasilkan latency yang lebih rendah
- **`v4l2loopback-ctl` lebih fleksibel dari parameter modprobe**: daripada hardcode `video_nr` saat load module, `v4l2loopback-ctl add` dan `delete` memungkinkan membuat dan menghapus device secara dinamis tanpa reload module. Ini juga memudahkan kalau butuh multiple device, cukup jalankan `v4l2loopback-ctl add` beberapa kali untuk feed dari beberapa Android device secara bersamaan

## Penutup

Kombinasi v4l2loopback dan scrcpy memberikan cara yang praktis untuk mengubah layar Android menjadi virtual webcam di Linux tanpa hardware tambahan. Setup-nya memang butuh beberapa langkah konfigurasi kernel module, tapi setelah jalan, hasilnya solid, device `/dev/video*` yang bisa dibaca oleh OpenCV, OBS, browser, atau aplikasi apapun yang mendukung V4L2.

## Referensi

- [scrcpy - V4L2 Documentation](https://github.com/Genymobile/scrcpy/blob/master/doc/v4l2.md), diakses pada2026-05-30
- [Arch Wiki - v4l2loopback](https://wiki.archlinux.org/title/V4l2loopback), diakses pada2026-05-30
- [Arch Wiki - Kernel module](https://wiki.archlinux.org/title/Kernel_module), diakses pada2026-05-30
