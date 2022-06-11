---
title: "Mengurutkan Update Berdasarkan Size File Download Di Pacman"
date: 2022-04-26T10:40:14+07:00
categories: ["tips"]
tags: ["pacman", "update", "bash"]
lastmod: 2022-04-26T15:52:00+07:00
---

# Intro

Yah dikarenakan saat ini aku sedang menggunakan koneksi internet yang cukup lelet, akupun mengalami kesuliat saat mau melakukan upgrade package yang ada didalam system archlinuxku, sampe sampe sudah beberapa bulan tak update karna hal tersebut. Karna udah jenuh dengan hal itu pengen ngerasain paket yang lebih baru akupun mendapatkan sebuah ide yang cukup mantab (hahahaha) yakni menggunakan metode sorting berdasarkan size update packagenya.

# Pembahasan

Disini aku mendapatkan dua metode yang bisa digunakan yakni menggunakan metode query package dam menggunakan metode update package pada pacman.

## metode query package

```
#!/bin/bash
LC_ALL=C pacman -Qui | awk '/^Name/{name=$3} /^Installed Size/{print $4$5, name}' | sort -h | cut -d " " -f 2 | while read line; do
  sudo pacman -Sw --noconfirm $line
done
```

## metode update package

```
#!/bin/bash
LC_ALL=C pacman -Su --print-format "%s %n" | sort -h | while read line; do
  size=$(echo $line | cut -d " " -f 1)
  name=$(echo $line | cut -d " " -f 2)
  if (($size > 0)); then
    sudo pacman -Sdw --noconfirm $name
  fi
done
```

Dari kedua metode diatas menurutku metode update package lebih cocok dikarenakan proses yang dibutuhkan cukup singkat dibandingkan metode query package, karna waktu proses query package si pacman perlu menampilkan keseluruhan informasi tiap paketnya. Informasi-informasi tersebut kemudian diolah menggunakan awk (overkill menurutku) sehingga hasilnya kurang cepat. Berbeda dengan metode update package, metode ini si pacman langsung memberikan informasi yang kubutuhkan sehingga bisa langsung masuk tahap sorting berdasarkan size update paketnya.

Setelah menjalankan perintah script diatas baru dilanjutkan dengan proses install.

```
sudo pacman -Su
```

# Kesimpulan

pakek aja metode update package, karna lebih cepet (wkwkwkw)

# Referensi

- [Wiki Archlinux](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks) Diakses pada 2022-04-26
- `man pacman` Diakses pada 2022-04-26
