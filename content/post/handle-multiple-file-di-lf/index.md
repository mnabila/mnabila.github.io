---
title: "Handle Multiple File Di Lf"
date: 2022-01-18T19:43:54+07:00
categories: ["terminal", "application"]
tags: ["tui", "file manager", "lf"]
---

## Biasa intro 🤣
Seperti yang sudah pernah kusinggung di catatan asbun sebelumnya, bahwa kelebihan dari lf file manager ini adalah banyak kekurangannya salah satunya untuk menghandle masalah multiple file yang akan dibuka dalam satu instance atau gambaran gampangnya pengen bikin sebuah playlist di pemutar video dari beberapa file yang di pilih.

## Pembahasan
Untuk menghandle multiple file di lf ini cukup susah asli aku ga becanda sampe sampe beberapa hari lalu aku coba ubah source code lf dibagian os.go akan tetapi walaupun berhasil membuat output yang sesuai ternyata hal ini mempengaruhi fitur yang lain, bukannya menjadi fitur malah jadi bug baru alhasil kuurungkan niatku haha.

Setelah keliling dibeberapa issue ternyata ada yang menanyakan hal serupa jadi aku tidak perlu membuat issue lagi dan langsung nimbrung pembahasan disana dengan skill bahasa lingginya level expert via google translate akhirnya mendapatkan beberapa kesimpulan, meliputi

* penggunaan environment IFS yang benar
* pemberian double quote pada nama file yang memiliki karakter space


Untuk penggunaan environment IFS sendiri di lf secara default memiliki value nill/kosong, apabila environment IFS ini diset ke karakter new line maka akan mempengaruhi output dari environment fx (list selected file lf).

Pemberian double quote (bisa juga pakek single quote) pada nama file ini menurutku wajib walaupun nama filenya tidak mengandung karakter space sama sekali. Hal ini bertujuan untuk berjaga-jaga apabila bertemu nama file yang memiliki karakter space didalamnya command tidak error.

udah dulu pembahasannya sekarang masuk ke implementasinya, pada `lfrc` aku membuat dua buah command yakni command open dan multifile

### Multiple
```
# multiple file handler
cmd multifile &{{
    IFS="";eval "$1 $(echo $fx | awk '{ printf "\"%s\" ", $0 }')"
}}
```

### Open

```
# file opener
cmd open ${{
    case $(file --mime-type "$f" -bL) in
        video/*) lf -remote "send $id multifile mpv";;
    esac
}}
```

pada command open ini aku hanya mencontohkan penggunaan dari command multifile untuk menghandle file video yang dibuka dengan program mpv.


## Kesimpulan
jangan coba lf klok ga mau ribet 🤣


## Referensi
. [Add double quotes to every line and then add a comma at the end of the line](https://unix.stackexchange.com/questions/223677/how-to-add-double-quotes-to-every-line-and-then-add-a-comma-at-the-end-of-the-li) Diakses pada 2022-01-18
. [Open multiple files with the same instance](https://github.com/gokcehan/lf/issues/719) Diakses pada 2022-01-18
. [Komentarku di issue lf](https://github.com/gokcehan/lf/issues/719#issuecomment-1015368978) Diakses pada 2022-01-18

