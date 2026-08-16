+++
draft = false
date = '2026-08-15'
title = 'Override systemd Service Tanpa Mengedit Unit Aslinya'
type = 'blog'
description = 'Cara mengubah perilaku systemd service pakai drop-in override lewat systemctl edit, tanpa menyentuh unit file bawaan yang bakal ketimpa setiap update paket'
image = ''
tags = ['systemd', 'linux']
+++

## Latar Belakang

Setelah kebiasaan [bikin systemd service sendiri buat jalanin binary custom](/blog/menjalankan-aplikasi-sebagai-systemd-service-di-linux/), makin ke sini saya lebih sering berurusan dengan service yang datang dari paket, misalnya `nginx.service`, `postgresql.service`, atau `docker.service`. Unit file-nya bukan saya yang bikin, melainkan ikut kepasang waktu install paket dan ditaruh di `/usr/lib/systemd/system/`.

Masalahnya muncul waktu saya butuh nyetel sedikit perilakunya. Kadang cuma pengin nambah environment variable, kadang mau ganti `Restart=` biar lebih agresif, atau naikin limit file descriptor. Refleks pertama saya dulu langsung buka unit file-nya pakai editor, ubah barisnya, save, selesai. Kelihatannya beres, sampai suatu kali paketnya di-update dan semua perubahan saya hilang tanpa jejak karena file-nya ditimpa versi baru dari maintainer.

## Permasalahan

Begitu sadar ngedit unit file bawaan itu keliru, muncul beberapa hal yang bikin saya penasaran:

- **Perubahan hilang saat update**, unit file di `/usr/lib/systemd/system/` itu milik paket, jadi tiap `apt upgrade` atau `pacman -Syu` bisa nimpa file-nya dan ngehapus editan saya
- **Nggak tahu file override taruh di mana**, sempat bingung apakah harus nyalin unit-nya ke `/etc/systemd/system/`, atau ada cara yang lebih rapi biar nggak nyalin seluruh isinya
- **Bingung cara override satu baris**, saya cuma mau ganti `ExecStart=`, tapi kalau nyalin seluruh unit malah harus ikut jaga baris lain yang sebenarnya nggak saya ubah
- **`ExecStart=` malah dobel**, waktu coba-coba, nulis `ExecStart=` di override bukannya mengganti, tapi malah nambah, dan service-nya gagal start
- **Susah lacak setelan efektifnya**, setelah ada beberapa lapis file, saya nggak yakin baris mana yang benar-benar dipakai systemd
- **Nggak tahu cara balikin**, kalau override saya ternyata salah, gimana cara mengembalikan service ke setelan aslinya tanpa nebak-nebak file mana yang harus dihapus

## Pendekatan Solusi

Ternyata systemd memang sudah nyiapin cara resmi buat kasus ini, yaitu **drop-in override**. Idenya, unit file bawaan dibiarin utuh, dan perubahan saya ditaruh di file terpisah yang di-merge di atasnya. Update paket nggak akan ngutak-ngatik file override saya, jadi setelannya awet.

## Kenapa Jangan Edit Unit File Bawaan

Systemd mencari unit file di beberapa direktori dengan prioritas yang beda. Tiga yang paling sering ketemu urutannya seperti ini, dari prioritas paling rendah ke paling tinggi:

| Lokasi                        | Pemilik            | Keterangan                                                   |
| ----------------------------- | ------------------ | ----------------------------------------------------------- |
| `/usr/lib/systemd/system/`    | paket / distro     | Unit file bawaan. Ketimpa setiap update paket               |
| `/run/systemd/system/`        | runtime            | Unit sementara, hilang saat reboot                          |
| `/etc/systemd/system/`        | admin (saya)       | Setelan lokal. Prioritas tertinggi, nggak disentuh update   |

