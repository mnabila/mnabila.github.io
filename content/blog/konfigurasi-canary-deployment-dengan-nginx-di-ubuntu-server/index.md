+++
draft = false
date = '2026-06-27'
title = 'Konfigurasi Canary Deployment dengan Nginx di Ubuntu Server'
type = 'blog'
description = 'Menggunakan canary deployment di Nginx untuk memvalidasi infrastruktur baru dengan mengalirkan sebagian kecil trafik sebelum cutover penuh ke setup High Availability'
image = ''
tags = ['nginx', 'canary-deployment', 'high-availability', 'ubuntu']
+++

## Latar Belakang

Infrastruktur yang saya kelola awalnya berjalan di single server. Satu **Nginx** sebagai reverse proxy, satu backend, satu database. Selama skala masih kecil, setup ini cukup. Tapi seiring trafik bertambah dan kebutuhan uptime makin ketat, saatnya migrasi ke infrastruktur **High Availability (HA)** dengan multiple backend server.

Masalahnya, migrasi infrastruktur bukan hal yang bisa dilakukan sekali jalan. Langsung memindahkan semua trafik ke server baru tanpa validasi adalah resep bencana. Perlu mekanisme untuk menguji server baru dengan trafik production secara bertahap, memastikan semuanya berjalan normal, baru kemudian melakukan cutover penuh.

## Permasalahan

Migrasi dari single server ke infrastruktur HA punya beberapa risiko yang harus dimitigasi:

- **Tidak ada cara validasi dengan trafik real**: testing di staging tidak pernah benar-benar merepresentasikan kondisi production. Load pattern, data volume, dan edge case di production berbeda jauh
- **Cutover langsung terlalu berisiko**: memindahkan 100% trafik ke server baru sekaligus berarti kalau ada masalah, semua pengguna terdampak
- **Rollback harus cepat**: kalau server baru bermasalah, harus bisa kembali ke server lama dalam hitungan detik, bukan menit
- **Konsistensi data antar server**: server lama dan baru harus mengakses data yang sama supaya pengguna tidak mengalami inkonsistensi
- **Session management lintas server**: kalau session disimpan di memori lokal, user yang request-nya pindah server akan kehilangan session
- **Perbedaan konfigurasi**: server baru bisa saja punya perbedaan environment, versi dependency, atau konfigurasi yang tidak terdeteksi saat testing

## Pendekatan Solusi

Ada beberapa strategi untuk migrasi infrastruktur secara bertahap:

| Pendekatan              | Kelebihan                                                         | Kekurangan                                                              |
| ----------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Blue-Green**          | Cutover instan, rollback cepat                                    | Butuh duplikasi penuh infrastruktur, tidak ada validasi bertahap        |
| **Canary Deployment**   | Validasi bertahap dengan trafik real, risiko terkontrol           | Butuh monitoring yang baik untuk mendeteksi masalah di canary           |
| **Rolling Update**      | Tidak butuh infrastruktur tambahan                                | Sulit rollback kalau masalah baru terdeteksi di tengah proses           |
| **Shadow / Dark Launch**| Tidak berdampak ke user, cocok untuk testing                      | Tidak menguji response path ke user, butuh setup duplikasi request      |

Saya pilih **Canary Deployment** karena paling sesuai untuk skenario migrasi infrastruktur. Trafik dialirkan secara bertahap ke server baru, mulai dari persentase kecil (misalnya 10%), lalu dinaikkan secara incremental setelah validasi di setiap tahap. Kalau ada masalah, cukup kembalikan weight ke server lama. Nginx mendukung ini melalui directive `weight` di block `upstream` dan module `split_clients` untuk kontrol yang lebih granular.

Arsitektur yang dibangun:

