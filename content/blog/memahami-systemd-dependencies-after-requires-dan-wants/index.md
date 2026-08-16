+++
draft = false
date = '2026-08-15'
title = 'Memahami systemd Dependencies: After vs Requires vs Wants'
type = 'blog'
description = 'Membedah perbedaan opsi dependency dan ordering di systemd, mulai dari kenapa After bukan Requires, kenapa Requires nggak menjamin urutan, sampai kombinasi mana yang benar buat bikin service saling bergantung tanpa race condition'
image = ''
tags = ['systemd', 'linux']
+++

## Latar Belakang

Sejak kebiasaan [bikin systemd service buat jalanin binary custom](/blog/menjalankan-aplikasi-sebagai-systemd-service-di-linux/), pola unit file saya nyaris selalu sama, yaitu satu `[Service]`, satu `ExecStart`, `enable --now`, selesai. Aman selama servicenya berdiri sendiri. Masalah baru muncul begitu ada service yang butuh service lain, misalnya aplikasi saya yang harus konek ke Postgres yang juga jalan di mesin yang sama.

Refleks pertama saya waktu itu nambahin `After=postgresql.service` di unit file, dengan asumsi baris itu artinya "service saya butuh Postgres". Sepintas jalan, sampai suatu kali Postgres-nya `failed` waktu boot, dan aplikasi saya tetap `active` sambil ngelempar error koneksi terus-terusan. Di titik itu saya sadar `After=` ternyata nggak ngelakuin yang saya kira.

Ternyata systemd memisahkan dua konsep yang selama ini saya campur jadi satu, yakni **ordering**, soal urutan siapa start duluan, dan **dependency**, soal unit mana yang wajib ikut dijalankan. `After=`, `Requires=`, dan `Wants=` itu ngurus hal yang beda-beda, dan salah kombinasi bikin race condition yang susah dilacak.

## Permasalahan

Begitu paham `After=` cuma ngurus urutan, muncul beberapa pertanyaan baru yang bikin saya bingung:

- **`After=` saja belum cukup**, service saya butuh Postgres ikut hidup, tapi `After=` cuma ngatur urutan dan nggak ikut menjalankan Postgres. Lalu opsi mana yang tugasnya menjalankan dependency?
- **`Requires=` malah nggak jamin urutan**, waktu ganti ke `Requires=`, Postgres ikut dijalankan tapi kadang service saya start duluan sebelum Postgres siap, muncul lagi race condition-nya
- **Bingung beda `Requires=` dan `Wants=`**, dua-duanya sama-sama ikut menjalankan unit lain, tapi nggak jelas kapan harus pakai yang mana
- **Efek gagal nggak konsisten**, kadang service saya ikut mati waktu dependency-nya mati, kadang enggak, tergantung opsi yang dipakai, dan saya nggak paham aturannya
- **Nggak tahu opsi lain**, ada `Requisite=`, `BindsTo=`, `PartOf=` yang kadang muncul di unit file orang, tapi nggak pernah paham bedanya sama `Requires=`

## Ordering dan Dependency Itu Dua Hal Berbeda

Kunci yang bikin semuanya klik itu ternyata sederhana. Di systemd, **ordering** dan **dependency** itu dua hal yang benar-benar terpisah. Ordering ngurus **kapan** sebuah unit start relatif ke unit lain, sedangkan dependency ngurus **apakah** unit lain ikut dijalankan waktu unit ini start.

| Konsep         | Opsi                            | Pertanyaan yang dijawab                                    |
| -------------- | ------------------------------- | ---------------------------------------------------------- |
| **Ordering**   | `After=`, `Before=`             | Kalau dua unit sama-sama jalan, siapa start duluan?        |
| **Dependency** | `Requires=`, `Wants=`, dan lain | Kalau unit ini start, unit lain mana yang ikut dijalankan? |

Yang penting, satu opsi cuma ngurus satu hal. `After=` nggak pernah ikut menjalankan unit lain, dan `Requires=` nggak pernah menentukan urutan. Karena itu, kombinasi yang benar hampir selalu butuh dua opsi sekaligus, satu buat ngatur urutan dan satu buat menjalankan dependency.

Salah paham saya di awal murni karena mencampur dua hal ini jadi satu kata "butuh".

## Ordering: After dan Before

