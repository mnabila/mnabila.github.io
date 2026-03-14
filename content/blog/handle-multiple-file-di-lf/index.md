+++
draft = false
date = '2022-01-18'
title = 'Handle Multiple File Di Lf'
type = 'blog'
description = 'Cara menghandle multiple file yang dibuka dalam satu instance di lf file manager.'
image = ''
tags = ['tui', 'file manager', 'lf']
+++

## Intro

Seperti yang sudah pernah saya bahas di tulisan sebelumnya, salah satu "kelebihan" lf file manager adalah banyaknya kekurangan yang harus kita atasi sendiri. Salah satunya? Membuka beberapa file sekaligus dalam satu instance program. Contoh sederhananya: kita pilih beberapa file video dan ingin membukanya sebagai playlist di pemutar video -- bukan membuka masing-masing di window terpisah.

## Pembahasan

Jujur, masalah ini cukup bikin pusing. Saya bahkan sempat mencoba memodifikasi source code lf di bagian `os.go`. Hasilnya? Output-nya memang sesuai harapan, tapi malah merusak fitur lain. Alih-alih jadi fitur baru, yang muncul justru bug baru. Ya sudah, niat mulia pun diurungkan.

Setelah menjelajahi beberapa issue di GitHub, ternyata ada pengguna lain yang menghadapi masalah serupa. Saya langsung ikut berdiskusi di sana (dengan kemampuan bahasa Inggris level Google Translate tentunya) dan akhirnya menemukan dua kunci solusi:

* Penggunaan environment variable `IFS` yang benar
* Pemberian tanda kutip (double/single quote) pada nama file yang mengandung spasi

Secara default, `IFS` di lf bernilai kosong. Ketika kita set `IFS` ke karakter newline, maka output dari environment variable `$fx` (daftar file yang dipilih) akan berubah format -- setiap file dipisahkan per baris.

Soal pemberian tanda kutip pada nama file, ini menurut saya wajib dilakukan meskipun nama file-nya tidak mengandung spasi. Tujuannya untuk berjaga-jaga agar command tidak error ketika bertemu file dengan karakter spasi di namanya.

Cukup teorinya, sekarang masuk ke implementasi. Di `lfrc`, saya membuat dua command: `open` dan `multifile`.

### Command Multifile

```sh
# multiple file handler
cmd multifile &{{
    IFS="";eval "$1 $(echo $fx | awk '{ printf "\"%s\" ", $0 }')"
}}
```

### Command Open

```sh
# file opener
cmd open ${{
    case $(file --mime-type "$f" -bL) in
        video/*) lf -remote "send $id multifile mpv";;
    esac
}}
```

Di contoh command `open` ini, saya hanya mendemonstrasikan penggunaan `multifile` untuk file video yang dibuka dengan mpv. Kamu bisa menambahkan case lain sesuai kebutuhan.

## Kesimpulan

Jangan coba lf kalau tidak siap diajak ribet -- tapi kalau sudah menemukan solusinya, kepuasan yang didapat tak ternilai.

## Referensi

- [Add double quotes to every line and then add a comma at the end of the line](https://unix.stackexchange.com/questions/223677/how-to-add-double-quotes-to-every-line-and-then-add-a-comma-at-the-end-of-the-li) -- Diakses pada 2022-01-18
- [Open multiple files with the same instance](https://github.com/gokcehan/lf/issues/719) -- Diakses pada 2022-01-18
- [Komentar saya di issue lf](https://github.com/gokcehan/lf/issues/719#issuecomment-1015368978) -- Diakses pada 2022-01-18