Karena `/etc/systemd/system/` menang atas `/usr/lib/`, semua penyesuaian lokal memang tempatnya di situ, bukan di file bawaan. Ngedit langsung file di `/usr/lib/` itu ngelawan mekanisme ini, karena selain rawan ketimpa, juga nyampur perubahan saya dengan file milik paket sampai susah dibedain mana yang asli dan mana editan.

Ada dua cara buat naruh perubahan di `/etc/`, yaitu drop-in override buat nyetel sebagian kecil, dan full override buat ganti seluruh unit. Dua-duanya dikelola paling gampang lewat `systemctl edit`.

## Drop-in Override: Mengubah Sebagian

**Drop-in** adalah file `.conf` potongan yang di-merge di atas unit asli. Systemd baca unit bawaan dulu, lalu numpuk isi drop-in di atasnya, jadi saya cukup nulis baris yang mau diubah saja tanpa nyalin sisanya.

Cara paling rapi bikin drop-in adalah lewat `systemctl edit`. Misalnya saya mau nyetel `nginx.service`:

```bash
$ sudo systemctl edit nginx.service
```

Perintah ini otomatis bikin direktori `/etc/systemd/system/nginx.service.d/` dan buka editor buat file `override.conf` di dalamnya. Apapun yang saya tulis di situ bakal di-merge di atas unit asli. Misalnya saya mau naikin limit file descriptor dan nambah environment variable:

```ini
[Service]
LimitNOFILE=65536
Environment=WORKER_MODE=aggressive
```

Setelah save dan keluar dari editor, `systemctl edit` otomatis manggil `daemon-reload`, jadi nggak perlu jalanin manual. Tinggal restart service-nya biar setelan baru kepakai:

```bash
$ sudo systemctl restart nginx.service
```

Yang di-merge cuma dua baris tadi. Sisa isi `nginx.service` (`ExecStart=`, `After=`, dan lainnya) tetap dari unit asli, jadi kalau maintainer ngubah salah satu baris itu di update berikutnya, saya tetap ikut kebagian perubahannya.

> **Note:** Nama file drop-in nggak harus `override.conf`. Systemd baca semua file `.conf` di dalam direktori `.d/` secara alfabetis, jadi bisa dipecah jadi `10-limits.conf`, `20-env.conf`, dan seterusnya kalau mau dipisah per kategori. `systemctl edit` cuma milih nama `override.conf` sebagai default.

## Mengganti Directive yang Sudah Ada Nilainya

Ini bagian yang dulu bikin saya kejeblos. Untuk directive yang menerima banyak nilai seperti `ExecStart=`, `Environment=`, atau `ExecStartPre=`, nulis directive-nya lagi di drop-in itu sifatnya **menambah**, bukan mengganti. Jadi kalau unit asli sudah punya `ExecStart=`, lalu saya tulis `ExecStart=` lagi di override, service-nya malah punya dua perintah start dan langsung gagal.