**`After=`** dan **`Before=`** cuma mengatur urutan start, tanpa efek lain. `After=b.service` di unit `a.service` artinya, **kalau** `a` dan `b` kebetulan sama-sama dijadwalkan jalan dalam satu flow, maka `b` harus selesai start dulu baru `a` menyusul. Karena opsi ini nggak bikin `b` ikut jalan.

```ini
[Unit]
Description=Aplikasi saya
After=postgresql.service

[Service]
ExecStart=/opt/myapp/server
```

Kalau saya cuma nulis seperti di atas lalu `postgresql.service` nggak di-enable, service saya tetap jalan tanpa nunggu siapa-siapa. `After=` di sini nggak melakukan apa-apa selama Postgres nggak ikut dijadwalkan start.

`Before=` cuma kebalikannya, `Before=b.service` sama artinya dengan nulis `After=a.service` di unit `b`. Biasanya cukup pakai `After=` di sisi yang bergantung biar nggak bingung.

> **Note:** `After=` juga menentukan urutan waktu shutdown, cuma dibalik. Unit yang start belakangan akan di-stop duluan, jadi urutan matinya otomatis kebalikan dari urutan hidupnya.

## Dependency: Requires dan Wants

Ini bagian yang menentukan unit lain ikut dijalankan atau nggak waktu unit ini di-start.

### Requires: dependency wajib

**`Requires=b.service`** artinya waktu unit ini di-start, `b` ikut dijalankan juga. Kalau `b` gagal start, unit ini ikut dibatalkan dan ditandai gagal. Kalau `b` di-stop belakangan, unit ini juga ikut di-stop.

```ini
[Unit]
Description=Aplikasi saya
Requires=postgresql.service
After=postgresql.service

[Service]
ExecStart=/opt/myapp/server
```

Perhatikan `After=` tetap ada. Tanpa `After=`, `Requires=` cuma menjamin Postgres **ikut dijalankan**, tapi systemd bebas start keduanya **barengan**. Aplikasi saya bisa saja jadi hidup sepersekian detik sebelum Postgres siap menerima koneksi, dan race condition-nya balik lagi. Inilah kesalahan kedua saya, yakni `Requires=` tanpa `After=`.

> **Penting:** `Requires=` dan `After=` itu pasangan, bukan pilihan. `Requires=` yang menjalankan unit-nya, `After=` yang memastikan urutannya. Butuh dua-duanya buat "start Postgres dulu, baru aplikasi saya".

### Wants: dependency opsional

**`Wants=b.service`** juga ikut menjalankan `b`, tapi bersifat best-effort. Kalau `b` gagal start, unit ini **tetap jalan** seolah nggak terjadi apa-apa. Ini bentuk dependency paling longgar dan paling sering dipakai, karena nggak bikin satu service ikut gagal cuma gara-gara dependency-nya gagal.

```ini
[Unit]
Description=Aplikasi saya
Wants=redis.service
After=redis.service

[Service]
ExecStart=/opt/myapp/server
```

Contoh di atas cocok buat dependency opsional. Redis dipakai buat cache, jadi kalau Redis gagal, saya masih mau aplikasinya hidup, cuma jalan tanpa cache. `Wants=` pas untuk kasus ini, sedangkan Postgres yang wajib ada tetap pakai `Requires=`.

> **Tip:** Directory `myapp.service.wants/` yang isinya symlink itu sebenarnya cara `enable` bekerja di balik layar. Waktu saya `systemctl enable`, systemd cuma bikin symlink di dalam `.wants/` milik target, yang secara efektif nambahin `Wants=` dari target ke service saya.

### Menggabungkan Requires dan Wants

Di kenyataan, satu service sering butuh lebih dari satu dependency dengan bobot yang beda. Aplikasi saya wajib punya Postgres, tapi Redis cuma buat cache. Dua-duanya bisa ditumpuk di satu unit file, tinggal taruh yang wajib di `Requires=` dan yang opsional di `Wants=`, lalu urutannya diatur `After=`.

```ini
[Unit]
Description=Aplikasi saya
Requires=postgresql.service
Wants=redis.service
After=postgresql.service redis.service

[Service]
ExecStart=/opt/myapp/server
```

`After=` bisa nyebut beberapa unit sekaligus dipisah spasi, jadi service saya baru start setelah Postgres dan Redis dua-duanya kelar. Bedanya cuma di efek gagal, kalau Postgres gagal service saya ikut batal, sedangkan kalau Redis gagal service saya tetap jalan tanpa cache.

