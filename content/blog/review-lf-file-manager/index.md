+++
draft = false
date = '2022-01-11'
lastmod = '2026-03-15'
title = 'Review Lf File Manager'
type = 'blog'
description = 'Perbandingan lf dan ranger serta konfigurasi lf file manager sebagai pengganti ranger.'
image = ''
tags = ['tui', 'file manager', 'lf', 'ranger']
+++

## Latar Belakang

Selama lebih dari 2 tahun, **ranger** menjadi file manager utama saya di terminal. Ranger termasuk dalam kategori TUI (Terminal User Interface) file manager yang cukup populer di kalangan pengguna GNU/Linux. Workflow-nya sudah menyatu dengan kebiasaan sehari-hari -- navigasi cepat, preview file, dan integrasi dengan berbagai program eksternal.

## Permasalahan

Seiring waktu, masalah-masalah "ghaib" mulai muncul di ranger. File manager tiba-tiba freeze saat copy file besar, CPU usage mendadak 100% gara-gara file previewer, dan terkadang responsnya terasa lambat -- seolah mengajak santai di saat saya butuh yang sat set. Yang paling menjengkelkan, list file kadang tidak terupdate meskipun sudah di-restart, terutama saat mengakses removable disk.

Karena hal-hal ghaib itulah, rasa penasaran untuk mencoba alternatif lain mulai muncul.

## Pendekatan Solusi

Saya menjelajahi GitHub dan menemukan dua kandidat pengganti: **nnn** dan **lf**. Sebelum memutuskan, saya menetapkan tiga kriteria perbandingan: user interface (UI), file konfigurasi, dan workflow.

Dari ketiganya, **lf** yang paling cocok -- UI dan workflow-nya mirip ranger sehingga transisi tidak perlu mengubah workflow yang sudah ada, sementara it file konfigurasinya berbasis shell script yang justru lebih mudah jika dibanding Python-nya ranger.

### Perbandingan Lf vs Ranger

| Aspek | Lf | Ranger |
|:------|:---|:-------|
| **Fitur** | Builtin fitur sangat minim, tapi karena konfigurasinya berbasis shell script, pengguna bebas menambahkan fitur sesuai kebutuhan. | Fitur jauh lebih lengkap, tanpa debat. Bisa juga ditambah fitur baru yang ditulis dengan Python. Sayangnya, belum semua fiturnya sempat saya manfaatkan. |
| **Konfigurasi** | Default config minimalis dan mudah dipahami. Tapi di balik kesederhanaannya, tersimpan tantangan tersendiri. | Default config cukup banyak dan bisa membingungkan pemula yang ingin melakukan kustomisasi. |
| **Performa** | Lebih cepat dari ranger -- terasa hampir instan saat dibuka. | Cukup cepat, tapi sesekali bisa sangat lambat -- terutama saat mengakses removable disk. |
| **Kenyamanan** | Di awal terasa tidak nyaman, terutama untuk handle multiple file dalam satu instance. Butuh beberapa hari eksplorasi untuk menemukan solusinya. | Kenyamanan sudah tidak diragukan. Zona nyaman yang membuat saya betah hampir 2 tahun lebih. |
| **Tampilan** | Mirip ranger tapi tidak identik. Sayangnya kurang informatif saat proses copy/move -- hanya menampilkan persentase kecil di pojok, bukan progress bar seperti ranger. | Tampilan informasi proses file lebih lengkap dengan progress bar yang jelas. |

## Implementasi Teknis

Lf menggunakan **shell scripting** untuk file konfigurasinya. Karena saya menganut paham "konfig panjang wajib dimodulasi", konfigurasinya saya pisah menjadi beberapa bagian dan di-source ke dalam `lfrc`.

### lfrc (Main Config)

```sh
#!/bin/sh
# source another config
source ~/.config/lf/config.d/options
source ~/.config/lf/config.d/keymap
source ~/.config/lf/config.d/command
```

### Options

```sh
#!/bin/sh

set shell sh
set shellopts "-eu"
set ifs "\n"
set ignorecase true
set scrolloff 10
set icons
set ratios 1:2:3
set promptfmt "\033[32;1m%u\033[0m ❱ \033[34;1m%d\033[0m\033[1m%f\033[0m"
```

