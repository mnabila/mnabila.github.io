+++
draft = false
date = '2026-04-08'
title = 'Instalasi Docker di Ubuntu Server 22.04 dan Solusi Error OCI Runtime'
type = 'blog'
description = 'Pengalaman instalasi Docker di Ubuntu 22.04 yang terkendala error OCI runtime saat menjalankan container dengan Docker Compose'
image = ''
tags = ['docker', 'ubuntu', 'docker-compose', 'troubleshooting', 'container']
+++

## Latar Belakang

Docker sudah jadi tool wajib untuk development modern hampir semua project yang melibatkan backend, database, atau service lain pasti pakai Docker supaya environment-nya konsisten. Instalasi Docker di Ubuntu biasanya straightforward, tinggal ikuti dokumentasi resmi dan selesai. Tapi kadang ada error yang muncul bukan saat instalasi, melainkan saat menjalankan container dan itu yang bikin frustasi karena kita merasa sudah install dengan benar.

## Permasalahan

Saya baru setup server dengan Ubuntu 22.04 dan install Docker mengikuti dokumentasi resmi. Instalasi berjalan lancar, daemon Docker jalan normal, `docker info` tidak ada error. Tapi begitu jalankan project dengan `docker compose up`, muncul error ini:

```
ERROR: for backend  Cannot start service backend: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied
ERROR: Encountered errors while bringing up the project.
```

Error ini muncul di tahap **container init** -- artinya image sudah berhasil di-pull, container sudah dibuat, tapi gagal saat proses inisialisasi. Masalahnya ada di `runc` yang tidak bisa membuka file sysctl `net.ipv4.ip_unprivileged_port_start` karena **permission denied**.

Kalau ditelusuri, ini bukan masalah permission user atau Docker group. Ini adalah bug compatibility antara versi `runc` yang terinstall dengan kernel dan konfigurasi security di Ubuntu 22.04. Versi Docker yang diinstall secara default dari repository Docker terkadang membawa versi `runc` yang punya masalah ini.

## Pendekatan Solusi

Ada beberapa pendekatan yang bisa dicoba untuk mengatasi error ini:

| Pendekatan | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **Downgrade runc secara manual** | Targeted fix | Risky, bisa break dependency lain |
| **Disable sysctl di compose file** | Quick workaround | Tidak menyelesaikan root cause, dan tidak selalu applicable |
| **Install Docker versi spesifik (v28)** | Fix dari upstream, semua komponen compatible | Perlu pin versi manual |

Saya memilih **install Docker versi 28** karena:

1. **Fix ada di upstream** -- Docker 28 membawa versi `runc` yang sudah memperbaiki masalah ini
2. **Semua komponen sinkron** -- install satu paket yang sudah tested compatible, bukan patch satu komponen saja
3. **Clean solution** -- tidak perlu workaround yang bisa menimbulkan masalah lain di kemudian hari

## Implementasi Teknis

### Setup Repository Docker

Pertama, pastikan repository Docker sudah di-setup dengan benar. Kalau belum, ikuti langkah berikut sesuai [dokumentasi resmi Docker](https://docs.docker.com/engine/install/ubuntu/):

```
# Hapus package Docker lama kalau ada
$ for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt remove $pkg; done

# Install dependency
$ sudo apt update
$ sudo apt install ca-certificates curl

# Tambah Docker GPG key
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Tambah repository
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt update
```

### Install Docker Versi 28

Daripada install versi latest yang bisa berubah-ubah, kita pin ke versi spesifik yang sudah confirmed working:

```
$ VERSION_STRING=5:28.0.0-1~ubuntu.22.04~jammy
$ sudo apt install \
    docker-ce=$VERSION_STRING \
    docker-ce-cli=$VERSION_STRING \
    containerd.io=1.7.28-1~ubuntu.22.04~jammy \
    docker-buildx-plugin \
    docker-compose-plugin
```

Penjelasan masing-masing package:

| Package | Fungsi |
|---------|--------|
| `docker-ce` | Docker Engine (daemon) |
| `docker-ce-cli` | CLI client untuk interaksi dengan Docker daemon |
| `containerd.io` | Container runtime yang digunakan Docker di bawah |
| `docker-buildx-plugin` | Plugin untuk build image dengan fitur extended (multi-platform, cache, dll) |
| `docker-compose-plugin` | Plugin Docker Compose v2, menggantikan `docker-compose` standalone |

### Verifikasi Instalasi

Setelah install, pastikan semua komponen jalan dengan benar:

```
# Cek versi Docker
$ docker version

# Cek info Docker daemon
$ docker info

# Test dengan hello-world
$ sudo docker run hello-world
```

Kalau `hello-world` berhasil jalan dan menampilkan pesan sukses, berarti instalasi sudah benar.

### Jalankan Docker Compose

Sekarang coba jalankan kembali project yang tadi error:

```
$ docker compose up -d
```

Container seharusnya sudah bisa start tanpa error OCI runtime.

## Tantangan yang Dihadapi

Tantangan utama dari masalah ini adalah **error message-nya tidak langsung menunjuk ke solusi**. Pesan `permission denied` pada file sysctl membuat kita berpikir ini masalah permission atau security policy, padahal root cause-nya ada di versi `runc` yang tidak compatible.

Kalau search error ini di internet, banyak saran yang mengarah ke modifikasi AppArmor profile atau menambah `--privileged` flag -- dua-duanya workaround yang **tidak disarankan** karena melemahkan security posture container.

## Insight dan Pembelajaran

Beberapa hal yang bisa diambil dari pengalaman ini:

- **Jangan selalu install versi latest tanpa pikir panjang** -- di production atau environment yang butuh stabilitas, pin versi package yang sudah tested. Versi latest belum tentu paling compatible dengan OS kita.
- **Error permission denied tidak selalu soal permission** -- kadang itu symptom dari bug di runtime atau incompatibility antar komponen. Jangan langsung chmod 777 atau jalankan privileged mode.
- **Baca error message sampai habis** -- bagian `runc create failed` dan `open sysctl ... file` adalah clue penting bahwa masalahnya ada di level runtime, bukan di aplikasi atau image.
- **Dokumentasi resmi Docker itu lengkap** -- termasuk cara install versi spesifik. Kalau versi default bermasalah, cek versi lain yang tersedia di repository.

## Penutup

Error OCI runtime saat menjalankan Docker Compose di Ubuntu 22.04 ini memang membingungkan di awal, tapi solusinya cukup simpel yakni install Docker versi 28 yang membawa `runc` versi terbaru yang sudah fix masalah permission pada file sysctl. Kuncinya jangan langsung panik dan mengubah security configuration, tapi telusuri dulu apakah ada versi package yang lebih compatible. Pin versi package yang sudah tested itu solusi terbaik, terutama untuk environment production.

## Referensi

- [Docker - Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/) -- Diakses pada 2026-04-08