| Komponen           | IP Address    | Peran                                                    |
| ------------------ | ------------- | -------------------------------------------------------- |
| **Load Balancer**  | `10.10.10.10` | Nginx, mengatur distribusi trafik antara server lama dan baru |
| **Server Lama**    | `10.10.10.11` | Backend production yang sudah berjalan (stable)          |
| **Server Baru 1**  | `10.10.10.12` | Backend baru, target canary deployment                   |
| **Server Baru 2**  | `10.10.10.13` | Backend baru, target canary deployment                   |

## Implementasi Teknis

### Persiapan Server Baru

Pastikan server baru sudah ter-setup dengan environment yang identik dengan server lama. Validasi versi runtime, dependency, dan konfigurasi aplikasi.

```bash
$ ssh 10.10.10.12 "cat /etc/os-release | grep VERSION"
$ ssh 10.10.10.12 "nginx -v"
$ ssh 10.10.10.12 "node --version"
```

Pastikan aplikasi sudah di-deploy dan berjalan di server baru sebelum mulai mengalirkan trafik.

```bash
$ ssh 10.10.10.12 "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/health"
```

Output `200` menandakan aplikasi sudah siap menerima request.

### Konfigurasi Upstream dengan Weight

Buat konfigurasi load balancer di Nginx. Tahap awal, server lama mendapat mayoritas trafik dan server baru hanya menerima sebagian kecil.

```bash
$ sudo nano /etc/nginx/conf.d/canary.conf
```

Konfigurasi awal dengan 10% trafik ke canary:

```nginx
upstream backend {
    # server lama - menerima mayoritas trafik
    server 10.10.10.11:8080 weight=9;

    # server baru (canary) - menerima ~10% trafik
    server 10.10.10.12:8080 weight=1;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }
}
```

Directive `weight` mengatur proporsi distribusi trafik. Dengan `weight=9` di server lama dan `weight=1` di server baru, dari setiap 10 request, sekitar 9 masuk ke server lama dan 1 ke server baru.

### Konfigurasi dengan split_clients untuk Kontrol Lebih Granular

Kalau butuh kontrol persentase yang lebih presisi atau ingin routing berdasarkan atribut request tertentu, gunakan module **split_clients**. Module ini membagi trafik berdasarkan hash dari variabel yang dipilih.

```nginx
split_clients "${remote_addr}${request_uri}" $backend_pool {
    10%     canary;
    *       stable;
}

upstream stable {
    server 10.10.10.11:8080;
}

upstream canary {
    server 10.10.10.12:8080;
    server 10.10.10.13:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://$backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Penjelasan konfigurasi `split_clients`:

| Parameter                            | Fungsi                                                                     |
| ------------------------------------ | -------------------------------------------------------------------------- |
| `"${remote_addr}${request_uri}"`     | String yang di-hash untuk menentukan pembagian trafik                      |
| `$backend_pool`                      | Variabel yang menyimpan hasil split, berisi nama upstream tujuan           |
| `10%`                                | Persentase request yang diarahkan ke upstream `canary`                     |
| `*`                                  | Sisanya diarahkan ke upstream `stable`                                     |

Keunggulan `split_clients` dibanding `weight` adalah konsistensi. Client dengan kombinasi IP dan URI yang sama akan selalu diarahkan ke upstream yang sama selama persentase tidak berubah. Ini membantu debugging karena behavior tiap client lebih predictable.

### Konfigurasi Failover pada Canary

Tambahkan parameter health check supaya Nginx otomatis melewati server canary yang bermasalah dan mengarahkan request ke server stable.

```nginx
upstream stable {
    server 10.10.10.11:8080 max_fails=3 fail_timeout=30s;
}

