+++
draft = false
date = '2026-06-27'
title = 'Konfigurasi Nginx Load Balancer dengan Round Robin di Ubuntu Server'
type = 'blog'
description = 'Setup Nginx sebagai load balancer dengan metode round robin untuk mendistribusikan trafik ke beberapa backend dan menjaga ketersediaan layanan di Ubuntu Server 22.04'
image = ''
tags = ['nginx', 'load-balancer', 'ubuntu', 'high-availability']
+++

## Latar Belakang

Layanan yang saya kelola mulai mengalami lonjakan trafik yang tidak bisa ditangani oleh satu backend server saja. Awalnya arsitekturnya sederhana, satu **Nginx** sebagai reverse proxy mengarahkan semua request ke satu backend. Selama trafik masih rendah, setup ini tidak bermasalah. Tapi begitu jumlah concurrent request naik, backend mulai kewalahan dan response time melonjak.

Masalah lain yang tidak kalah kritis adalah soal availability. Kalau backend satu-satunya itu down, entah karena deployment, crash, atau maintenance, seluruh layanan ikut mati. Tidak ada fallback, tidak ada redundansi. Pengguna langsung kena dampaknya.

## Permasalahan

Mengandalkan satu backend server untuk menangani semua trafik mulai menimbulkan masalah:

- **Single point of failure**: kalau backend down, tidak ada yang menggantikan dan layanan langsung tidak bisa diakses
- **Kapasitas terbatas**: satu server punya batas resource, menambah CPU atau RAM secara vertikal ada limitnya dan biayanya makin mahal
- **Response time naik saat peak hour**: semua request masuk ke satu titik yang sama, server overloaded dan latency meningkat
- **Deployment berisiko**: setiap kali deploy versi baru, layanan harus down karena tidak ada server lain yang bisa handle trafik sementara
- **Tidak bisa horizontal scaling**: tanpa load balancer, menambah server baru tidak ada artinya karena trafik tetap hanya diarahkan ke satu backend

## Pendekatan Solusi

Ada beberapa metode load balancing yang didukung Nginx:

| Pendekatan          | Kelebihan                                                    | Kekurangan                                                             |
| ------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------- |
| **Round Robin**     | Default Nginx, distribusi merata, setup paling simpel        | Tidak mempertimbangkan beban aktual tiap server                        |
| **Least Connected** | Request diarahkan ke server dengan koneksi paling sedikit    | Kurang optimal kalau response time antar server sangat bervariasi      |
| **IP Hash**         | Client selalu diarahkan ke server yang sama (sticky session) | Distribusi bisa tidak merata kalau ada IP yang generate trafik tinggi  |
| **Weighted**        | Bisa atur proporsi trafik sesuai kapasitas server            | Perlu tuning manual, harus tahu kapasitas masing-masing server         |

Saya pilih **Round Robin** karena paling simpel dan sesuai dengan kebutuhan saat ini. Semua backend server punya spesifikasi yang sama, jadi distribusi merata sudah cukup. Tidak perlu kompleksitas tambahan seperti sticky session karena aplikasi sudah menyimpan session di **Redis**, bukan di server lokal. Trafik didistribusikan secara merata ke beberapa backend, dan kalau salah satu backend down, Nginx otomatis mengarahkan request ke backend yang masih hidup. Setup ini dilakukan di Ubuntu Server 22.04.

Arsitektur yang dibangun:

| Komponen         | IP Address     | Peran                                              |
| ---------------- | -------------- | -------------------------------------------------- |
| **Load Balancer** | `10.10.10.10` | Nginx sebagai reverse proxy dan load balancer      |
| **Backend 1**    | `10.10.10.11` | Application server, handle request dari load balancer |
| **Backend 2**    | `10.10.10.12` | Application server, handle request dari load balancer |
| **Backend 3**    | `10.10.10.13` | Application server, handle request dari load balancer |

## Implementasi Teknis

### Instalasi Nginx

Update repository dan install Nginx di server yang akan menjadi load balancer.

```bash
$ sudo apt update
$ sudo apt install -y nginx
```

Pastikan Nginx sudah berjalan dan aktif.

```bash
$ sudo systemctl enable --now nginx
$ sudo systemctl status nginx
```

Output yang diharapkan menunjukkan status `active (running)`.

### Konfigurasi Upstream

Buat file konfigurasi baru untuk load balancer. Saya pisahkan dari konfigurasi default supaya lebih rapi dan mudah dikelola.

```bash
$ sudo nano /etc/nginx/conf.d/loadbalancer.conf
```

Isi konfigurasi upstream yang mendefinisikan daftar backend server.

```nginx
upstream backend_servers {
    server 10.10.10.11:8080;
    server 10.10.10.12:8080;
    server 10.10.10.13:8080;
}
```

