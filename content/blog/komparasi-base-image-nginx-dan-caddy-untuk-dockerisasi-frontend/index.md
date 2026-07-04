+++
draft = true
date = '2026-06-27'
title = 'Komparasi Base Image Nginx dan Caddy untuk Dockerisasi Frontend'
type = 'blog'
description = 'Membandingkan Nginx dan Caddy sebagai base image Docker untuk menyajikan static frontend hasil build, dari ukuran image hingga kemudahan konfigurasi'
image = ''
tags = ['nginx', 'caddy', 'docker', 'frontend', 'vite']
+++

## Latar Belakang

Hampir semua project frontend modern, entah itu **React**, **Vue**, **Svelte**, atau framework lain yang pakai **Vite**, output akhirnya adalah kumpulan file static (HTML, CSS, JS). File-file ini perlu disajikan oleh web server di dalam container Docker, dan dua pilihan yang paling umum dipakai sebagai base image adalah **Nginx** dan **Caddy**.

Saya sudah cukup lama pakai Nginx sebagai default pilihan untuk dockerisasi frontend. Tapi belakangan, saya mulai mempertimbangkan Caddy setelah melihat beberapa project yang menggunakannya dengan konfigurasi yang jauh lebih ringkas. Sebelum memutuskan apakah perlu migrasi atau tetap dengan Nginx, saya perlu data perbandingan yang konkret.

## Permasalahan

Memilih base image untuk frontend di Docker bukan sekadar soal preferensi:

- **Ukuran image berpengaruh ke deployment speed**: image yang besar memperlambat pull di CI/CD pipeline dan konsumsi storage di registry
- **Konfigurasi SPA routing tidak trivial**: Single Page Application butuh fallback ke `index.html` untuk semua route yang tidak match file static, dan ini harus dikonfigurasi manual
- **Cache strategy untuk static asset**: file hasil build punya hash di nama file (`app.a1b2c3.js`) yang harus di-cache agresif, tapi `index.html` tidak boleh di-cache
- **Kompresi bawaan berbeda**: Nginx hanya support gzip secara default, Brotli butuh module tambahan. Caddy support keduanya out of the box
- **HTTPS otomatis bisa jadi kelebihan atau komplikasi**: Caddy otomatis provision HTTPS, tapi di balik reverse proxy atau load balancer, fitur ini justru menambah kompleksitas
- **Trade-off antara fleksibilitas dan kesederhanaan**: Nginx sangat fleksibel tapi konfigurasinya verbose, Caddy simpel tapi opsinya lebih terbatas

## Pendekatan Solusi

Perbandingan kedua web server sebagai base image Docker untuk frontend:

| Aspek                  | Nginx                                                      | Caddy                                                    |
| ---------------------- | ---------------------------------------------------------- | -------------------------------------------------------- |
| **Base image size**    | `nginx:alpine` ~20 MB                                     | `caddy:alpine` ~25 MB                                   |
| **Konfigurasi**        | Verbose, file `.conf` dengan directive block               | Ringkas, **Caddyfile** dengan sintaks deklaratif         |
| **SPA fallback**       | Perlu `try_files` manual                                   | Perlu `try_files` manual                                 |
| **Gzip**               | Built-in, perlu enable manual                              | Built-in, aktif secara default                           |
| **Brotli**             | Butuh module tambahan atau custom build                    | Built-in, aktif secara default                           |
| **HTTPS otomatis**     | Tidak ada                                                  | Ada, otomatis provision Let's Encrypt                    |
| **Reverse proxy**      | Sangat fleksibel, banyak directive                         | Simpel, cukup `reverse_proxy` satu baris                 |
| **Komunitas & docs**   | Sangat besar, banyak referensi                             | Lebih kecil, tapi dokumentasi resmi sangat baik          |
| **Ekosistem production** | Standar industri, banyak tooling                         | Makin populer, tapi adopsi enterprise masih di bawah Nginx |

Kedua opsi punya kelebihan masing-masing. Nginx cocok untuk production yang butuh konfigurasi granular dan sudah punya ekosistem monitoring yang mature. Caddy cocok untuk deployment yang mengutamakan kecepatan setup dan fitur bawaan yang sudah cukup lengkap tanpa konfigurasi tambahan.

Untuk perbandingan ini, saya menggunakan aplikasi frontend berbasis **React + Vite** sebagai contoh kasus.

## Implementasi Teknis

### Build Stage (Sama untuk Kedua Image)

Tahap build menggunakan **multi-stage build**. Stage pertama compile aplikasi frontend, stage kedua hanya menyalin hasil build ke web server image. Ini menjaga ukuran final image tetap kecil karena `node_modules` dan source code tidak ikut masuk.

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

Stage `build` ini dipakai oleh kedua Dockerfile, baik yang pakai Nginx maupun Caddy.

