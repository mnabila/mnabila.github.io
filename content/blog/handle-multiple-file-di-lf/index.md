+++
draft = false
date = '2022-01-18'
lastmod = '2026-03-15'
title = 'Handle Multiple File Di Lf'
type = 'blog'
description = 'Cara menghandle multiple file yang dibuka dalam satu instance di lf file manager.'
image = ''
tags = ['tui', 'file manager', 'lf']
+++

## Latar Belakang

Setelah beberapa hari menggunakan lf sebagai file manager utama menggantikan ranger, saya mulai menemukan berbagai keterbatasan yang harus diatasi sendiri. Seperti yang pernah saya bahas di tulisan sebelumnya, lf adalah file manager minimalis yang secara default hanya menyediakan fitur seadanya -- sisanya diserahkan sepenuhnya kepada pengguna untuk diimplementasikan melalui shell script.

## Permasalahan

Salah satu masalah yang cukup mengganggu adalah ketidakmampuan lf untuk membuka beberapa file sekaligus dalam satu instance program. Contoh skenarionya: saya memilih beberapa file video dan ingin membukanya sebagai playlist di mpv -- bukan membuka masing-masing video di window terpisah. Di ranger, hal ini bisa dilakukan secara default. Di lf, memilih 10 video dan menekan enter akan membuka 10 window mpv.

## Pendekatan Solusi

Awalnya saya mencoba pendekatan yang paling "engineer": memodifikasi source code lf langsung di bagian `os.go`. Hasilnya? Output-nya memang sesuai harapan, tapi justru merusak fitur lain. Alih-alih fitur baru, yang muncul malah bug baru. Niat mulia pun diurungkan.

Saya kemudian beralih ke pendekatan yang lebih pragmatis -- mencari solusi dari komunitas. Setelah menjelajahi beberapa issue di GitHub, saya menemukan pengguna lain yang menghadapi masalah serupa. Ikut berdiskusi di sana (dengan kemampuan bahasa Inggris level Google Translate) dan akhirnya menemukan dua kunci solusi:

- Penggunaan environment variable `IFS` yang benar
- Pemberian tanda kutip (double/single quote) pada nama file yang mengandung spasi

## Implementasi Teknis

Secara default, `IFS` di lf bernilai kosong. Ketika kita set `IFS` ke karakter newline, output dari environment variable `$fx` (daftar file yang dipilih) berubah format -- setiap file dipisahkan per baris. Ini memungkinkan kita mengolah daftar file secara proper.

Pemberian tanda kutip pada nama file saya terapkan secara konsisten meskipun nama file-nya tidak mengandung spasi -- tujuannya berjaga-jaga agar command tidak error ketika bertemu file dengan karakter spasi.

Di `lfrc`, saya membuat dua command: `multifile` dan modifikasi pada `open`.

### Command Multifile

```sh
# multiple file handler
cmd multifile &{{
    IFS="";eval "$1 $(echo $fx | awk '{ printf "\"%s\" ", $0 }')"
}}
```

Command ini menerima satu argumen (nama program), lalu menggunakan `awk` untuk membungkus setiap path file dengan double quote, kemudian menjalankan semuanya sebagai satu command melalui `eval`.

### Command Open

```sh
# file opener
cmd open ${{
    case $(file --mime-type "$f" -bL) in
        video/*) lf -remote "send $id multifile mpv";;
    esac
}}
```

Di contoh ini, saya mendemonstrasikan penggunaan `multifile` khusus untuk file video yang dibuka dengan mpv. Case lain bisa ditambahkan sesuai kebutuhan -- misalnya `image/*` untuk image viewer atau `audio/*` untuk music player.

## Tantangan yang Dihadapi

Tantangan terbesar justru di awal -- saat mencoba memodifikasi source code Go secara langsung. Saya menghabiskan waktu cukup lama untuk memahami flow `os.go` di lf, membuat perubahan, compile, test, dan akhirnya menemukan bahwa perubahan tersebut menimbulkan side effect yang tidak diinginkan. Ini menjadi pelajaran penting bahwa tidak semua masalah harus diselesaikan di level source code.

Selain itu, memahami perilaku `IFS` dan cara `awk` memproses output `$fx` memerlukan beberapa kali percobaan. Edge case seperti nama file dengan spasi atau karakter spesial perlu dihandle dengan benar agar command tidak pecah.

## Insight dan Pembelajaran

Pengalaman ini mengajarkan bahwa solusi dari komunitas sering kali lebih efektif dan sustainable dibandingkan memodifikasi source code secara langsung. Solusi berbasis konfigurasi lebih mudah di-maintain -- tidak perlu re-compile setiap kali ada update versi lf.

Berdiskusi di GitHub issue, meskipun dengan bahasa Inggris yang pas-pasan, tetap memberikan hasil yang produktif. Komunitas open source umumnya sangat membantu selama kita menjelaskan masalah dengan jelas dan menunjukkan usaha yang sudah dilakukan.

## Penutup

Masalah multiple file handling di lf akhirnya terselesaikan dengan pendekatan shell script yang relatif sederhana. Solusi ini sudah berjalan stabil di workflow harian saya. Meskipun lf memang memaksa penggunanya untuk "ribet" di awal, kepuasan menemukan solusi sendiri adalah bagian dari pengalaman menggunakan tool minimalis ini.

## Referensi

- [Add double quotes to every line and then add a comma at the end of the line](https://unix.stackexchange.com/questions/223677/how-to-add-double-quotes-to-every-line-and-then-add-a-comma-at-the-end-of-the-li) -- Diakses pada 2022-01-18
- [Open multiple files with the same instance](https://github.com/gokcehan/lf/issues/719) -- Diakses pada 2022-01-18
- [Komentar saya di issue lf](https://github.com/gokcehan/lf/issues/719#issuecomment-1015368978) -- Diakses pada 2022-01-18