Block `upstream` ini adalah inti dari load balancing di Nginx. Karena tidak ada directive khusus yang ditambahkan, Nginx otomatis menggunakan metode **round robin** sebagai default. Request pertama masuk ke backend 1, request kedua ke backend 2, request ketiga ke backend 3, lalu kembali ke backend 1, begitu seterusnya.

### Konfigurasi Reverse Proxy

Tambahkan block `server` di file yang sama untuk menerima request dari client dan meneruskannya ke upstream.

```nginx
upstream backend_servers {
    server 10.10.10.11:8080;
    server 10.10.10.12:8080;
    server 10.10.10.13:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Penjelasan header yang dikirim ke backend:

| Header                | Fungsi                                                              |
| --------------------- | ------------------------------------------------------------------- |
| `Host`                | Meneruskan hostname asli dari request client                        |
| `X-Real-IP`           | Mengirim IP asli client, bukan IP load balancer                     |
| `X-Forwarded-For`     | Chain IP address dari client hingga proxy terakhir                  |
| `X-Forwarded-Proto`   | Memberitahu backend apakah request asli menggunakan HTTP atau HTTPS |

Header ini penting supaya backend tahu siapa client sebenarnya. Tanpa header ini, backend hanya melihat IP load balancer untuk semua request.

### Konfigurasi Passive Health Check dan Failover

Nginx open source tidak punya fitur **active health check** seperti NGINX Plus yang bisa secara berkala mengirim probe ke backend. Tapi Nginx punya mekanisme **passive health check** yang cukup memadai untuk mendeteksi backend yang bermasalah.

Update block `upstream` dengan parameter failover.

```nginx
upstream backend_servers {
    server 10.10.10.11:8080 max_fails=3 fail_timeout=30s;
    server 10.10.10.12:8080 max_fails=3 fail_timeout=30s;
    server 10.10.10.13:8080 max_fails=3 fail_timeout=30s;
}
```

Penjelasan parameter:

| Parameter      | Fungsi                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------- |
| `max_fails`    | Jumlah request gagal berturut-turut sebelum server dianggap down                            |
| `fail_timeout` | Durasi server dianggap down setelah mencapai `max_fails`, sekaligus window untuk menghitung gagal |

Dengan konfigurasi di atas, kalau satu backend gagal merespons 3 kali berturut-turut, Nginx akan menandainya sebagai down selama 30 detik. Selama periode itu, semua request diarahkan ke backend lain yang masih aktif. Setelah 30 detik, Nginx mencoba mengirim request ke backend tersebut lagi untuk mengecek apakah sudah pulih.

### Konfigurasi Timeout

Tambahkan pengaturan timeout di block `location` untuk mengontrol berapa lama Nginx menunggu respons dari backend.

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend_servers;
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

| Parameter                  | Fungsi                                                                              |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `proxy_connect_timeout`    | Batas waktu untuk membuat koneksi ke backend                                        |
| `proxy_send_timeout`       | Batas waktu untuk mengirim request ke backend                                       |
| `proxy_read_timeout`       | Batas waktu untuk membaca respons dari backend                                      |
| `proxy_next_upstream`      | Kondisi yang memicu Nginx mencoba backend lain (error, timeout, HTTP 502/503/504)   |
| `proxy_next_upstream_tries`| Maksimal berapa kali Nginx mencoba backend lain sebelum mengembalikan error ke client |

Directive `proxy_next_upstream` adalah kunci failover di level request. Kalau backend pertama mengembalikan error 502, Nginx tidak langsung mengembalikan error tersebut ke client, melainkan mencoba request ke backend berikutnya.

### Nonaktifkan Default Site

Supaya tidak bentrok dengan konfigurasi default Nginx, nonaktifkan default site.

```bash
$ sudo rm /etc/nginx/sites-enabled/default
```

### Validasi dan Reload

Sebelum menerapkan konfigurasi, pastikan syntax sudah benar.

```bash
$ sudo nginx -t
```

Output yang diharapkan:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Kalau tidak ada error, reload Nginx untuk menerapkan konfigurasi baru.

```bash
$ sudo systemctl reload nginx
```

### Pengujian Load Balancer

Untuk memverifikasi bahwa round robin berjalan sesuai harapan, kirim beberapa request berturut-turut dan perhatikan distribusinya.

```bash
$ for i in {1..6}; do curl -s http://10.10.10.10 | head -1; done
```

Kalau masing-masing backend mengembalikan identifier yang berbeda (misalnya hostname atau response body yang unik), seharusnya terlihat pola distribusi merata ke tiga backend secara bergantian.

Untuk menguji failover, matikan salah satu backend dan kirim request lagi.

```bash
$ for i in {1..6}; do curl -s -o /dev/null -w "%{http_code}\n" http://10.10.10.10; done
```

Semua request seharusnya tetap mengembalikan status `200` meskipun salah satu backend sudah dimatikan. Nginx secara otomatis melewati backend yang down dan mendistribusikan request ke backend yang tersisa.

### Monitoring dengan Access Log

Untuk monitoring distribusi trafik, bisa ditambahkan variabel `$upstream_addr` di format log Nginx.

```nginx
log_format upstreamlog '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       'upstream: $upstream_addr response_time: $upstream_response_time';

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/loadbalancer.log upstreamlog;

    location / {
        proxy_pass http://backend_servers;
        # ... header dan timeout seperti sebelumnya
    }
}
```

Dengan format log ini, setiap entry di access log menunjukkan backend mana yang handle request tersebut beserta response time-nya. Berguna untuk memverifikasi distribusi dan mendeteksi backend yang lambat.

```bash
$ tail -f /var/log/nginx/loadbalancer.log
```

Output log akan menampilkan informasi seperti `upstream: 10.10.10.11:8080 response_time: 0.005` di setiap baris.

## Tantangan yang Dihadapi

Tantangan pertama yang saya temui adalah soal **health check**. Nginx open source hanya punya passive health check, artinya Nginx baru tahu backend down setelah ada request yang gagal. Request pertama yang mengenai backend bermasalah tetap akan mengalami delay atau error sebelum Nginx menandainya sebagai down dan memindahkan request ke backend lain. Solusinya, saya mengkombinasikan `proxy_next_upstream` supaya request yang gagal langsung dicoba ke backend berikutnya tanpa mengembalikan error ke client. Bukan solusi sempurna, tapi cukup efektif untuk kebutuhan saat ini. Kalau butuh active health check yang benar-benar probing backend secara berkala, opsinya adalah menggunakan NGINX Plus atau menambahkan module pihak ketiga seperti `nginx_upstream_check_module`.

Hal lain yang perlu diperhatikan adalah soal **session management**. Round robin mendistribusikan request secara merata tanpa mempertimbangkan afinitas client. Artinya, request pertama dan kedua dari client yang sama bisa masuk ke backend yang berbeda. Kalau aplikasi menyimpan session di memori lokal server, user akan kehilangan session setiap kali request-nya pindah backend. Solusi yang saya terapkan adalah memindahkan session storage ke Redis yang terpisah dari backend. Semua backend mengakses session dari satu Redis instance yang sama, jadi tidak peduli request masuk ke backend mana, session tetap konsisten.

Konfigurasi `fail_timeout` juga butuh pertimbangan. Nilai yang terlalu kecil membuat Nginx terlalu agresif menandai backend sebagai down, padahal mungkin backend hanya mengalami spike sesaat. Sebaliknya, nilai yang terlalu besar membuat Nginx terlalu lama mengirim request ke backend yang sudah bermasalah. Saya menggunakan `max_fails=3` dengan `fail_timeout=30s` sebagai titik awal, lalu menyesuaikan berdasarkan monitoring di production.

## Insight dan Pembelajaran

- **Round robin cukup untuk backend homogen**: kalau semua backend punya spesifikasi yang sama, tidak perlu algoritma yang lebih kompleks. Round robin mendistribusikan trafik secara merata dan mudah di-debug
- **Passive health check bisa diandalkan dengan konfigurasi yang tepat**: kombinasi `max_fails`, `fail_timeout`, dan `proxy_next_upstream` sudah cukup untuk menangani failover dasar tanpa perlu NGINX Plus
- **Session harus stateless atau centralized**: load balancing tidak akan berjalan mulus kalau aplikasi masih menyimpan session di memori lokal. Pindahkan ke Redis atau database yang bisa diakses semua backend
- **Monitoring upstream itu wajib**: tanpa log yang mencatat `$upstream_addr` dan `$upstream_response_time`, tidak ada cara mudah untuk mengetahui distribusi trafik dan performa masing-masing backend
- **Timeout perlu di-tune sesuai karakteristik aplikasi**: default timeout Nginx adalah 60 detik, yang terlalu lama untuk kebanyakan web application. Sesuaikan dengan SLA dan response time normal backend

## Penutup

Setup Nginx sebagai load balancer dengan round robin cukup simpel dan efektif untuk mendistribusikan trafik ke beberapa backend. Dengan tambahan konfigurasi passive health check dan `proxy_next_upstream`, failover bisa berjalan otomatis tanpa intervensi manual. Untuk production yang lebih demanding, pertimbangkan upgrade ke NGINX Plus untuk active health check atau tambahkan monitoring layer seperti Prometheus dan Grafana untuk observability yang lebih baik.

## Referensi

- [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html), diakses pada 2026-06-27
- [Module ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html), diakses pada 2026-06-27
- [HTTP Load Balancing - NGINX Admin Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/), diakses pada 2026-06-27
