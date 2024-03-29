---
title: "Berkenalan Dengan tmux"
date: 2019-05-27T16:25:05+07:00
categories: ["terminal", "application"]
tags: ["tmux"]
---

## intro
Tmux atau Terminal Multiplexer adalah sebuah terminal yang memungkinkan kita membuka banyak sesi terminal dalam satu window, sehingga dalam satu window ini pengguna memungkinkan untuk menjalankan banyak aplikasi sekaligus. Bagiku tmux ini sangat membantu apabila sedang berurusan dengan banyak aplikasi dalam sebuah  tty alias `the real true terminal` yang dimiliki oleh sistem operasi GNU/Linux 😆.

## Permasalahan
Untuk pengguna baru tmux rata-rata kesulitan untuk mengakses key bindingnya dikarenakan belum terlalu hafal bahkan belum tahu tentang key binding yang ada di tmux. Sehingga tmux memiliki kesan cukup merepotkan atau susah untuk pengguna baru padahal tmux adalah salah satu inverstasi tools yang dapat memudahkan aku untuk menjalankan banyak program dalam satu sesi terminal.

## Instalasi
untuk instalasi tmux cukup ketikkan perintah dibawah ini (jika kamu menggunakan distro archlinux dan turunannya)

```
$ sudo pacman -S tmux
```

## Konfigurasi
Setelah berhasi memasang paket aplikasi tmux, tahap selanjutnya yakni melakukan konfigurasi, tujuannya untuk memudahkan kita saat menggunakannya nanti serta enak dilihat dan tidak terkesan biasa.
adapun yang akan dikonfigurasi yakni :

* Ubah default prefix key
* Ubah tampilan status bar

Pada umumnya file konfigurasi tmux berada di `~/.tmux.conf` untuk setiap user, sedangkan untuk configurasi global berada di `/etc/tmux.conf`.

### Konfigurasi default prefix key
Secara default prefix key-nya menggunakan Ctrl-b, sebagai contoh saat kita ingin memuat tampilan tmux menjadi dual sesi dengan mode split kita harus menekan prefix kemudian diikuti dengan `%` atau `Ctrl-b %`. Akan tetapi apabila belum terbiasa akan cukup sulit mengingat posisinya cukup berjauhan sehingga membutuhkan waktu lebih apabila ingin menggunakan keybinding.sehingga aku mengggantinya menjadi Ctrl-a agar memudahkan aku ketika mengakses prefix tersebut. Untuk mengubah key binding menjadi Ctrl-a cukup tambahkan kode dibawah ini pada file .tmux.conf-nya

```
unbind C-b
set -g prefix C-a
bind C-a send-prefix
```

### Konfigurasi status bar
Tampilan awal status bar dari Tmux cukup sederhana dan terkesan biasa, maka dari itu saya melakukan perubahan sedikit sehingga enak dilihat dan tidak terkesan monoton. sehingga saya menambhkan potongan kode berikut ini kedalam file .tmux.conf-nya

```
# STATUS
set -g status-position bottom
set -g status on
set -g status-interval 60
set -g status-style "fg=brightwhite, bg=black"

## Left
set -g status-left-length 40
set -g status-left "#[fg=black,bg=yellow, bold]   #[fg=black,bg=white, bold] #(whoami) "

## Center
set -g window-status-format "#[fg=black,bg=brightblack] #I #{pane_current_command} "
set -g window-status-current-format "#[fg=black,bg=yellow, bold] #I #{pane_current_command} "
set -g window-status-separator " "
set -g status-justify centre

## Right
set -g status-right-length 40
set -g status-right "#{prefix_highlight} #[fg=black,bg=yellow, bold]   #[fg=black,bg=white, bold] #(lsb_release -d | cut -f 2) "
```

bonus untuk judul pada terminal yang dibuka
```
# WINDOW
set -g base-index 1
set -g renumber-windows on
setw -g automatic-rename on
setw -g window-style "fg=white bg=black"
setw -g window-active-style "fg=brightwhite bg=black"
```

## Hasil
setelah potongan kode di atas ditambahkan kedalam `.tmux.conf` maka hasilnya akan seperti berikut ini.
![hasil](img/hasil.png "hasil")
ps: hasil ini diambil dari `terminal emulator` bukan dari tty langsung, kenapa ? aku ga tahu cara screenshot di tty gimana 🤣


## Referensi
* Tmux manpages(man tmux) Diakses pada: 2019-05-25

* [panduan-minimal-belajar-tmux](https://agung-setiawan.com/panduan-minimal-belajar-tmux/) Diakses pada: 2019-05-25

* [Archwiki Tmux](link:https://wiki.archlinux.org/index.php/tmux) Diakses pada: 2019-05-25
