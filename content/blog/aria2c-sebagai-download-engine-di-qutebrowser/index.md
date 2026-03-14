+++
draft = false
date = '2022-01-06'
title = 'Aria2c Sebagai Download Engine Di Qutebrowser'
type = 'blog'
description = 'Menggunakan aria2c sebagai download engine di qutebrowser melalui userscript dan monkeyscript.'
image = ''
tags = ['browser', 'download', 'cli', 'userscript', 'monkeyscript']
+++

## Intro

Pernah merasa download bawaan browser itu kurang greget? Nah, kenalan dulu sama **aria2c** -- sebuah download manager berbasis CLI yang bisa dibilang serba bisa. Dia mendukung berbagai protokol mulai dari HTTP(S), FTP, SFTP, BitTorrent, hingga Metalink. Artinya, cukup satu tool ini saja dan kita tidak perlu lagi repot install torrent client terpisah.

Yang bikin aria2c makin menarik, dia bisa dijadikan download service yang dapat diakses dari berbagai frontend, baik yang berbasis GUI seperti [UGET](https://ugetdm.com/) dan [Persepolis](https://persepolisdm.github.io/), yang berbasis web seperti [AriaNg](https://github.com/mayswind/AriaNg) dan [webui-aria2](https://github.com/ziahamza/webui-aria2), bahkan yang berbasis TUI seperti [aria2p](https://github.com/pawamoy/aria2p/).

Lalu ada **Qutebrowser** -- browser yang mengajak penggunanya merasakan nikmatnya kesakitan menghafal keymap. Kenapa begitu? Jelas-jelas ada browser lain yang jauh lebih mudah dioperasikan, jadi bisa dibilang pengguna qutebrowser punya selera tersendiri. Qutebrowser adalah browser yang mengandalkan keyboard untuk navigasi dengan tampilan yang minimalis. Dibangun menggunakan Python dengan module PyQt5, backend-nya menggunakan qt5-webengine (default) dan qt5-webkit.

Nah, bagaimana kalau dua tool keren ini digabungkan? Mari kita bahas.

## Implementasi

Ada dua pendekatan yang saya gunakan untuk mengintegrasikan aria2c ke dalam qutebrowser: **userscript** dan **monkeyscript**.

### UserScript

Userscript mirip dengan monkeyscript, tapi punya satu keunggulan penting -- bisa ditulis dengan bahasa pemrograman apapun, selama bahasa tersebut mampu membaca environment variable yang disediakan oleh qutebrowser. Daftar lengkap environment variable-nya bisa dilihat di [qutebrowser/userscript](https://qutebrowser.org/doc/userscripts.html).

Untuk implementasinya, saya menulis userscript bernama `qb2aria` yang disesuaikan dengan kebutuhan saya. Berikut isinya:

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

Penjelasan singkat:

| Komponen    | Fungsi                                                                                                  |
| :---------: | :------------------------------------------------------------------------------------------------------ |
| `datagen`   | Menggenerate payload JSON yang dibutuhkan untuk mengirim request download ke aria2c melalui curl         |
| `$QUTE_URL` | Environment variable bawaan qutebrowser yang berisi URL halaman aktif, hanya tersedia di dalam userscript |

Simpan file `qb2aria` di folder `/home/$USER/.config/qutebrowser/userscripts`, lalu beri hak akses eksekusi:

```
$ chmod +x qb2aria
```

Langkah terakhir, tambahkan keymap berikut pada `config.py` qutebrowser:

```python
config.bind(",d", "hint links userscript qb2aria", "normal")
```

Cara pakainya gampang: tekan `,` lalu `d`, kemudian pilih hint link yang ingin di-download. Selesai!

### Monkeyscript

Monkeyscript atau yang lebih dikenal sebagai **Greasemonkey** awalnya adalah extension di Mozilla Firefox. Sekarang, greasemonkey sudah tersedia di berbagai browser termasuk qutebrowser. Berbeda dengan userscript di atas, monkeyscript hanya bisa ditulis dengan JavaScript.

Berikut monkeyscript yang saya gunakan untuk otomatis mengirim link download dari beberapa file hosting ke aria2c:

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

## Referensi

- manpage aria2c -- Diakses pada 2022-01-07
- [Qutebrowser](https://qutebrowser.org/index.html) -- Diakses pada 2022-01-07