Triknya, kosongin dulu directive-nya dengan assignment kosong, baru isi nilai baru. Assignment kosong ini nge-reset daftar sebelumnya jadi kosong:

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/nginx-custom -g 'daemon off;'
```

Baris `ExecStart=` pertama menghapus perintah dari unit asli, baris kedua ngeset yang baru. Tanpa baris kosong itu, systemd bakal ngeluh ada dua `ExecStart=` dan nolak start service. Pola ini berlaku buat semua directive yang sifatnya numpuk daftar.

> **Penting:** Directive yang cuma nampung satu nilai seperti `Restart=` atau `User=` nggak butuh trik reset ini. Nulis ulang langsung nimpa nilai lamanya. Baris kosong cuma perlu buat directive yang sifatnya menambah ke daftar.

## Full Override: Mengganti Seluruh Unit

Kalau perubahannya kebanyakan sampai drop-in malah lebih ribet, saya bisa nyalin seluruh unit ke `/etc/` dan ngedit bebas di sana. Tinggal tambahin flag `--full`:

```bash
$ sudo systemctl edit --full nginx.service
```

Bedanya, perintah ini nyalin isi lengkap `nginx.service` ke `/etc/systemd/system/nginx.service`, lalu buka buat diedit. File di `/etc/` ini bakal sepenuhnya menggantikan yang di `/usr/lib/`, bukan di-merge. Konsekuensinya, saya jadi pegang seluruh isi unit dan **nggak lagi kebagian perubahan** dari update paket, karena versi saya yang menang penuh.

Karena itu full override saya pakai cuma kalau perubahannya besar. Selama cuma nyetel beberapa baris, drop-in jauh lebih aman karena tetap ngikutin unit asli buat bagian yang nggak saya ubah.

## Melihat Hasil Merge dan Membalikkannya

Setelah ada beberapa lapis, saya butuh cara mastiin setelan efektifnya benar. `systemctl cat` nampilin isi unit final beserta semua drop-in yang ke-merge, lengkap dengan komentar dari file mana tiap bagian berasal:

```bash
$ systemctl cat nginx.service
```

```ini
# /usr/lib/systemd/system/nginx.service
[Unit]
Description=nginx web server
...

# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65536
Environment=WORKER_MODE=aggressive
```

Komentar path di atas tiap blok bikin gampang lihat baris mana datang dari unit asli dan mana dari override saya. Buat ngecek nilai satu properti yang benar-benar dipakai, `systemctl show` juga membantu:

```bash
$ systemctl show nginx.service -p LimitNOFILE
```

```
LimitNOFILE=65536
```

Kalau override-nya ternyata salah dan saya mau balik ke setelan bawaan, nggak perlu nebak-nebak file mana yang harus dihapus. `systemctl revert` ngapus semua drop-in dan full override lokal buat unit itu sekaligus, lalu balikin ke versi `/usr/lib/`:

```bash
$ sudo systemctl revert nginx.service
$ sudo systemctl restart nginx.service
```

Perintah ini otomatis ngehapus direktori `nginx.service.d/` dan file `/etc/systemd/system/nginx.service` kalau ada, jadi service-nya bener-bener balik ke keadaan seperti baru dipasang.

## Insight dan Pembelajaran

- **Jangan pernah ngedit unit file di `/usr/lib/systemd/system/`**, itu milik paket dan ketimpa tiap update. Semua penyesuaian tempatnya di `/etc/`, dan `systemctl edit` naruhnya ke sana otomatis
- **Drop-in itu default, full override itu pengecualian**, selama cuma nyetel beberapa baris, drop-in bikin saya tetap kebagian perubahan maintainer di baris yang lain. Full override baru masuk akal kalau perubahannya kebanyakan
- **Directive daftar butuh baris kosong buat reset**, `ExecStart=` dan sejenisnya sifatnya menambah, jadi harus dikosongin dulu (`ExecStart=`) sebelum diisi nilai baru, atau service-nya gagal start
- **`systemctl cat` buat ngintip hasil merge**, komentar path di tiap blok nunjukin baris mana dari unit asli dan mana dari override, jadi nggak perlu nebak setelan efektifnya
- **`systemctl revert` buat balik bersih**, satu perintah ngapus semua override lokal dan balikin unit ke versi bawaan, tanpa perlu inget-inget file mana yang tadi dibikin

## Penutup

Inti dari override yang bener cuma satu, biarin file milik paket tetap utuh dan taruh perubahan di lapisan sendiri. `systemctl edit` bikin drop-in di `/etc/` yang di-merge di atas unit asli, sedangkan `systemctl edit --full` buat kasus yang butuh ganti seluruhnya. Dengan pola ini, setelan saya awet lintas update dan gampang dibalikin lewat `systemctl revert` kalau ternyata keliru.

## Referensi

- [systemd.unit(5) Manual Page](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html), diakses pada 2026-08-15
- [systemctl(1) Manual Page](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html), diakses pada 2026-08-15
