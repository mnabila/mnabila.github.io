+++
draft = false
date = '2022-01-05'
lastmod = '2026-03-15'
title = 'Implementasi GNU Stow Untuk Management Dotfiles Di Archlinux'
type = 'blog'
description = 'Cara menggunakan GNU Stow sebagai symlink manager untuk mengelola dotfiles di Archlinux.'
image = ''
tags = ['stow']
+++

## Latar Belakang

Mengelola [dotfiles](https://github.com/mnabila/dotfiles) adalah rutinitas yang tidak bisa dihindari bagi pengguna GNU/Linux yang gemar melakukan kustomisasi. Setiap kali mengubah konfigurasi -- entah itu neovim, zathura, atau tmux -- perubahan tersebut idealnya langsung ter-track di version control. Saya sudah menggunakan git untuk mengelola dotfiles, tapi workflow-nya masih manual: copy file dari `~/.config` ke folder dotfiles, commit, lalu push. Prosedur ini rentan terlupa dan cukup merepotkan kalau dilakukan berulang kali.

Topik ini sebenarnya sudah lama ingin saya tulis, tapi rasa malas yang lebih kuat dari niat menulis membuatnya mangkrak selama 2 tahun -- dari 2020 sampai akhirnya bisa tereksekusi di 2022.

## Permasalahan

Workflow manual copy-paste dotfiles memiliki beberapa masalah:

- File konfigurasi asli di `~/.config` dan salinannya di folder dotfiles bisa saling menyimpang jika lupa disinkronkan.
- Menentukan path sumber dan tujuan satu per satu untuk setiap file sangat tedious.
- Risiko lupa meng-copy perubahan terbaru sebelum commit ke git cukup tinggi.

Saya butuh mekanisme yang membuat file konfigurasi di `~/.config` dan di folder dotfiles selalu identik -- tanpa harus copy secara manual.

## Pendekatan Solusi

Solusi yang saya pilih adalah **GNU Stow** -- sebuah *symlink farm manager*. Alih-alih menyimpan salinan file konfigurasi di dua tempat, stow membuat symlink dari lokasi tujuan ke sumber di folder dotfiles. Dengan begitu, file yang ada di `~/.config/zathura` sebenarnya adalah symlink yang mengarah ke `~/Dotfiles/zathura/.config/zathura`. Perubahan di salah satu otomatis terefleksi di keduanya karena pada dasarnya itu file yang sama.

Saya mengkombinasikan stow dengan git subtree untuk mengelola seluruh dotfiles. Hasilnya jauh lebih nyaman dibandingkan workflow manual sebelumnya.

## Implementasi Teknis

Sebelum masuk ke implementasi, ada tiga konsep penting dalam stow:

|      Istilah         | Penjelasan                                                                                                                               |
| :------------------: | :--------------------------------------------------------------------------------------------------------------------------------------- |
| **Package**          | Sekumpulan folder atau file yang akan dikelola ketika stow dijalankan.                                                                   |
| **Target Directory** | Folder tujuan di mana package akan di-install. Biasanya merupakan parent directory dari stow directory.                                  |
| **Stow Directory**   | Root direktori yang berisi paket-paket yang akan di-install menggunakan stow.                                                            |

### Contoh: Symlink Konfigurasi Zathura

Skenario pertama -- folder dotfiles di `/home/$USER/Dotfiles`, konfigurasi zathura di `/home/$USER/.config/zathura`:

|    Istilah       | Nilai                   |
| :--------------: | :---------------------- |
| Package          | zathura                 |
| Struktur Package | zathura/.config/zathura |
| Target Directory | /home/$USER             |
| Stow Directory   | /home/$USER/Dotfiles    |

Kunci utamanya ada di struktur folder package yang harus menyerupai path tujuan:

```
$ tree -a /home/$USER/Dotfiles/zathura
/home/nabil/Dotfiles/zathura/
└── .config
    └── zathura
        └── zathurarc
2 directories, 1 file
```

Setelah struktur folder siap, jalankan stow dari dalam stow directory:

```
$ stow zathura
```

Hasilnya:

```
$ ls -l /home/$USER/.config | grep zathura
lrwxrwxrwx 1 nabil users 35 Jan 23 2020 zathura -> ../Dotfiles/zathura/.config/zathura
```

### Contoh: Target Instalasi di /etc

Skenario kedua -- bagaimana jika target bukan di home directory melainkan di `/etc/`? Misalnya untuk `mkinitcpio.conf`:

|    Istilah       | Nilai                |
| :--------------: | :------------------- |
| Package          | mkinitcpio           |
| Struktur Package | mkinitcpio/etc       |
| Target Directory | / (root)             |
| Stow Directory   | /home/$USER/Dotfiles |

Target directory diset ke `/` dan subfolder `etc` disertakan di dalam package, sehingga hanya dengan melihat struktur folder kita langsung tahu ke mana arah instalasinya:

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

## Tantangan yang Dihadapi

Tantangan utama ada di pemahaman konsep awal. Hubungan antara package, target directory, dan stow directory tidak langsung intuitif -- butuh beberapa kali percobaan sebelum benar-benar memahami bagaimana stow menentukan lokasi symlink berdasarkan struktur folder.

Selain itu, ketika target instalasinya di luar home directory (seperti `/etc`), diperlukan `sudo` dan parameter `-t` yang harus diingat. Salah menentukan target directory bisa mengakibatkan symlink dibuat di lokasi yang tidak diinginkan.

Perlu diperhatikan juga bahwa stow akan menolak membuat symlink jika file tujuan sudah ada dan bukan symlink. Dalam kasus ini, file asli perlu dipindahkan atau dihapus terlebih dahulu sebelum stow bisa berjalan.

## Insight dan Pembelajaran

GNU Stow mengubah cara saya mengelola dotfiles secara fundamental. Dari yang sebelumnya harus ingat-ingat mana file yang sudah di-copy dan mana yang belum, sekarang cukup `stow <package>` dan selesai. Kombinasi stow + git subtree memberikan workflow yang solid -- setiap perubahan konfigurasi otomatis ter-track tanpa langkah manual tambahan.

Satu insight penting: konsep "struktur folder package mengikuti path tujuan" adalah desain yang sangat elegan. Ini membuat konfigurasi stow bersifat self-documenting -- hanya dengan melihat struktur folder, kita langsung tahu ke mana file tersebut akan di-symlink.

## Penutup

Dengan GNU Stow, mengelola dotfiles menjadi jauh lebih praktis dan reliable. Tidak perlu lagi copy-paste manual -- cukup atur struktur folder dengan benar, dan biarkan stow yang mengurus sisanya. Bagi pengguna Archlinux atau distro apapun yang sering mengutak-atik konfigurasi, stow adalah tool yang sangat layak dicoba.

## Referensi

- manpage stow (`man stow`) -- Diakses pada 2021-01-06
