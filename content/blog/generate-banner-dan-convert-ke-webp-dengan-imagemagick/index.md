+++
draft = false
date = '2026-04-08'
title = 'Generate Banner dan Convert Image ke WebP Dengan ImageMagick'
type = 'blog'
description = 'Cara menggunakan ImageMagick untuk resize, optimize, dan convert gambar ke format WebP langsung dari terminal tanpa perlu aplikasi GUI.'
image = ''
tags = ['imagemagick', 'webp', 'cli']
+++

## Latar Belakang

Punya gambar 4MB untuk banner blog itu bukan hal yang lumrah terjadi. Download dari Unsplash atau stock photo, resolusinya bisa 4000x6000 pixel hal ini bagus untuk dicetak, tapi terlalu besar untuk sebuah banner di web. Kalau tidak optimalkan, halaman akan menjadi lambat, skor Lighthouse turun kayak harga btc, dan pengunjung kabur sebelum konten sempat ke-load.

Biasanya orang buka Photoshop atau GIMP untuk resize dan compress gambar. Tapi kalau cuma butuh resize, crop, dan convert format terlalu berlebihan jikalau menggunakan aplikasi tersebut. Bermodalnya satu tools **ImageMagick** bisa melakukan semua hal itu langsung dari terminal dengan satu baris command.

## Permasalahan

Beberapa masalah yang sering ditemui saat mengelola gambar untuk web:

- **Ukuran file terlalu besar** -- gambar dari kamera atau stock photo biasanya berukuran beberapa MB, sedangkan untuk web idealnya di bawah 200KB
- **Resolusi terlalu tinggi** -- banner blog tidak perlu resolusi 4000px, cukup 1200px sudah lebih dari cukup
- **Format tidak optimal** -- PNG dan JPEG masih banyak dipakai, padahal WebP bisa menghasilkan ukuran 25-35% lebih kecil dengan kualitas visual yang sama
- **Proses manual yang repetitif** -- kalau punya banyak gambar, resize satu per satu lewat GUI itu membuang waktu

## Kenapa WebP?

WebP adalah format gambar yang dikembangkan oleh Google, dirancang khusus untuk web. Perbandingan dengan format lain:

| Format   | Kompresi             | Transparansi | Animasi | Browser Support      |
| -------- | -------------------- | ------------ | ------- | -------------------- |
| **JPEG** | Lossy                | Tidak        | Tidak   | Semua                |
| **PNG**  | Lossless             | Ya           | Tidak   | Semua                |
| **WebP** | Lossy & Lossless     | Ya           | Ya      | Semua browser modern |

WebP menggabungkan kelebihan JPEG (kompresi lossy yang efisien) dan PNG (transparansi) dalam satu format dengan ukuran file yang lebih kecil. Semua browser modern sudah support WebP, jadi tidak ada alasan untuk tidak memakainya.

## Implementasi Teknis

### Instalasi ImageMagick

```bash
$ sudo pacman -S imagemagick
```

Verifikasi instalasi dan pastikan support WebP sudah ada:

```bash
$ magick -version
$ magick identify -list format | grep -i webp
```

Kalau output menunjukkan `WEBP* WEBP rw+`, berarti ImageMagick sudah bisa baca dan tulis format WebP.

### Mulai Dari yang Simpel

#### Cek Detail Gambar

Sebelum memproses gambar, cek dulu informasi dasarnya:

```bash
$ magick identify banner.png
```

Output akan menampilkan format, resolusi, color depth, dan ukuran file. Ini berguna untuk menentukan seberapa banyak gambar perlu di-resize.

Untuk informasi lebih detail:

```bash
$ magick identify -verbose banner.png | head -30
```

#### Resize Gambar

Resize berdasarkan lebar (height otomatis menyesuaikan aspect ratio):

```bash
$ magick banner.png -resize 1200x banner-resized.png
```

Resize berdasarkan tinggi:

```bash
$ magick banner.png -resize x630 banner-resized.png
```

