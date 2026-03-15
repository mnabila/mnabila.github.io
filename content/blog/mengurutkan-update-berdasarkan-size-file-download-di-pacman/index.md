+++
draft = false
date = '2022-04-26'
lastmod = '2026-03-15'
title = 'Mengurutkan Update Berdasarkan Size File Download Di Pacman'
type = 'blog'
description = 'Metode sorting update package berdasarkan ukuran file download di pacman untuk koneksi internet lambat.'
image = ''
tags = ['pacman', 'update', 'bash']
+++

## Latar Belakang

Sebagai pengguna Archlinux, rutinitas update sistem adalah sesuatu yang idealnya dilakukan secara berkala. Archlinux adalah rolling release -- artinya update datang terus-menerus, bukan dalam siklus rilis besar seperti Ubuntu atau Fedora. Menunda update terlalu lama justru meningkatkan risiko dependency conflict dan breaking changes yang menumpuk.

Masalahnya, koneksi internet yang saya gunakan saat itu sangat terbatas dari sisi bandwidth. Proses `pacman -Syu` yang harus mengunduh ratusan megabyte atau bahkan gigabyte setelah lama tidak update menjadi beban tersendiri buat pengguna kouta internet seperti saya. 

## Permasalahan

Perilaku default `pacman -Syu` adalah mengunduh semua paket secara bersamaan tanpa memperhatikan urutan ukuran. Ini berarti paket-paket kecil yang sebenarnya bisa langsung selesai didownload harus "mengantri" bersama paket besar yang butuh waktu lama. Di koneksi yang tidak stabil, download bisa gagal di tengah jalan dan harus diulang dari awal.

Muncul sebuah ide sederhana: bagaimana kalau proses download diurutkan berdasarkan ukuran file, dari yang terkecil ke terbesar? Dengan begitu, paket-paket kecil bisa diunduh lebih dahulu.

## Pendekatan Solusi

Saya menulis bash script yang memisahkan proses download dan instalasi. Script mengambil daftar paket yang tersedia untuk update, mengurutkannya berdasarkan ukuran download, lalu mengunduhnya satu per satu dari yang terkecil. Setelah semua selesai didownload, baru dilanjutkan dengan proses instalasi.

Ada dua metode yang saya eksplorasi: **query package** dan **update package**. Keduanya menghasilkan output yang sama, tapi dengan pendekatan yang berbeda.

## Implementasi Teknis

### Metode Query Package

```bash
#!/bin/bash
LC_ALL=C pacman -Qui | awk '/^Name/{name=$3} /^Installed Size/{print $4$5, name}' | sort -h | cut -d " " -f 2 | while read line; do
  sudo pacman -Sw --noconfirm $line
done
```

Metode ini mengambil informasi dari database paket yang sudah terinstall (`pacman -Qui`), kemudian mengekstrak nama dan ukuran menggunakan `awk`, mengurutkannya dengan `sort -h` (human-readable sort), lalu mendownload satu per satu.

### Metode Update Package

```bash
#!/bin/bash
LC_ALL=C pacman -Su --print-format "%s %n" | sort -h | while read line; do
  size=$(echo $line | cut -d " " -f 1)
  name=$(echo $line | cut -d " " -f 2)
  if (($size > 0)); then
    sudo pacman -Sdw --noconfirm $name
  fi
done
```

Metode ini memanfaatkan `--print-format` dari `pacman -Su` yang langsung memberikan ukuran download dan nama paket dalam format yang ringkas. Flag `%s` untuk size dan `%n` untuk nama.

Setelah script download selesai dijalankan, lanjutkan dengan proses instalasi:

```
$ sudo pacman -Su
```

## Tantangan yang Dihadapi

Tantangan pertama ada pada pemilihan metode yang tepat. Metode query package ternyata jauh lebih lambat karena `pacman -Qui` harus menampilkan seluruh informasi tiap paket -- termasuk deskripsi, dependency, dan metadata lain -- lalu diolah menggunakan `awk`. Ini terasa overkill dan boros waktu, terutama di sistem dengan ratusan paket terinstall.

Metode update package jauh lebih efisien karena `pacman -Su --print-format` langsung memberikan informasi yang dibutuhkan tanpa overhead parsing informasi yang tidak relevan.

Tantangan kedua adalah memastikan script hanya mendownload paket yang benar-benar butuh update. Filter `if (($size > 0))` pada metode update package diperlukan untuk menyaring paket yang sudah up-to-date (size = 0).

## Insight dan Pembelajaran

Dari pengalaman ini, ada beberapa hal yang bisa diambil:

Pertama, memahami opsi-opsi pacman yang tersedia sangat membantu. Flag `--print-format` adalah fitur yang powerful tapi jarang dibahas di tutorial-tutorial umum. Dengan flag ini, output pacman bisa diformat sesuai kebutuhan scripting.

Kedua, memisahkan proses download (`-Sdw`) dan instalasi (`-Su`) adalah strategi yang berguna untuk koneksi tidak stabil. Paket yang sudah berhasil didownload tersimpan di cache dan tidak perlu diunduh ulang saat instalasi.

Ketiga, tidak semua pendekatan yang intuitif itu efisien. Metode query package terlihat "lebih benar" karena langsung membaca database, tapi dalam praktiknya jauh lebih lambat dibanding metode update package yang memanfaatkan fitur built-in pacman.

## Penutup

Metode update package terbukti menjadi solusi yang paling efisien untuk kebutuhan ini. Script-nya sederhana, langsung memanfaatkan fitur bawaan pacman, dan hasilnya sesuai harapan. Bagi pengguna Archlinux lain yang menghadapi keterbatasan bandwidth serupa, pendekatan ini bisa menjadi alternatif yang pragmatis.

## Referensi

- [Wiki Archlinux - Pacman Tips and Tricks](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks) -- Diakses pada 2022-04-26
- `man pacman` -- Diakses pada 2022-04-26
