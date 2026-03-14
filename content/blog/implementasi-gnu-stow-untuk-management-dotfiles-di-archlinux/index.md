+++
draft = false
date = '2022-01-05'
title = 'Implementasi GNU Stow Untuk Management Dotfiles Di Archlinux'
type = 'blog'
description = 'Cara menggunakan GNU Stow sebagai symlink manager untuk mengelola dotfiles di Archlinux.'
image = ''
tags = ['stow']
+++

## Curhat Sebentar

Topik ini sebenarnya sudah lama ingin saya bahas, tapi karena rasa malas yang lebih kuat dari niat menulis, baru sekarang bisa tereksekusi. Ya, kurang lebih mangkrak selama 2 tahun -- dari 2020 sampai 2022. Sudah cukup curhatnya, langsung masuk ke inti pembahasan.

## Apa Itu GNU Stow?

**GNU Stow** -- atau yang sering disebut `stow` saja oleh para penggiat ricing desktop -- adalah sebuah *symlink farm manager*. Sederhananya, stow membantu kita membuat symlink secara otomatis tanpa perlu repot mengetikkan path lengkap dari sumber ke tujuan.

Awalnya stow memang dirancang untuk mengelola berkas dalam paket aplikasi, tapi ternyata sangat cocok juga untuk mengelola file konfigurasi di direktori home. Apalagi kalau dikombinasikan dengan version control system seperti Git.

Dari pengalaman saya, menggunakan stow yang dikombinasikan dengan git subtree untuk mengelola [dotfiles](https://github.com/mnabila/dotfiles) terasa jauh lebih nyaman dibandingkan tanpa stow. Tidak perlu lagi bolak-balik copy config dari folder `~/.config` ke folder dotfiles secara manual.

## Bagaimana Cara Kerja GNU Stow?

Cara kerjanya cukup sederhana: stow akan membuatkan symlink file dan folder secara dinamis berdasarkan **struktur direktori** yang ada di dalam sumber. Jadi kita tidak perlu lagi menentukan sumber dan tujuan satu per satu -- cukup atur struktur foldernya dengan benar, dan stow yang mengurus sisanya.

## Implementasi

Sebelum mulai, ada tiga konsep penting yang perlu dipahami:

|      Istilah         | Penjelasan                                                                                                                               |
| :------------------: | :--------------------------------------------------------------------------------------------------------------------------------------- |
| **Package**          | Sekumpulan folder atau file yang akan dikelola ketika stow dijalankan.                                                                   |
| **Target Directory** | Folder tujuan di mana package akan di-install. Biasanya merupakan parent directory dari stow directory.                                  |
| **Stow Directory**   | Root direktori yang berisi paket-paket yang akan di-install menggunakan stow.                                                            |

### Contoh: Symlink Konfigurasi Zathura

Misalkan folder dotfiles berada di `/home/$USER/Dotfiles` dan konfigurasi zathura ada di `/home/$USER/.config/zathura`. Maka konfigurasinya sebagai berikut:

|    Istilah       | Nilai                   |
| :--------------: | :---------------------- |
| Package          | zathura                 |
| Struktur Package | zathura/.config/zathura |
| Target Directory | /home/$USER             |
| Stow Directory   | /home/$USER/Dotfiles    |

Struktur folder package-nya harus menyerupai path tujuan:

```
$ tree -a /home/$USER/Dotfiles/zathura
/home/nabil/Dotfiles/zathura/
└── .config
    └── zathura
        └── zathurarc
2 directories, 1 file
```

Perhatikan bahwa isi folder package ini mencerminkan path yang akan dituju -- inilah kunci utama cara kerja stow.

Setelah struktur folder siap, jalankan stow dari dalam stow directory:

```
$ stow zathura
```

Hasilnya:

```
$ ls -l /home/$USER/.config | grep zathura
lrwxrwxrwx 1 nabil users 35 Jan 23 2020 zathura -> ../Dotfiles/zathura/.config/zathura
```

### Contoh: Target Instalasi Berbeda

Bagaimana kalau target instalasi bukan di home directory, melainkan di `/etc/`? Misalnya untuk file `mkinitcpio.conf`:

|    Istilah       | Nilai                |
| :--------------: | :------------------- |
| Package          | mkinitcpio           |
| Struktur Package | mkinitcpio/etc       |
| Target Directory | / (root)             |
| Stow Directory   | /home/$USER/Dotfiles |

Kenapa target directory-nya `/` dan bukan langsung `/etc`? Karena dengan menyertakan subfolder `etc` di dalam package, kita langsung tahu ke mana arah instalasinya hanya dengan melihat struktur folder.

Struktur package-nya:

```
$ tree -a /home/$USER/Dotfiles/mkinitcpio/
/home/nabil/Dotfiles/mkinitcpio/
└── etc
    └── mkinitcpio.conf
1 directory, 1 file
```

Gunakan parameter `-t` untuk menentukan target directory yang berbeda:

```
$ sudo stow -t / mkinitcpio
```

Hasilnya:

```
$ ls -l /etc/ | grep mkinitcpio
lrwxrwxrwx 1 root root 53 Jan 28 2020 mkinitcpio.conf -> ../home/nabil/Dotfiles/mkinitcpio/etc/mkinitcpio.conf
-rw-r--r-- 1 root root 2510 Des 3 17:11 mkinitcpio.conf.pacnew
drwxr-xr-x 2 root root 4096 Des 28 12:26 mkinitcpio.d
```

## Kesimpulan

Dengan GNU Stow, mengelola dotfiles jadi jauh lebih praktis. Tidak perlu lagi copy-paste manual antara folder konfigurasi dan folder dotfiles -- cukup atur struktur folder dengan benar, dan biarkan stow yang bekerja.

## Referensi

- manpage stow (`man stow`) -- Diakses pada 2021-01-06