### Dockerfile dengan Nginx

Buat file `Dockerfile.nginx` untuk versi Nginx.

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

Nginx butuh file konfigurasi terpisah. Buat file `nginx.conf` di root project.

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # cache agresif untuk asset dengan hash di nama file
    location ~* \.(?:js|css|woff2?|ttf|eot|svg|png|jpg|jpeg|gif|ico|webp)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # jangan cache index.html
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml text/javascript image/svg+xml;
    gzip_min_length 256;
    gzip_vary on;
}
```

Penjelasan konfigurasi:

| Directive                     | Fungsi                                                                    |
| ----------------------------- | ------------------------------------------------------------------------- |
| `try_files $uri $uri/ /index.html` | SPA fallback, route yang tidak match file static dikembalikan ke `index.html` |
| `expires 1y`                  | Cache static asset selama 1 tahun karena nama file sudah punya hash       |
| `Cache-Control "public, immutable"` | Browser boleh cache dan tidak perlu revalidate                       |
| `gzip on`                     | Aktifkan kompresi gzip                                                    |
| `gzip_min_length 256`         | Hanya compress file di atas 256 bytes                                     |

### Dockerfile dengan Caddy

Buat file `Dockerfile.caddy` untuk versi Caddy.

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM caddy:alpine

COPY --from=build /app/dist /srv
COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80
```

Buat file `Caddyfile` di root project.

```caddyfile
:80 {
    root * /srv
    file_server

    # SPA fallback
    try_files {path} /index.html

    # cache agresif untuk asset dengan hash
    @static path *.js *.css *.woff2 *.woff *.ttf *.eot *.svg *.png *.jpg *.jpeg *.gif *.ico *.webp
    header @static Cache-Control "public, max-age=31536000, immutable"

    # jangan cache index.html
    @html path /index.html
    header @html Cache-Control "no-store, no-cache, must-revalidate"
}
```

Perbandingan langsung: konfigurasi Caddy membutuhkan 12 baris untuk fungsionalitas yang sama dengan 20 baris di Nginx. Gzip dan Brotli sudah aktif secara default di Caddy tanpa perlu dikonfigurasi. Prefix `:80` memastikan Caddy berjalan di port 80 tanpa mencoba provision HTTPS.

### Build dan Perbandingan Ukuran Image

Build kedua image dan bandingkan ukurannya.

```bash
$ docker build -f Dockerfile.nginx -t frontend:nginx .
$ docker build -f Dockerfile.caddy -t frontend:caddy .
```

Cek ukuran masing-masing image.

```bash
$ docker images --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}" | grep frontend
```

Output yang diharapkan (ukuran bisa bervariasi tergantung aplikasi):

```
frontend:nginx    25MB
frontend:caddy    30MB
```

Selisih ukuran sekitar 5 MB, tidak signifikan untuk kebanyakan use case. Faktor yang lebih berpengaruh ke ukuran final image adalah hasil build frontend itu sendiri, bukan base image-nya.

### Verifikasi SPA Routing

Jalankan container dan pastikan SPA routing berfungsi. Route yang tidak ada sebagai file static harus mengembalikan `index.html`, bukan 404.

```bash
$ docker run -d -p 3001:80 --name test-nginx frontend:nginx
$ docker run -d -p 3002:80 --name test-caddy frontend:caddy
```

Test SPA fallback di kedua container.

```bash
$ curl -s -o /dev/null -w "%{http_code}" http://localhost:3001/dashboard
$ curl -s -o /dev/null -w "%{http_code}" http://localhost:3002/dashboard
```

Kedua command harus mengembalikan `200`. Kalau mengembalikan `404`, berarti konfigurasi `try_files` belum benar.

### Verifikasi Kompresi

Cek apakah response sudah terkompresi.

```bash
$ curl -s -H "Accept-Encoding: gzip" -o /dev/null -w "%{size_download}" http://localhost:3001/assets/index.js
$ curl -s -H "Accept-Encoding: br" -o /dev/null -w "%{size_download}" http://localhost:3002/assets/index.js
```

Caddy akan mengembalikan response yang terkompresi dengan Brotli (lebih kecil dari gzip) tanpa konfigurasi tambahan. Untuk mendapatkan Brotli di Nginx, perlu build custom image dengan module `ngx_brotli`, yang menambah kompleksitas maintenance.

### Verifikasi Cache Header

Pastikan cache header sudah benar untuk static asset dan `index.html`.

```bash
$ curl -sI http://localhost:3001/assets/index.js | grep -i cache-control
$ curl -sI http://localhost:3001/index.html | grep -i cache-control
```

Output yang diharapkan:

```
Cache-Control: public, immutable
Cache-Control: no-store, no-cache, must-revalidate
```