## Kalau Dependency Mati, Apa yang Ikut Mati

Ini bagian yang dulu paling bikin bingung buat saya, dan jawabannya beda-beda per opsi. Ada dua momen yang perlu dipisah, gagal waktu start dan berhenti waktu sudah jalan.

| Opsi    | Dep gagal saat start | Dep di-stop setelah jalan | Dep crash mendadak     |
| ------------ | -------------------- | ------------------------- | ---------------------- |
| `Wants=`     | unit tetap jalan     | unit tetap jalan          | unit tetap jalan       |
| `Requires=`  | unit ikut gagal      | unit ikut di-stop         | unit **tetap** jalan   |
| `BindsTo=`   | unit ikut gagal      | unit ikut di-stop         | unit **ikut** di-stop  |

Kolom terakhir sering luput. `Requires=` cuma ikut mati kalau dependency-nya di-stop lewat systemd secara eksplisit, misalnya `systemctl stop postgresql`. Tapi kalau prosesnya mati sendiri di luar sepengetahuan systemd, `Requires=` nggak bereaksi. Kalau saya butuh unit yang benar-benar terikat nasib sampai crash sekalipun, itu tugasnya `BindsTo=`.

## Opsi Lain yang Sering Muncul

Selain tiga yang utama, ada beberapa yang kadang muncul di unit file orang dan bikin penasaran:

| Opsi     | Fungsi                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------- |
| `Requisite=`  | Mirip `Requires=`, tapi nggak ikut men-start dependency. Kalau dependency belum aktif, unit langsung gagal |
| `BindsTo=`    | Versi lebih ketat dari `Requires=`, ikut mati bahkan waktu dependency crash mendadak, bukan cuma di-stop |
| `PartOf=`     | Efek satu arah, stop dan restart pada parent unit ikut berpengaruh ke unit ini, tapi start enggak    |
| `Conflicts=`  | Kebalikan dependency, start unit ini otomatis men-stop unit lawannya, cocok buat dua service yang nggak boleh hidup bareng |

Yang paling praktis dari daftar ini menurut saya `PartOf=`. Gunanya buat ngelompokin beberapa service di bawah satu **parent unit**, biasanya sebuah `.target`. Tiap service dikasih `PartOf=grup.target`, jadi begitu `grup.target` di-restart atau di-stop, semua anggotanya ikut restart atau ikut berhenti sekaligus. Efeknya cuma satu arah, nge-stop salah satu service nggak berpengaruh ke parent unit atau service lain di grup itu.

## Insight dan Pembelajaran

- **Ordering dan dependency itu dua hal terpisah**, `After=` ngurus urutan, `Requires=`/`Wants=` ngurus apakah unit lain ikut dijalankan. Mencampur keduanya jadi satu kata "butuh" adalah akar semua kebingungan saya
- **`Requires=` hampir selalu butuh `After=`**, tanpa pasangan `After=`, dependency ikut jalan tapi urutannya nggak dijamin, dan race condition-nya balik lagi
- **Pilih `Wants=` sebagai default**, pakai `Requires=` cuma buat dependency yang benar-benar wajib. Dependency opsional bikin sistem lebih tahan banting waktu satu komponen gagal
- **`Requires=` nggak ikut mati kalau dependency crash**, dia cuma ikut mati kalau dependency di-stop lewat systemd secara eksplisit, bukan kalau prosesnya mati sendiri. Kalau butuh ikatan yang berlaku sampai dependency crash, pakai `BindsTo=`
- **`enable` itu cuma `Wants=` terselubung**, symlink di dalam `.wants/` adalah mekanisme di balik `systemctl enable`, jadi paham `Wants=` sekaligus paham cara enable bekerja

## Penutup

Inti dari semua kebingungan saya cuma satu, mencampur "kapan jalan" dengan "apa yang ikut jalan" jadi satu konsep. Begitu dua hal itu dipisah, `After=` buat urutan dan `Requires=`/`Wants=` buat menentukan unit lain ikut jalan, semua opsi lain jadi masuk akal. Kombinasi paling aman untuk dependency wajib adalah `Requires=` plus `After=`, sedangkan untuk yang opsional cukup `Wants=` plus `After=`.

## Referensi

- [systemd.unit(5) Manual Page](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html), Diakses pada 2026-08-15
- [systemd.service(5) Manual Page](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html), Diakses pada 2026-08-15
