---
title: "Implementasi Gnu Stow Untuk Management Dotfiles Di Archlinux"
date: 2022-01-05T13:16:00+07:00
categories: ["terminal", "application"]
tags: ["stow"]
---

## curhat sebentar

Sebenernya sudah lama ingin membahas ini topik, tapi karna aku sendirinya males nulis baru sekarang bisa tereksekusi, ya bisa dibilang ini topik mangkrak hampir 2 tahun 🤣 eh genap 2 tahun kayaknya karna dari tahun 2020 sampe 2022. curhatnya segini aja ga usah panjang panjang langsung ke point utamanya aja lah males nulis akunya.

## Apa sih GNU/Stow itu ?

GNU/Stow atau mungkin yang dikenal sebagai stow oleh kawan-kawan perias desktop, merupakan sebuah symlink farm manager atau lebih mudah dipahami dengan symlink manager sehingga pengguna tidak direpotkan lagi mengetikkan semua pathnya apabila ingin melakukan symlink dari folder asal ke folder tujuan.

Pada dasarnya stow diperuntukkan memanagement berkas dalam paket aplikasi namun tidak menutup kemungkinan stow ini digunakan untuk tujuan lain seperti memfasilitasi pengelolaan file konfigurasi yang ada di directori home pengguna, terutama jika digabungkan dengan `version control system(vcs)` seperti git atau yang lainnya.

Berdasarkan pengalamanku pakek stow yang dikombinasikan dengan git subtree untuk memanagement [dotfiles](https://github.com/mnabila/dotfiles) bener bener nyaman jika dibandingkan tanpa stow, karna aku tidak perlu lagi memindahkan konfig dari folder `~/.config` ke folder dotfiles lagi.

## Bagaimaa sih cara kerja GNU/stow itu ?

Cara kerja dari stow sendiri sebenernya cukup sederhana yakni stow akan membuatkan symlink file/folder secara dinamis jadi pengguna tidak perlu lagi mengetikkan sumber dan tujuan dari file yang akan dibuatkan symlinknya. Kunci untuk memahami cara kerja dari stow yakni terletak pada **struktur directory** yang ada dalam sumber tersebut.

## Implementasi

Sebelum mencoba mengimplementasikan si stow ini, perlu tahu terlebih dahulu mengenai **package**, **target directory** dan **stow directory**.

|      Kata kunci      | Penjelasan                                                                                                                               |
| :------------------: | :--------------------------------------------------------------------------------------------------------------------------------------- |
|     **package**      | merupakan sekumpulan folder atau file yang akan dikelola atau diiinstal ketika stow dijalankan.                                          |
| **target directory** | merupakan folder tujuan dimana si `package` akan diinstall, biasanya letak `target directory` ini adalah parent dirnya `stow directory`. |
|  **stow directory**  | merupakan root direktori yang berisikan paket-paket yang nanti akan diinstall menggunakan stow.                                          |

### Contoh Implementasi

Folder dotfiles berada di `/home/$USER/Dotfiles` sedangkan folder confignya berada di `/home/$USER/.config/zathura`, jadi bisa kita definisikan seperti dibawah ini.

|    Kata kunci    | Penjelasan              |
| :--------------: | :---------------------- |
|     package      | zathura                 |
| struktur package | zathura/.config/zathura |
| target directory | /home/$USER             |
|  stow directory  | /home/$USER/Dotfiles    |

jadi struktur folder packagenya adalah sebagai berikut ini.

```
$ tree -a /home/$USER/Dotfiles/zathura
/home/nabil/Dotfiles/zathura/
└── .config
└── zathura
└── zathurarc (ini file confignya)
2 directories, 1 file
```

Seperti yang sudah kusinggung di section sebelumnya yakni kuncinya terletak di struktur folder yang akan distow jadi isi dari folder packagenya mirip path yang akan dituju.

Setelah folder package dibuat maka dilanjutkan dengan tahap instalasi si package zathura, pastikan posisi pwd berada di folder `stow directory`

```
$ stow zathura
```

maka hasilnya seperti dibawah ini

```
$ ls -l /home/$USER/.config | grep zathura
lrwxrwxrwx 1 nabil users 35 Jan 23 2020 zathura -> ../Dotfiles/zathura/.config/zathura
```

Tapi bagaimana jika package ingin diinstall target instalasinya berbeda contoh `/etc/` sedangkan folder dotfilesnya berada di `/home/$USER/Dotfiles` ?

jadi bisa kita definisikan seperti dibawah ini.

|    Kata kunci    | Penjelasan           |
| :--------------: | :------------------- |
|     package      | mkinitcpio           |
| struktur package | mkinitcpio/etc       |
| target directory | /(root)              |
|  stow directory  | /home/$USER/Dotfiles |

Kenapa target directorynya berada di `/(root)` bukan mengarah ke `/etc` ?
Sebenernya bisa saja mengarah ke /etc akan tetapi terkadang kita lupa ini paket mengarah kemana maka dari itu minimal dikasi 1 subfolder dalam paket tersebut agar pengguna nanti tahu arah installasinya kemana.

dibawah ini contoh struktur dari paket mkinitcpio

```
❯ tree -a /home/$USER/Dotfiles/mkinitcpio/
/home/nabil/Dotfiles/mkinitcpio/
└── etc
└── mkinitcpio.conf
1 directories, 1 file
```

Jika target instalasinya berbeda dengan target directory kita cukup memainkan parameter `-t` yakni untuk mengubah target directorynya.

```
$ sudo stow -t / mkinitcpio
```

maka hasilnya seperti dibawah ini

```
❯ ls -l /etc/ | grep mkinitcpio
lrwxrwxrwx 1 root root 53 Jan 28 2020 mkinitcpio.conf -> ../home/nabil/Dotfiles/mkinitcpio/etc/mkinitcpio.conf
-rw-r--r-- 1 root root 2510 Des 3 17:11 mkinitcpio.conf.pacnew
drwxr-xr-x 2 root root 4096 Des 28 12:26 mkinitcpio.d
```

## Kesimpulan

Dengan adanya GNU/Stow ini pengguna sedikit terbantu karna tidak perlu lagi copy manual config dari folder dotfiles ke folder configurasi yang ada.

## Referensi

- manpage stow(man stow) Diakses pada: 2021-01-06
