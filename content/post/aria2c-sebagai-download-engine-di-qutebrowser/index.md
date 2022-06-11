---
title: "Aria2c Sebagai Download Engine Di Qutebrowser"
date: 2022-01-06T10:51:23+07:00
categories: ["application"]
tags: ["browser", "download", "cli", "userscript", "monkeyscript"]
---

## Intro

Aria2c sebuah program Command Line Interface atau yang biasa dikenal CLI, aria2c banyak kelebihannya dibandingkan dengan download engine lainnya dikarenakan banyak protocol yang aria2c support seperti HTTP(S), FTP, SFTP, BitTorrent, and Metalink. Dikarenakan banyak protocol yang dia support maka aku tak perlu lagi install torrent downloader selain itu si aria2c ini biisa dijadikan service download yang mana bisa diakses melalui berbagai program GUI seperti [UGET](https://ugetdm.com/), [Persepolish](https://persepolisdm.github.io/) atau yang webapps seperti [AriaNg](https://github.com/mayswind/AriaNg), [webui-aria2](https://github.com/ziahamza/webui-aria2) atau bahkan yang berbasis TUI juga ada seperti [aria2p](https://github.com/pawamoy/aria2p/).

Qutebrowser sebuah browser yang memungkinan penggunanya untuk merasakan kenyamanan dikala kesakitan menghafalkan keymapnya, kenapa kok begitu ? karna jelas jelas ada browser yang lebih mudah dioperasikan dibandingkan qutebrowser, jadi ada kemungkinan penggunanya memiliki kelainan didalam diri mereka 🤣. Qutebrowser adalah browser yang berfokus pada keyboard untuk navigasinya dengan tampilan yang minimalist tidak seperti browser pada umumnya. Qutebrowser dibangun dengan bahasa pemrograman python menggunakan module pyqt5 dan backendnnya menggunakan qt5-webengine dan qt5-webkit akan tetapi secara default qutebrowser menggunakanqt5-webengine sebagai default backendnya.

## Implementasi

Untuk pengimplementasian aku menggunakan dua pendekatan yang pertama yakni menggunakan userscript dan yang kedua menggunakan monkeyscript.

### UserScript

Userscript hampir mirip dengan monkeyscript akan tetapi userscript ini bisa ditulis dengan berbagai bahasa pemrograman selama bahasa pemrograman tersebut bisa membaca environment variable yang telah disediakan oleh qutebrowser. Untuk daftar environment variable teman teman bisa akses di [qutebrowser/userscript](https://qutebrowser.org/doc/userscripts.html). Dalam pengimplementasiannya di qutebrowser aku menggunakan userscript yang bernama `qb2aria`, userscript aku tulis sendiri sesuai dengan kebutuhanku saat ini, dibawah ini ada isi dari userscriptnya.

```
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

Penjelasan:

| Nama Fungsi | Penjelasan                                                                                                                       |
| :---------: | :------------------------------------------------------------------------------------------------------------------------------- |
|   datagen   | fungsi ini diperuntukkan untuk mengenerate data yang dibutuhkan sewaktu proses pengiriman data yang dilakukan oleh curl nantinya |
|  $QUTE_URL  | ini adalah environment variable yang telah disediakan oleh qutebrowser, environment ini tidak dapat digunakan diluar userscript  |

Simpan file `qb2aria` ini didalam folder `/home/$USER/.config/qutebrowser/userscripts`, kemudian tambahkan hak akses eksekusi pada file tersebut dengan perintah

```
$ chmod +x qb2aria
```

Langkah terakhir tambahkan keymap ini pada config.py qutebrowser.

```
config.bind(",d", "hint links userscript qb2aria", "normal")
```

Untuk menggunakan qb2aria ini tinggal tekan `coma` lalu `d` kemudian tekan hintnya.

### Monkeyscript

Monkeyscript atau lebih dikenal dengan sebutan Greasemonkey adalah sebuah extensions yang ada di mozzila firefox, kini greasemonkey tersedia diberbagai web browser salah satunya adalah di qutebrowser. Berbeda dengan userscriptnya qutebrowser, sepengetahuanku monkeyscript ini hanya bisa ditulis dengan bahasa pemrograman javascript. Dibawah ini monkeyscript yang aku gunakan.

```
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

- manpage aria2c Diakses pada 2022-01-07
- [Qutebrowser](https://qutebrowser.org/index.html) Diakses pada 2022-01-07