upstream canary {
    server 10.10.10.12:8080 max_fails=2 fail_timeout=15s;
    server 10.10.10.13:8080 max_fails=2 fail_timeout=15s;

    # fallback ke server stable kalau semua canary down
    server 10.10.10.11:8080 backup;
}
```

Parameter `max_fails=2` dan `fail_timeout=15s` di canary sengaja dibuat lebih agresif dibanding server stable. Canary sedang dalam fase validasi, jadi lebih baik cepat mendeteksi masalah dan fallback ke server stable. Directive `backup` memastikan kalau semua server canary down, trafik otomatis dialihkan ke server stable.

### Monitoring Distribusi Trafik

Konfigurasi custom log format untuk memantau distribusi trafik dan performa masing-masing upstream.

```nginx
log_format canary '$remote_addr [$time_local] '
                  '"$request" $status $body_bytes_sent '
                  'upstream: $upstream_addr '
                  'response_time: $upstream_response_time '
                  'pool: $backend_pool';

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/canary.log canary;

    # ... konfigurasi location seperti sebelumnya
}
```

Variabel `$backend_pool` di log memudahkan filtering untuk melihat performa masing-masing pool secara terpisah.

Untuk memantau distribusi trafik secara real-time:

```bash
$ tail -f /var/log/nginx/canary.log | awk '{print $NF}' | sort | uniq -c
```

Output akan menunjukkan jumlah request per pool, misalnya:

```
     89 pool: stable
     11 pool: canary
