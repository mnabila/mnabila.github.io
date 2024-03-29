---
title: "Lf Si File Manager Baruku"
date: 2022-01-11T08:27:24+07:00
categories: ["terminal", "application"]
tags: ["tui", "file manager", "lf", "ranger"]
lastmod: 2022-01-15T12:43:24+07:00
---

# alasan selingkuh

Berawal dari temen sekarang jadi mantan, kalimat ini sepertinya cocok untuk menggambarkan pergantian file manager yang kugunakan saat ini. Sebelumnya aku menggunakan ranger untuk file manager utamaku, akan tetapi sewaktu menggunakan ranger aku sering mengalami hal-hal ghaib seperti file manager berhenti sendiri sewaktu copy file, cpu usage tiba-tiba naik 100% gara gara file previewer dan tak hanya itu terkadang si ranger ini agak males klok disuruh gerak cepet maunya ngajakin santai.

Dikarenakan beberapa hal ghaib itu tiba-tiba setan dari pedalaman dotfiles menghantuiku untuk mencari selingkuhan baru, karna imanku tipis alhasilnya aku coba jalan jalan di gang github siapa tahu dapet gebetan baru dan bump dapet 2 calon (hiahiahia), calon pertama `nnn` dan calon yang kedua adalah `lf`.

Sebelum memutuskan pilih yang mana, aku buatkan 3 kriteria meliputi user interface(UI), file konfigurasinya, dan workflownya. Dari ketika kriteria ini lf adalah yang paling pas denganku karna dari UI dan workflownya lf cenderung mirip dengan ranger, sedangkan untuk file Konfigurasinya si lf ini cukup menarik jika dibandingkan ranger dan nnn.

# Kelebihan adalah kekurangan

Kelebihannya banyak kekurangannya, yups tapi ini beneran bukan becandaan. Lf ini bisa kubilang file manager minimalist keduaku setelah si mc(midnight commander) jika dibandingkan dengan ranger yang kupakek. Karna secara default lf hanya menyediakan fitur alakadarnya saja ga ada yang magic yang membuatku tekesan. Tapi karna minimnya fitur yang disediakan oleh lf pengguna dibebaskan membuat fitur sendiri sesuai dengan yang mereka inginkan yang bisa dituliskan melalui file konfignya. Dibawah ini adalah beberapa gambaran kenapa aku pindah dari ranger ke lf.

# Perbandingan lf dan ranger
LF | Ranger
------------ | -------------
**fitur**
Fitur ? builtin fiturnya sedikit banget, tapi karna file konfignya bisa ditulis dengan shell script pengguna memungkinkan menambahkan fitur yang sesuai dengan keinginan pengguna. | Jelas secara fitur ranger lebih lengkap daripada lf no debat, akan tetapi dibalik kelengkapan dari fitur ranger aku masih belum bisa memanfaatkan semua fiturnya. belum lagi si ranger ini juga bisa ditambahkan fitur baru oleh penggunanya yang ditulis dengan bahasa pemrograman python.
**konfig**
default konfignya minimalist dan mudah dipahami, akan tetapi dibaliknya tedapat sesuatu yang tidak bisa diungkapkan dengan kata kata (haha) | default konfignya mayan banyak dan agak membingungkan pemula jika ingin diutak atik lebih lanjut
**Performa**
Secara performa lf lebih cepat dibandingkan ranger tapi aku ga berani bilang sangat cepat walaupun terasa ga sampe sedetik udah kebuka ini file manager 🤣. Tapi karna masi baru beberapa hari pakek aku belum bisa berkomentar banyak soal ini file manager (nanti akan diupdate jika sudah mulai muncul hal hal ghaib di lf) | Performa dari ranger sendiri cukup cepat menurutku akan tetapi terkadang bisa sangat lambat apalagi klok ketemu removabledisk seperti flashdrive terkadang list file yang ditampilkan tidak sesua, walaupun sudah direstart sekalipun list filenya tidak terupdate (solusi buka tutup ranger)
**Kenyamana**
Dari segi kenyamanan jelas lf diawal tidak nyaman sama sekali terutama untuk handle multiple file yang dibuka dalam satu instance, aku sendiri udah beberapa hari mencari pendekatan yang tepat untuk mengatasi masalah ini. | Kenyaman di ranger udah ga perlu diragukan lagi buatku karna aku terjebak zona nyaman sama dia hampir 2 tahun lebih kayaknya aku pakek ini file manager.
**Tampilan**
Kembar tapi tak sama, keduanya sama sama bagus. Akan tetapi untuk lf kurang informatif untuk proses memindahkan dan menyalin filenya (cuman muncul persentase dipojokan bukan bar mirip ranger)

# Konfigurasi

Seperti yang sudah kusinggung di intro, lf ini menggunakan shell scripting untuk file konfigurasinya. Penggunaan shell scripting ini menurutku suatu kelebihan tersendiri dikarenakan dengan shell scripting pengguna bisa bebas menambahkan command kedalam file manager ini. Untuk mendapatkan default file konfigurasi lf kalian bisa ambil [disini](https://github.com/gokcehan/lf/blob/master/etc/lfrc.example).

Dibawah ini adalah beberapa potongan konfigurasi yang aku gunakan, dikarenakan aku menganut paham konfig panjang wajib dimodularin maka aku pisah menjadi beberapa bagian meliputi options, keymap, command kemudian di source kedalam lfrc.

## keymap

```
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

## command

```
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

## options
```
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

## lfrc

```
#!/bin/sh
# source another config
source ~/.config/lf/config.d/options
source ~/.config/lf/config.d/keymap
source ~/.config/lf/config.d/command
```

**Catatan:** potongan konfigurasi diatas didapatkan dari eksplorasiku di lf -doc dan wiki akan tetapi ada beberapa snippet dari wiki lf yang menurutku itu bukanlah sebuah tips tapi malah nambah masalah jadi mau ga mau harus eksporasi sendiri. contohnya ? cmd open bagian all file, klok kita pilih beberapa file dan ingin dibuka dalam single instance maka jatuhnya malah tiap file jadi satu instance jadi klok kita select 10 video maka nanti akan muncul 10 window video player bukan menjadi sebuah playlist

# Apakah ranger akan ditinggalkan ?

Ya enggak dong walaupun udah mantan, tapi kita harus tetep menjaga silaturahmi. Saat ini si ranger kumanfaatkan sebagai file manager keduaku siapa tahu nanti bakalan CLBK lagi (hiahiahia). Selain dimanfaatkan sebagai file manager kedua, ranger juga kumanfaatkan sebagai file chooser untuk qutebrowserku dan file picker untuk neomuttku karna di lf sendiri fitur ini masih belum ada.

# Preview lf

video::./video/video-220111-2106-00.mp4[width=700]
Dikarenakan lf tidak mengusung fitur tabnya si ranger jadi kita memerlukan tmux untuk bermanuver seperti diatas.

# Kesimpulan

Lf si file manager yang penuh dengan kejutan, banyak fitur yang belum tersedia jadi pengguna dipaksa membuat fitur sendiri sesuai dengan keinginannya.
Lf ini cocok buat mereka yang suka tantangan dan memiliki pemikiran yang kreatif seperti penulis 🤣.

# Referensi

- [LF wiki](https://github.com/gokcehan/lf/wiki) Diakses pada 2022-01-11
