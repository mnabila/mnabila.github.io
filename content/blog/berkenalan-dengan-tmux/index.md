+++
draft = false
date = '2019-05-27'
lastmod = '2026-03-15'
title = 'Berkenalan Dengan tmux'
type = 'blog'
description = 'Pengenalan dan konfigurasi dasar tmux untuk membuka banyak sesi terminal dalam satu window.'
image = ''
tags = ['tmux']
+++

## Latar Belakang

Sebagian besar waktu kerja saya di GNU/Linux dihabiskan di terminal. Ada kalanya saya bekerja langsung di TTY -- *the real true terminal* -- tanpa desktop environment. Di situasi seperti ini, kemampuan untuk menjalankan beberapa program sekaligus dalam satu layar menjadi kebutuhan yang sangat mendasar. Tidak ada tab browser, tidak ada window manager yang bisa di-split -- yang ada hanya satu layar terminal polos.

Di sinilah **tmux** (Terminal Multiplexer) masuk. Tmux memungkinkan kita membuka banyak sesi terminal dalam satu window, melakukan split pane, dan bahkan detach session yang bisa di-attach kembali nanti.

## Permasalahan

Meskipun tmux sangat powerful, pengalaman pertama menggunakannya cukup membuat frustrasi. Key binding default-nya terasa asing -- prefix key `Ctrl-b` yang posisinya berjauhan di keyboard, tampilan status bar yang polos tanpa informasi berguna, dan kebiasaan navigasi yang harus dibangun dari nol. Banyak yang menyerah di tahap awal karena learning curve ini.

Saya perlu mengkustomisasi tmux agar lebih nyaman digunakan sehari-hari sebelum bisa benar-benar produktif dengan tool ini.

## Pendekatan Solusi

Ada dua hal utama yang ingin saya sesuaikan: **prefix key** yang lebih ergonomis dan **status bar** yang lebih informatif. Pendekatan saya sederhana -- mulai dari konfigurasi minimal yang mengatasi friction terbesar, lalu perlahan menambahkan kustomisasi seiring kebutuhan. File konfigurasi tmux berada di `~/.tmux.conf` untuk tiap user, sedangkan konfigurasi global ada di `/etc/tmux.conf`.

Untuk instalasi di Archlinux:

```
$ sudo pacman -S tmux
```

## Implementasi Teknis

### Mengganti Prefix Key

Prefix key default `Ctrl-b` saya ganti menjadi `Ctrl-a`. Alasannya praktis -- posisi tombol `Ctrl` dan `a` berdekatan sehingga lebih mudah dijangkau dengan satu tangan:

```
unbind C-b
set -g prefix C-a
bind C-a send-prefix
```

### Konfigurasi Status Bar

Tampilan default status bar tmux sangat monoton dan kurang informatif. Saya merancang status bar yang menampilkan informasi username, nama window aktif, dan informasi distribusi OS:

```
# STATUS
set -g status-position bottom
set -g status on
set -g status-interval 60
set -g status-style "fg=brightwhite, bg=black"

## Left
set -g status-left-length 40
set -g status-left "#[fg=black,bg=yellow, bold]   #[fg=black,bg=white, bold] #(whoami) "

## Center
set -g window-status-format "#[fg=black,bg=brightblack] #I #{pane_current_command} "
set -g window-status-current-format "#[fg=black,bg=yellow, bold] #I #{pane_current_command} "
set -g window-status-separator " "
set -g status-justify centre

## Right
set -g status-right-length 40
set -g status-right "#{prefix_highlight} #[fg=black,bg=yellow, bold]   #[fg=black,bg=white, bold] #(lsb_release -d | cut -f 2) "
```

### Konfigurasi Window

Beberapa pengaturan tambahan untuk window agar lebih intuitif -- index dimulai dari 1 (bukan 0), window otomatis di-rename sesuai program yang berjalan, dan visual distinction antara window aktif dan tidak aktif:

```
# WINDOW
set -g base-index 1
set -g renumber-windows on
setw -g automatic-rename on
setw -g window-style "fg=white bg=black"
setw -g window-active-style "fg=brightwhite bg=black"
```

## Tantangan yang Dihadapi

Tantangan terbesar bukan di sisi teknis konfigurasi, melainkan membangun kebiasaan baru. Setelah bertahun-tahun menggunakan terminal tanpa multiplexer, jari-jari harus dilatih ulang untuk mengingat prefix key dan shortcut navigasi antar pane. Butuh sekitar satu minggu penggunaan konsisten sebelum tmux terasa natural.

Selain itu, menemukan kombinasi warna dan informasi yang tepat untuk status bar memerlukan beberapa kali iterasi. Format string tmux yang menggunakan `#[]` untuk styling tidak begitu intuitif dan dokumentasinya harus sering dirujuk.

## Insight dan Pembelajaran

Tmux adalah salah satu investasi tools terbaik untuk produktivitas di terminal. Begitu muscle memory terbentuk, workflow berubah drastis -- tidak perlu lagi buka-tutup terminal, session bisa di-detach dan di-attach kembali (sangat berguna saat SSH ke remote server), dan split pane memungkinkan monitoring sambil bekerja.

Kunci untuk melewati fase awal yang membingungkan adalah memulai dengan konfigurasi minimal. Tidak perlu langsung menghafal semua shortcut -- cukup prefix key, split pane, dan navigasi antar pane. Sisanya bisa dipelajari secara bertahap.

## Penutup

Setelah konfigurasi di atas diterapkan, tmux sudah jauh lebih nyaman digunakan dibanding setup default. Hasilnya terlihat seperti ini:

![hasil](img/hasil.png "hasil")

*PS: screenshot ini diambil dari terminal emulator, bukan langsung dari TTY -- soalnya saya belum tahu cara screenshot di TTY.*

Ke depannya, konfigurasi ini masih bisa dikembangkan -- misalnya menambahkan plugin via TPM (Tmux Plugin Manager) atau mengintegrasikan tmux dengan workflow lain seperti vim dan fzf.

## Referensi

- Tmux manpages (`man tmux`) -- Diakses pada 2019-05-25
- [Panduan Minimal Belajar Tmux](https://agung-setiawan.com/panduan-minimal-belajar-tmux/) -- Diakses pada 2019-05-25
- [Archwiki Tmux](https://wiki.archlinux.org/index.php/tmux) -- Diakses pada 2019-05-25