Static asset di-cache selamanya (karena hash di nama file menjamin uniqueness), sementara `index.html` tidak pernah di-cache supaya browser selalu mendapat versi terbaru.

### Cleanup

Setelah selesai testing, hentikan dan hapus container.

```bash
$ docker stop test-nginx test-caddy
$ docker rm test-nginx test-caddy
```

## Tantangan yang Dihadapi

Tantangan pertama adalah soal **Brotli di Nginx**. Brotli memberikan kompresi yang lebih baik dibanding gzip (rata-rata 15-20% lebih kecil), dan semua browser modern sudah mendukungnya. Tapi Nginx tidak menyertakan module Brotli secara default. Untuk mengaktifkannya, perlu build custom Nginx image dengan module `ngx_brotli`, yang berarti maintain Dockerfile tersendiri untuk base image Nginx. Di sisi lain, Caddy sudah menyertakan Brotli out of the box dan otomatis memilih kompresi terbaik berdasarkan header `Accept-Encoding` dari client. Untuk production yang peduli dengan ukuran transfer, ini keunggulan Caddy yang cukup signifikan.

Tantangan kedua adalah **HTTPS otomatis Caddy di balik reverse proxy**. Secara default, Caddy mencoba provision sertifikat HTTPS via Let's Encrypt saat menerima request di domain yang belum punya sertifikat. Kalau container Caddy berjalan di balik load balancer atau reverse proxy yang sudah handle TLS termination, fitur ini justru menimbulkan masalah karena Caddy mencoba bind port 443 dan gagal mendapatkan sertifikat. Solusinya simpel, pastikan Caddyfile menggunakan `:80` sebagai alamat listener, bukan domain name. Dengan begitu, Caddy tahu bahwa HTTPS tidak diperlukan dan berjalan sebagai plain HTTP server.

Satu hal lagi yang perlu diperhatikan adalah soal **default behavior** yang berbeda. Nginx secara default tidak menjalankan gzip, tidak ada SPA fallback, dan tidak ada cache header. Semua harus dikonfigurasi manual. Caddy lebih opinionated, gzip dan Brotli sudah aktif, tapi `try_files` untuk SPA tetap harus ditambahkan manual. Perbedaan ini bisa jadi jebakan kalau terbiasa dengan satu web server lalu pindah ke yang lain tanpa cek ulang behavior default-nya.

## Insight dan Pembelajaran

- **Untuk frontend static, perbedaan performa Nginx dan Caddy tidak signifikan**: keduanya menyajikan file static dengan sangat cepat. Bottleneck biasanya ada di network, bukan di web server
- **Caddy menang di kesederhanaan konfigurasi**: Caddyfile lebih ringkas dan readable, terutama untuk use case yang umum seperti SPA routing dan reverse proxy. Cocok untuk tim yang ingin reduce cognitive load di infra
- **Nginx menang di fleksibilitas dan ekosistem**: kalau butuh konfigurasi yang sangat spesifik (rate limiting, complex rewrite rules, custom header manipulation), Nginx punya lebih banyak directive dan referensi
- **Brotli out of the box adalah keunggulan nyata Caddy**: tidak perlu custom build atau module tambahan. Untuk frontend yang peduli dengan ukuran transfer, ini bisa menghemat bandwidth secara signifikan
- **Multi-stage build adalah keharusan**: tanpa multi-stage build, image bisa membengkak ratusan MB karena `node_modules` dan source code ikut masuk. Build stage yang terpisah menjaga image tetap kecil
- **Jangan cache `index.html`**: ini kesalahan umum. Kalau `index.html` ter-cache di browser, user tidak akan mendapat update terbaru meskipun asset JS/CSS sudah berubah

## Penutup

Kedua web server bisa menjalankan tugas menyajikan frontend static dengan baik. Pilihan antara Nginx dan Caddy lebih tergantung pada kebutuhan spesifik, Nginx untuk production yang butuh konfigurasi granular dan sudah punya ekosistem yang established, Caddy untuk deployment yang mengutamakan kesederhanaan dan fitur bawaan yang sudah lengkap. Untuk project baru yang tidak punya constraint khusus, Caddy menawarkan developer experience yang lebih baik dengan konfigurasi yang lebih sedikit.

## Referensi

- [Nginx Documentation](https://nginx.org/en/docs/), diakses pada 2026-06-27
- [Caddy Documentation](https://caddyserver.com/docs/), diakses pada 2026-06-27
- [Nginx Docker Hub](https://hub.docker.com/_/nginx), diakses pada 2026-06-27
- [Caddy Docker Hub](https://hub.docker.com/_/caddy), diakses pada 2026-06-27
- [Deploying a Static Site - Vite](https://vite.dev/guide/static-deploy.html), diakses pada 2026-06-27