### Keymap

```sh
#!/bin/sh

map f fzf_select
map D delete
map x %dragon -a -x "$fx" 2&> /dev/null &
map R reload

# removable disk
map <f-12>u %udiskie-umount "$fx"
map <f-12>d %udiskie-umount "$fx" --force --detach
map <f-12>e %udiskie-umount "$fx" --force --eject
map <f-12>l %udiskie-umount "$fx" --lock

# iso file
map <f-11>m %udisksctl loop-setup --file "$f"
map <f-11>u %dmenu_iso &
map <f-11>c %dir2iso "$f"

# Directory
map gi cd /run/media/
map gm cd /mnt/

# split window
map <a-H> %tmux split -hb lf
map <a-J> %tmux split lf
map <a-K> %tmux split -vb lf
map <a-L> %tmux split -h lf

# select window
map <a-h> %tmux select-pane -L
map <a-j> %tmux select-pane -D
map <a-k> %tmux select-pane -U
map <a-l> %tmux select-pane -R
```

Integrasi tmux di keymap sangat membantu karena lf tidak memiliki fitur tab seperti ranger. Dengan tmux, saya bisa bermanuver antar panel menggunakan `Alt+H/J/K/L` untuk split dan `Alt+h/j/k/l` untuk navigasi antar pane.

### Command

```sh
#!/bin/sh

# file opener
cmd open ${{
    case $(file --mime-type "$f" -bL) in
        text/*) $EDITOR "$fx" ;;
        image/*) sxiv "$fx" 2&>/dev/null &;;
        application/x-iso9660-image) udisksctl loop-setup --file "$f";;
        application/x-executable) lf -remote "send $id echo 'not allowed'";;
        *) xdg-open "$f" &>/dev/null & ;;
    esac
}}

# create directory
cmd mkdir ${{
    printf "Directory Name: "
    read name
    if [ ! -z "$name" ]; then
        mkdir "$name"
    fi
}}

# create file
cmd touch ${{
    printf "File Name: "
    read name
    if [ ! -z "$name" ]; then
        $EDITOR "$name"
    fi
}}

# rename selected file/folder
cmd bulkrename ${{
    old="$(mktemp)"
    new="$(mktemp)"
    [ -n "$fs" ] && fs="$(ls)"
    printf '%s\n' "$fs" >"$old"
    printf '%s\n' "$fs" >"$new"
    $EDITOR "$new"
    [ "$(wc -l < "$new")" -ne "$(wc -l < "$old")" ] && exit
    paste "$old" "$new" | while IFS= read -r names; do
        src="$(printf '%s' "$names" | cut -f1)"
        dst="$(printf '%s' "$names" | cut -f2)"
        if [ "$src" = "$dst" ] || [ -e "$dst" ]; then
            continue
        fi
        mv -- "$src" "$dst"
    done
    rm -- "$old" "$new"
    lf -remote "send $id unselect"
}}

# fuzzy finder
cmd fzf_select ${{
    res="$(find -L . \( -path '*/\.*' -o -fstype 'dev' -o -fstype 'proc' \) -prune -o -print 2> /dev/null | sed 1d | cut -b3- | fzf +m)"
    if [ -d "$res" ]; then
        cmd="cd"
    else
        cmd="select"
    fi
    lf -remote "send $id $cmd \"$res\""
}}

# copy file to clipboard
cmd yank2clipboard ${{
    xclip -selection clipboard -i "$f" -t $(file --mime-type "$f" | cut -d ":" -f 2 | xargs)
}}

# print mimetype
cmd mimetype &{{
    mim=$(file --mime-type -bL "$f")
    lf -remote "send $id echomsg 'mimetype : $mim'" &
}}

# extract current file or selected files
cmd extract ${{
    set -f
    case "$f" in
        *.tar.bz|*.tar.bz2|*.tbz|*.tbz2) tar xjvf "$f";;
        *.tar.gz|*.tgz) tar xzvf "$f";;
        *.tar.xz|*.txz) tar xJvf "$f";;
        *.zip) unzip "$f";;
        *.rar) unrar x "$f";;
        *.7z) 7z x "$f";;
    esac
}}

# compress current file or selected files with tar and gunzip
cmd tar ${{
    set -f
    mkdir "$1"
    cp -r $fx "$1"
    tar czf "$1.tar.gz" "$1"
    rm -rf "$1"
}}

# compress current file or selected files with zip
cmd zip ${{
    set -f
    mkdir "$1"
    cp -r $fx "$1"
    zip -r "$1.zip" "$1"
    rm -rf "$1"
}}

# compress current file or selected files with 7z
cmd 7z ${{
    set -f
    mkdir "$1"
    cp -r $fx "$1"
    7z a "$1.7z" "$1"
    rm -rf "$1"
}}

cmd rclone ${{
    function rmount () {
        if [ ! -d "/tmp/rclone" ]; then
            mkdir /tmp/lfrclone
        fi
        rclone mount "$1" /tmp/lfrclone --daemon && notify-send "LF - Rclone" "Success Mount $1"
    }

    case $1 in
        m|mount) rmount "$2";;
        u|umount) fusermount -u /tmp/lfrclone && rmdir /tmp/lfrclone && notify-send "LF - Rclone" "Success Umount" ;;
        *) lf -remote "send $id echo example: rclone m remote:/path and rclone u"
    esac
}}
```

