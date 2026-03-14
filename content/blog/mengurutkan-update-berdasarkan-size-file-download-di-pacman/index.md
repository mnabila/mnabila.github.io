+++
draft = false
date = '2022-04-26'
title = 'Mengurutkan Update Berdasarkan Size File Download Di Pacman'
type = 'blog'
description = 'Metode sorting update package berdasarkan ukuran file download di pacman untuk koneksi internet lambat.'
image = ''
tags = ['pacman', 'update', 'bash']
+++

## Intro

Koneksi internet lemot itu musuh utama pengguna Archlinux. Bagaimana tidak -- sistem yang idealnya rutin di-update jadi terbengkalai berbulan-bulan karena bandwidth yang tidak mendukung. Situasi ini persis yang saya alami, dan setelah jenuh menunda-nunda update, akhirnya muncul sebuah ide: **bagaimana kalau proses download update diurutkan berdasarkan ukuran file?**

Logikanya sederhana -- dengan mengunduh paket terkecil lebih dulu, kita bisa segera menginstall paket-paket yang sudah selesai tanpa harus menunggu semua selesai didownload. Lebih efisien, terutama di koneksi yang tidak stabil.

## Pembahasan

Ada dua metode yang bisa digunakan: **query package** dan **update package**.

### Metode Query Package

```bash
#!/bin/bash
LC_ALL=C pacman -Qui | awk '/^Name/{name=$3} /^Installed Size/{print $4$5, name}' | sort -h | cut -d " " -f 2 | while read line; do
  sudo pacman -Sw --noconfirm $line
done
```

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

Dari kedua metode di atas, **metode update package** jauh lebih efisien. Alasannya? Pada metode query package, pacman perlu menampilkan seluruh informasi tiap paket terlebih dahulu, lalu diolah menggunakan `awk` -- ini terasa overkill dan lambat. Sementara pada metode update package, pacman langsung memberikan informasi ukuran dan nama paket dalam format yang ringkas, sehingga proses sorting bisa langsung dilakukan.

Setelah script download selesai dijalankan, lanjutkan dengan proses instalasi:

```
$ sudo pacman -Su
```

## Kesimpulan

Gunakan metode update package -- lebih cepat dan lebih efisien. Sesederhana itu.

## Referensi

- [Wiki Archlinux - Pacman Tips and Tricks](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks) -- Diakses pada 2022-04-26
- `man pacman` -- Diakses pada 2022-04-26