Resize dengan lebar dan tinggi spesifik (mempertahankan aspect ratio, fit di dalam batas):

```bash
$ magick banner.png -resize 1200x630 banner-resized.png
```

Resize dengan paksa (ignore aspect ratio):

```bash
$ magick banner.png -resize 1200x630! banner-resized.png
```

> **Tip:** Tanda `!` di akhir dimensi memaksa gambar di-stretch ke ukuran yang ditentukan. Tanpa `!`, ImageMagick akan mempertahankan aspect ratio dan fit gambar di dalam batas yang diberikan.

#### Atur Kualitas Kompresi

Parameter `-quality` mengontrol level kompresi (0-100, semakin tinggi semakin bagus tapi file semakin besar):

```bash
# Kualitas 80% -- sweet spot untuk web
$ magick banner.png -quality 80 banner-compressed.jpg

# Kualitas 60% -- lebih kecil, sedikit penurunan visual
$ magick banner.png -quality 60 banner-compressed.jpg
```

Untuk kebanyakan banner blog, kualitas 75-85% sudah cukup -- perbedaan visualnya hampir tidak terlihat di layar.

### Convert ke WebP

#### Langsung Convert

```bash
$ magick banner.png banner.webp
```

Cukup ganti extension output ke `.webp` dan ImageMagick otomatis convert ke webp.

#### Convert Dengan Kualitas Spesifik

```bash
$ magick banner.png -quality 80 banner.webp
```

#### Resize Sekaligus Convert

Ini yang paling sering dipakai -- resize dan convert dalam satu command:

```bash
$ magick banner.png -resize 1200x -quality 80 banner.webp
```

Command di atas akan:

1. Baca `banner.png`
2. Resize lebar ke 1200px (height auto)
3. Set kualitas kompresi 80%
4. Simpan sebagai `banner.webp`

#### Perbandingan Hasil

Untuk melihat seberapa efektif konversi ke WebP, bandingkan ukuran file sebelum dan sesudah:

```bash
# Gambar asli
$ ls -lh banner.png

# Convert ke JPEG
$ magick banner.png -resize 1200x -quality 80 banner.jpg && ls -lh banner.jpg

# Convert ke WebP
$ magick banner.png -resize 1200x -quality 80 banner.webp && ls -lh banner.webp
```

Biasanya WebP menghasilkan file 25-35% lebih kecil dibanding JPEG dengan kualitas visual yang setara.

### Batch Processing

Kalau punya banyak gambar yang perlu di-convert sekaligus, pakai loop di shell:

#### Convert Semua PNG ke WebP

```bash
$ for img in *.png; do
    magick "$img" -resize 1200x -quality 80 "${img%.png}.webp"
  done
```

#### Convert Semua JPEG ke WebP

```bash
$ for img in *.jpg; do
    magick "$img" -resize 1200x -quality 80 "${img%.jpg}.webp"
  done
```

#### Convert Semua Gambar dan Hapus File Asli

```bash
$ for img in *.{png,jpg,jpeg}; do
    [ -f "$img" ] || continue
    magick "$img" -resize 1200x -quality 80 "${img%.*}.webp" && rm "$img"
  done
```

> **Hati-hati:** Script di atas menghapus file asli setelah convert. Pastikan sudah punya backup atau yakin hasilnya sesuai sebelum menjalankan.

## Bonus Tambahan

#### Strip Metadata

Gambar dari kamera atau stock photo biasanya berisi metadata EXIF (lokasi GPS, model kamera, dll). Untuk web, metadata ini tidak perlu dan hanya menambah ukuran file:

```bash
$ magick banner.png -strip -resize 1200x -quality 80 banner.webp
```

Flag `-strip` menghapus semua metadata dari gambar.

#### Crop ke Aspect Ratio Tertentu

Kalau butuh banner dengan aspect ratio 16:9:

```bash
$ magick banner.png -resize 1200x -gravity center -crop 1200x675+0+0 +repage banner.webp
```

Penjelasan:

| Parameter            | Fungsi                               |
| -------------------- | ------------------------------------ |
| `-resize 1200x`      | Resize lebar ke 1200px               |
| `-gravity center`    | Set titik referensi ke tengah gambar |
| `-crop 1200x675+0+0` | Crop ke 1200x675 (16:9) dari tengah  |
| `+repage`            | Reset virtual canvas setelah crop    |

#### Tambah Border atau Padding

```bash
$ magick banner.png -resize 1200x -bordercolor white -border 20x20 banner.webp
```

## Tips dan Best Practices

- **Gunakan `-strip` untuk production** -- metadata EXIF tidak diperlukan di web dan bisa menambah 50-100KB ke ukuran file
- **Kualitas 75-85% adalah sweet spot** -- di bawah 75% mulai terlihat artefak kompresi, di atas 85% penambahan ukuran file tidak sebanding dengan peningkatan kualitas
- **Resize sebelum compress** -- selalu resize dulu baru set kualitas. Compress gambar 4000px lalu resize ke 1200px hasilnya berbeda (dan lebih buruk) dari resize dulu baru compress
- **Lebar 1200px cukup untuk kebanyakan blog** -- ini sudah cover layar desktop dan retina mobile. Kalau blog layout-nya lebih sempit (800px misalnya), bisa turunkan ke 960px
- **Selalu cek hasil secara visual** -- angka kualitas itu relatif tergantung konten gambar. Foto dengan banyak detail mungkin butuh kualitas lebih tinggi dari gambar yang simpel

## Bonus: Bash Script

Supaya tidak perlu ketik command panjang berulang-ulang, berikut script yang bisa disimpan dan dipakai kapan saja:

```bash
#!/bin/bash
# img2webp - Resize dan convert gambar ke WebP
# Usage: img2webp <input> [width] [quality]
# Example: img2webp photo.png 1200 80

set -euo pipefail

INPUT="${1:?Usage: img2webp <input> [width] [quality]}"
WIDTH="${2:-1200}"
QUALITY="${3:-80}"
OUTPUT="${INPUT%.*}.webp"

if [ ! -f "$INPUT" ]; then
    echo "Error: file '$INPUT' tidak ditemukan"
    exit 1
fi

if ! command -v magick &>/dev/null; then
    echo "Error: imagemagick belum terinstall"
    exit 1
fi

BEFORE=$(du -h "$INPUT" | cut -f1)

magick "$INPUT" -strip -resize "${WIDTH}x" -quality "$QUALITY" "$OUTPUT"

AFTER=$(du -h "$OUTPUT" | cut -f1)
echo "$INPUT ($BEFORE) -> $OUTPUT ($AFTER)"
```

Simpan sebagai `img2webp`, buat executable, lalu pakai:

```bash
$ chmod +x img2webp

# Default: resize 1200px, quality 80%
$ ./img2webp banner.png

# Custom width dan quality
$ ./img2webp banner.png 960 75

# Batch: convert semua PNG di folder saat ini
$ for img in *.png; do ./img2webp "$img"; done
```

Output script akan menampilkan perbandingan ukuran sebelum dan sesudah, jadi langsung kelihatan berapa banyak yang di-hemat.

## Penutup

ImageMagick itu tool yang powerful tapi sering di-underestimate karena tidak punya GUI. Untuk kebutuhan optimasi gambar web mulai resize, compress, convert ke WebP dengan satu baris command sudah cukup. Tidak perlu buka aplikasi berat, tidak perlu klik-klik menu, dan yang paling penting bisa di-automate lewat script untuk batch processing. Kombinasi `-strip -resize 1200x -quality 80 output.webp` itu sudah jadi template yang bisa dipakai untuk hampir semua kebutuhan banner blog.

## Referensi

- [ImageMagick - Command-line Processing](https://imagemagick.org/script/command-line-processing.php) -- Diakses pada 2026-04-08
- [Google Developers - WebP](https://developers.google.com/speed/webp) -- Diakses pada 2026-04-08
