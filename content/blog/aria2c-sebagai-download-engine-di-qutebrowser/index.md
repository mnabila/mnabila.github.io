+++
draft = false
date = '2022-01-06'
lastmod = '2026-03-15'
title = 'Aria2c Sebagai Download Engine Di Qutebrowser'
type = 'blog'
description = 'Menggunakan aria2c sebagai download engine di qutebrowser melalui userscript dan monkeyscript.'
image = ''
tags = ['browser', 'download', 'cli', 'userscript', 'monkeyscript']
+++

## Latar Belakang

Sebagai pengguna **Qutebrowser** -- browser yang mengandalkan keyboard untuk navigasi -- saya sudah cukup lama merasa kurang puas dengan kemampuan download bawaannya. Qutebrowser memang bukan browser mainstream; dibangun menggunakan Python dengan module PyQt5, backend-nya qt5-webengine, dan tampilannya sangat minimalis. Bisa dibilang, pengguna qutebrowser punya selera tersendiri yang rela menghafal keymap demi kenyamanan navigasi tanpa mouse.

Di sisi lain, saya sudah menggunakan **aria2c** sebagai download manager utama di terminal. Aria2c adalah download manager berbasis CLI yang mendukung berbagai protokol -- HTTP(S), FTP, SFTP, BitTorrent, hingga Metalink. Cukup satu tool ini saja dan tidak perlu lagi install torrent client terpisah. Yang lebih menarik, aria2c bisa dijadikan download service yang diakses dari berbagai frontend, baik GUI seperti [UGET](https://ugetdm.com/) dan [Persepolis](https://persepolisdm.github.io/), web seperti [AriaNg](https://github.com/mayswind/AriaNg) dan [webui-aria2](https://github.com/ziahamza/webui-aria2), hingga TUI seperti [aria2p](https://github.com/pawamoy/aria2p/).

Muncul pertanyaan sederhana: bagaimana kalau dua tool ini digabungkan?

## Permasalahan

Download bawaan qutebrowser cukup terbatas -- tidak mendukung resume, tidak bisa multi-connection, dan tidak ada integrasi dengan download manager eksternal secara default. Sementara aria2c sudah berjalan sebagai service di background via JSON-RPC, tinggal mencari cara mengirimkan URL dari qutebrowser ke aria2c.

## Pendekatan Solusi

Saya menemukan dua pendekatan untuk mengintegrasikan keduanya: **userscript** dan **monkeyscript** (Greasemonkey).

Userscript dipilih untuk kebutuhan manual -- ketika saya ingin memilih link tertentu di halaman web lalu mengirimnya ke aria2c. Keunggulan userscript di qutebrowser adalah bisa ditulis dalam bahasa pemrograman apapun selama mampu membaca environment variable yang disediakan qutebrowser. Daftar lengkap environment variable-nya tersedia di [qutebrowser/userscript](https://qutebrowser.org/doc/userscripts.html).

Monkeyscript dipilih untuk kebutuhan otomatis -- khususnya di situs file hosting tertentu yang ingin langsung dikirim ke aria2c tanpa interaksi manual.

## Implementasi Teknis

### UserScript: qb2aria

Saya menulis userscript sederhana bernama `qb2aria` yang mengirimkan URL aktif ke aria2c melalui JSON-RPC:

```bash
#!/bin/bash

ARIA2URI="http://localhost:6800/jsonrpc"

function datagen() {
    jq -n --arg id "$(date +%N)" --arg url "$1" \
        '{ "jsonrpc": "2.0", "id": $id, "method":"aria2.addUri", "params":[[$url]], }'
}

if [[ ! -z $QUTE_URL ]]; then
    notify-send "Aria2 | Downloading" "$QUTE_URL" -u normal

    data=$(datagen "$QUTE_URL")
    curl -s --request POST --url $ARIA2URI --header 'Content-Type: application/json' --data "$data"
fi
```

| Komponen    | Fungsi                                                                                                  |
| :---------: | :------------------------------------------------------------------------------------------------------ |
| `datagen`   | Menggenerate payload JSON yang dibutuhkan untuk mengirim request download ke aria2c melalui curl         |
| `$QUTE_URL` | Environment variable bawaan qutebrowser yang berisi URL halaman aktif, hanya tersedia di dalam userscript |

File disimpan di `/home/$USER/.config/qutebrowser/userscripts` dan diberi hak akses eksekusi:

```
$ chmod +x qb2aria
```

Kemudian ditambahkan keymap di `config.py` qutebrowser:

```python
config.bind(",d", "hint links userscript qb2aria", "normal")
```

Workflow-nya: tekan `,` lalu `d`, pilih hint link yang ingin di-download. Selesai.

### Monkeyscript: Monkey D Aria2

Untuk otomasi di situs file hosting, saya menulis monkeyscript yang mendeteksi hostname dan langsung mengirimkan direct link ke aria2c:

```javascript
// ==UserScript==
// @name                Monkey D Aria2
// @version             1.0.0
// @description         Automatically download file from file hosting with aria2-RCP.
// @author              mnabila
// @namespace           https://github.com/mnabila
// @license             MIT License
// @icon                https://github.com/mnabila/monkeydscript/raw/main/asset/slime.svg
// @match               https://*.zippyshare.com/*
// @match               https://anonfiles.com/*
// @match               http://www.solidfiles.com/*
// @match               https://www.mediafire.com/*
// @grant               none
// ==/UserScript==

(function () {
    'use strict';

    function sendNotification(body) {
        if (Notification.permission != 'granted') {
            Notification.requestPermission();
        }
        let notif = new Notification('Aria2 | Download', { body: body });
    }

    function download(url) {
        // TODO: fix cors, i think use mode no-cors is not fix this problem
        const uri = 'http://localhost:6800/jsonrpc';
        fetch(uri, {
            method: 'POST',
            mode: 'no-cors',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                jsonrpc: '2.0',
                id: Math.floor(Math.random() * 5),
                method: 'aria2.addUri',
                params: [[url]],
            }),
        });
        sendNotification(document.title);
    }

    function zippyshare() {
        const url = document.querySelector('a#dlbutton');
        download(url.href);
    }

    function solidfile() {
        const script = document
            .evaluate(
                "//script[contains(., 'downloadUrl')]",
                document,
                null,
                XPathResult.ANY_TYPE,
                null,
            )
            .iterateNext();
        const data = script.innerHTML;
        const result = JSON.parse(data.match('({.*})')[0]);
        download(result.downloadUrl);
    }

    function anonfiles() {
        const url = document.querySelector('a#download-url');
        download(url.href);
    }

    function mediafire() {
        const url = document.querySelector('a#downloadButton');
        download(url.href);
    }

    let hostname = window.location.hostname;

    if (hostname.includes('zippyshare')) {
        zippyshare();
    } else if (hostname.includes('anonfiles')) {
        anonfiles();
    } else if (hostname.includes('solidfile')) {
        solidfile();
    } else if (hostname.includes('mediafire')) {
        mediafire();
    }
})();
```

Setiap fungsi (`zippyshare`, `solidfile`, `anonfiles`, `mediafire`) mengekstrak direct download link dari masing-masing situs menggunakan DOM query selector, lalu mengirimnya ke aria2c via fetch API.

## Tantangan yang Dihadapi

Tantangan utama ada pada monkeyscript -- khususnya masalah **CORS**. Karena aria2c berjalan di `localhost:6800` sementara monkeyscript dieksekusi di domain situs file hosting, browser memblokir request cross-origin. Menggunakan `mode: 'no-cors'` pada fetch memang menghilangkan error di console, tapi bukan solusi yang ideal karena response dari server tidak bisa dibaca. Untuk kebutuhan saya saat itu -- cukup fire-and-forget tanpa perlu membaca response -- pendekatan ini masih bisa diterima.

Selain itu, setiap situs file hosting memiliki struktur DOM yang berbeda untuk menyembunyikan direct link. Solidfiles misalnya, menyimpan URL download di dalam tag `<script>` sebagai JSON, sehingga perlu XPath dan regex untuk mengekstraknya.

## Insight dan Pembelajaran

Pendekatan userscript ternyata jauh lebih fleksibel dibandingkan monkeyscript. Dengan userscript, kita bisa memanfaatkan seluruh ekosistem CLI tools (`jq`, `curl`, `notify-send`) yang sudah tersedia di sistem. Sementara monkeyscript terbatas pada JavaScript dan sandbox browser.

Dari sisi maintainability, monkeyscript lebih rapuh karena sangat bergantung pada struktur DOM situs pihak ketiga. Begitu situs mengubah layout-nya, script langsung rusak. Userscript yang bekerja di level URL jauh lebih stabil.

## Penutup

Integrasi aria2c dengan qutebrowser melalui dua pendekatan ini sudah berjalan sesuai harapan. Userscript `qb2aria` menjadi pilihan utama untuk download manual, sementara monkeyscript melengkapi untuk otomasi di situs file hosting tertentu. Ke depannya, pendekatan monkeyscript bisa digantikan dengan browser extension yang lebih proper untuk mengatasi masalah CORS.

## Referensi

- manpage aria2c -- Diakses pada 2022-01-07
- [Qutebrowser](https://qutebrowser.org/index.html) -- Diakses pada 2022-01-07