Beberapa command yang cukup berguna dalam workflow harian: `fzf_select` untuk navigasi cepat menggunakan fuzzy finder, `extract` untuk ekstraksi berbagai format arsip, dan `rclone` untuk mount cloud storage langsung dari file manager.

**Catatan penting:** Beberapa snippet dari wiki lf justru bisa menambah masalah. Contohnya pada `cmd open` -- secara default jika kita memilih beberapa file lalu membukanya, setiap file akan dibuka di instance terpisah. Pilih 10 video? Maka akan muncul 10 window video player, bukan satu playlist. Masalah ini saya bahas lebih detail di tulisan terpisah.

## Tantangan yang Dihadapi

Tantangan terbesar adalah melepaskan zona nyaman ranger. Setelah 2 tahun terbiasa, banyak muscle memory yang harus di-rewire. Beberapa fitur yang di ranger tinggal pakai, di lf harus dibuat sendiri dari nol -- mulai dari file opener, bulk rename, hingga integrasi dengan removable disk via udisks.

Selain itu, kurangnya progress bar yang informatif saat copy/move file besar cukup mengganggu. Lf hanya menampilkan persentase kecil di pojok layar, berbeda dengan ranger yang menampilkan progress bar lengkap.

## Insight dan Pembelajaran

Migrasi dari ranger ke lf mengajarkan beberapa hal penting:

Pertama, tool yang minimalis belum tentu inferior. Justru karena minim fitur bawaan, lf memberikan kebebasan total untuk membangun workflow sesuai kebutuhan tanpa harus berurusan dengan fitur-fitur yang tidak terpakai.

Kedua, konfigurasi berbasis shell script adalah pendekatan yang sangat powerful. Berbeda dengan konfigurasi Python di ranger yang butuh pemahaman API internal, di lf cukup menulis shell command biasa. Setiap orang yang familiar dengan terminal bisa langsung produktif.

Ketiga, ranger tidak benar-benar dipensiunkan -- lebih tepatnya dipindahkan ke bangku cadangan. Ranger masih berperan sebagai file chooser untuk qutebrowser dan file picker untuk neomutt, karena fitur ini belum tersedia di lf.

## Penutup

Lf berhasil menggantikan ranger sebagai file manager utama di workflow harian saya. Performanya lebih ringan, konfigurasinya lebih fleksibel, dan masalah-masalah "ghaib" yang sering muncul di ranger tidak lagi ditemui. Meskipun butuh effort lebih di awal untuk setup, hasilnya sepadan. Bagi mereka yang suka tantangan dan senang mengutak-atik konfigurasi, lf adalah pilihan yang sangat layak dicoba.

## Referensi

- [LF Wiki](https://github.com/gokcehan/lf/wiki) -- Diakses pada 2022-01-11