```

Untuk melihat error rate di canary:

```bash
$ grep "pool: canary" /var/log/nginx/canary.log | awk '{print $5}' | sort | uniq -c | sort -rn
```

### Validasi dan Reload

Sebelum menerapkan setiap perubahan konfigurasi, selalu validasi syntax.

```bash
$ sudo nginx -t
```

Output yang diharapkan:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx untuk menerapkan konfigurasi. Proses reload tidak memutus koneksi yang sedang berjalan.

```bash
$ sudo systemctl reload nginx
```

### Tahapan Peningkatan Trafik

Setelah canary berjalan stabil di 10%, naikkan persentase secara bertahap. Setiap kenaikan, monitor minimal 1-2 jam sebelum menaikkan lagi.

| Tahap | Persentase Canary | Durasi Monitoring | Aksi Selanjutnya                         |
| ----- | ----------------- | ----------------- | ---------------------------------------- |
| 1     | 10%               | 2 jam             | Cek error rate, response time, log       |
| 2     | 25%               | 2 jam             | Bandingkan metrik canary vs stable       |
| 3     | 50%               | 4 jam             | Validasi di peak hour                    |
| 4     | 75%               | 4 jam             | Pastikan canary handle mayoritas trafik  |
| 5     | 100%              | 24 jam            | Cutover penuh, server lama jadi standby  |

Untuk menaikkan persentase, cukup ubah nilai di `split_clients` dan reload Nginx.

Contoh konfigurasi tahap 3 (50% canary):

```nginx
split_clients "${remote_addr}${request_uri}" $backend_pool {
    50%     canary;
    *       stable;
}
```

```bash
$ sudo nginx -t && sudo systemctl reload nginx
```

### Rollback

Kalau ditemukan masalah di canary, rollback cukup dengan mengembalikan semua trafik ke server stable.

```nginx
split_clients "${remote_addr}${request_uri}" $backend_pool {
    0%      canary;
    *       stable;
}
```

Atau cara paling cepat, langsung ubah `proxy_pass` ke upstream stable tanpa `split_clients`:

```nginx
location / {
    proxy_pass http://stable;
    # ... header seperti sebelumnya
}
```

```bash
$ sudo nginx -t && sudo systemctl reload nginx
```

Rollback efektif dalam hitungan detik. Request yang sedang diproses di canary tetap selesai, request baru langsung masuk ke stable.

### Cutover Penuh

Setelah canary berjalan 100% selama 24 jam tanpa masalah, saatnya cutover penuh. Ubah konfigurasi menjadi load balancer standar tanpa `split_clients`.

```nginx
upstream backend {
    server 10.10.10.12:8080 max_fails=3 fail_timeout=30s;
    server 10.10.10.13:8080 max_fails=3 fail_timeout=30s;

    # server lama tetap sebagai backup untuk safety net
    server 10.10.10.11:8080 backup;
}

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/access.log;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }
}
```

Server lama tetap dijadikan `backup` selama masa transisi. Setelah yakin infrastruktur baru stabil, server lama bisa di-decommission.

## Tantangan yang Dihadapi

Tantangan terbesar adalah soal **konsistensi session**. Saat trafik terbagi antara server stable dan canary, user yang login di server stable bisa saja request berikutnya masuk ke server canary. Kalau session disimpan di memori lokal masing-masing server, user tersebut akan kehilangan session dan diminta login ulang. Solusinya, saya memindahkan session storage ke **Redis** yang diakses oleh semua server. Dengan begitu, tidak peduli request masuk ke server mana, session tetap tersedia.

Hal lain yang tidak obvious adalah soal **menentukan persentase canary yang aman**. Terlalu kecil (misalnya 1%) membuat validasi tidak representatif karena sample terlalu sedikit untuk mendeteksi masalah. Terlalu besar membuat risiko meningkat. Saya memulai dari 10% karena cukup untuk mendapatkan sample yang bermakna tapi dampaknya masih terkontrol kalau ada masalah. Angka ini juga bergantung pada volume trafik, kalau trafik rendah, persentase awal mungkin perlu lebih besar supaya data monitoring cukup signifikan.

Monitoring juga butuh perhatian ekstra selama proses canary. Tidak cukup hanya melihat error rate secara agregat. Perlu membandingkan metrik antara pool stable dan canary secara terpisah, response time, error rate, memory usage, dan CPU di masing-masing server. Perbedaan kecil yang tidak terlihat di metrik agregat bisa jadi indikasi masalah yang akan membesar saat persentase dinaikkan. Saya menambahkan variabel `$backend_pool` di log format supaya bisa filter dan membandingkan performa kedua pool dengan mudah.

## Insight dan Pembelajaran

- **Canary deployment cocok untuk migrasi infrastruktur**: bukan hanya untuk deploy versi aplikasi baru, canary juga efektif untuk memvalidasi perubahan infrastruktur dengan risiko yang terkontrol
- **`split_clients` lebih predictable dibanding `weight`**: hash-based splitting memberikan konsistensi yang lebih baik untuk debugging dan monitoring karena client yang sama selalu masuk ke pool yang sama
- **Session harus centralized sebelum mulai canary**: pindahkan session storage ke Redis atau database sebelum membagi trafik ke multiple server. Ini prasyarat, bukan opsional
- **Monitor per-pool, bukan agregat**: metrik agregat bisa menyembunyikan masalah di canary karena volume-nya masih kecil. Selalu bandingkan performa canary vs stable secara terpisah
- **Rollback harus satu command**: pastikan proses rollback sudah disiapkan dan diuji sebelum memulai canary. Saat masalah terjadi, bukan waktu yang tepat untuk memikirkan cara rollback
- **Jangan skip tahapan**: godaan untuk langsung loncat dari 10% ke 100% itu besar kalau canary terlihat stabil. Tapi masalah tertentu baru muncul di volume trafik yang lebih tinggi, tetap naikkan secara bertahap

## Penutup

Canary deployment dengan Nginx memberikan cara yang aman untuk migrasi dari single server ke infrastruktur HA. Kuncinya ada di tiga hal, konfigurasi `split_clients` atau `weight` untuk mengontrol distribusi trafik, monitoring per-pool yang memadai untuk mendeteksi masalah sejak dini, dan proses rollback yang sudah disiapkan sebelum canary dimulai. Setelah canary berhasil di 100%, server lama tetap dijadikan backup selama masa transisi sebelum akhirnya di-decommission.

## Referensi

- [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html), diakses pada 2026-06-27
- [Module ngx_http_split_clients_module](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html), diakses pada 2026-06-27
- [Module ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html), diakses pada 2026-06-27
- [CanaryRelease - Martin Fowler](https://martinfowler.com/bliki/CanaryRelease.html), diakses pada 2026-06-27
