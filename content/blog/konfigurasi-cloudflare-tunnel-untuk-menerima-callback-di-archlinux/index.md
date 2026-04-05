+++
draft = false
date = '2026-04-05'
title = 'Konfigurasi Cloudflare Tunnel Untuk Menerima Callback Di Archlinux'
type = 'blog'
description = 'Cara menggunakan Cloudflare Tunnel di Archlinux untuk mengekspos local server agar bisa menerima callback dari service eksternal saat development.'
image = ''
tags = ['cloudflare', 'tunnel', 'networking', 'archlinux']
+++

## Latar Belakang

Saat develop aplikasi yang terintegrasi dengan service pihak ketiga -- payment gateway, webhook dari GitHub, notifikasi dari messaging platform -- ada satu kebutuhan yang sering muncul: **menerima callback**. Service eksternal perlu mengirim HTTP request ke endpoint kita, tapi masalahnya server development kita cuma jalan di `localhost`. Tidak bisa diakses dari internet.

Biasanya solusi yang dipakai adalah **ngrok** -- tool populer untuk mengekspos local server ke internet. Tapi ngrok punya limitasi di free tier-nya: URL berubah setiap restart, rate limit yang cukup ketat, dan koneksi yang kadang tidak stabil. **Cloudflare Tunnel** menawarkan alternatif yang lebih fleksibel -- bisa pakai quick tunnel untuk kebutuhan cepat, atau named tunnel dengan custom domain untuk setup yang lebih permanen.

## Permasalahan

Contoh kasusnya: saat mengembangkan integrasi payment gateway, server mereka perlu mengirim callback ke URL kita untuk notifikasi status pembayaran. Atau saat develop bot Telegram/Discord yang butuh webhook. Masalahnya:

- Local server jalan di `localhost:8080` -- tidak bisa diakses dari luar
- Jaringan berada di balik NAT tanpa public IP
- Port forwarding bukan opsi (atau terlalu ribet untuk kebutuhan development)
- Butuh HTTPS karena kebanyakan service pihak ketiga mensyaratkan URL callback dengan SSL

Yang dibutuhkan adalah cara cepat untuk membuat `localhost:8080` bisa diakses dari internet dengan URL yang valid dan HTTPS.

## Pendekatan Solusi

Cloudflare Tunnel punya dua mode yang relevan untuk kebutuhan ini:

| Mode | Kegunaan | Kelebihan | Kekurangan |
|------|----------|-----------|------------|
| **Quick Tunnel** | Testing cepat, one-off | Tanpa konfigurasi, langsung jalan | URL random, tidak persisten, max 200 concurrent request |
| **Named Tunnel** | Development jangka panjang | Custom domain, persisten, bisa jadi service | Butuh domain di Cloudflare dan setup awal |

Cara kerjanya sederhana -- `cloudflared` (daemon dari Cloudflare) membuat **koneksi outbound** dari mesin kita ke jaringan Cloudflare. Traffic dari internet masuk lewat Cloudflare, lalu diteruskan ke local server melalui tunnel tersebut. Karena koneksinya outbound, tidak perlu buka port apapun di firewall.

Untuk kebutuhan menerima callback saat development, **quick tunnel** biasanya sudah cukup. Tapi kalau butuh URL yang tetap (misalnya untuk mendaftarkan webhook yang tidak bisa sering diganti), **named tunnel** dengan custom domain lebih cocok.

## Implementasi Teknis

### Instalasi

`cloudflared` tersedia di repository resmi Archlinux:

```
$ sudo pacman -S cloudflared
```

Verifikasi instalasi:

```
$ cloudflared --version
```

### Quick Tunnel (Cara Cepat)

Ini cara paling cepat untuk mengekspos local server. Misalnya ada server jalan di port 8080:

```
$ cloudflared tunnel --url http://localhost:8080
```

Output-nya akan seperti ini:

```
2026-04-05T10:00:00Z INF +--------------------------------------------------------------------------------------------+
2026-04-05T10:00:00Z INF |  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
2026-04-05T10:00:00Z INF |  https://random-words-here.trycloudflare.com                                               |
2026-04-05T10:00:00Z INF +--------------------------------------------------------------------------------------------+
```

URL `https://random-words-here.trycloudflare.com` langsung bisa dipakai sebagai callback URL. HTTPS sudah otomatis ter-handle oleh Cloudflare.

Sekarang tinggal daftarkan URL tersebut ke service pihak ketiga sebagai callback endpoint. Misalnya untuk payment gateway:

```
callback_url: https://random-words-here.trycloudflare.com/api/payment/callback
```

Quick tunnel cocok untuk:
- Testing integrasi webhook sekali jalan
- Demo ke rekan kerja
- Debug callback dari service eksternal

### Named Tunnel (Custom Domain)

Untuk kebutuhan yang lebih permanen -- misalnya development yang berjalan berminggu-minggu dan callback URL tidak boleh berubah -- gunakan named tunnel dengan custom domain.

#### 1. Login ke Cloudflare

```
$ cloudflared tunnel login
```

Browser akan terbuka untuk autentikasi. Pilih domain yang akan digunakan. Setelah berhasil, file `cert.pem` tersimpan di `~/.cloudflared/`.

#### 2. Buat Tunnel

```
$ cloudflared tunnel create dev-callback
```

Output:

```
Tunnel credentials written to /home/user/.cloudflared/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json.
Created tunnel dev-callback with id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Catat UUID tunnel yang dihasilkan.

#### 3. Buat Konfigurasi

Buat file `~/.cloudflared/config.yml`:

```yaml
tunnel: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
credentials-file: /home/user/.cloudflared/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json

ingress:
  - hostname: dev.mnabila.com
    service: http://localhost:8080
  - service: http_status:404
```

Bagian `ingress` mendefinisikan routing -- request ke `dev.mnabila.com` diteruskan ke `localhost:8080`. Rule terakhir (`http_status:404`) adalah catch-all yang wajib ada.

Kita juga bisa routing ke beberapa service sekaligus:

```yaml
ingress:
  - hostname: dev.mnabila.com
    service: http://localhost:8080
  - hostname: api.mnabila.com
    service: http://localhost:3000
  - service: http_status:404
```

#### 4. Buat DNS Record

```
$ cloudflared tunnel route dns dev-callback dev.mnabila.com
```

Perintah ini membuat CNAME record di Cloudflare DNS yang mengarahkan `dev.mnabila.com` ke tunnel.

#### 5. Jalankan Tunnel

```
$ cloudflared tunnel run dev-callback
```

Sekarang `https://dev.mnabila.com` mengarah ke `localhost:8080`. URL ini persisten -- tidak berubah meskipun `cloudflared` di-restart.

### Menjalankan Sebagai Service

Untuk named tunnel yang perlu selalu aktif, jalankan sebagai systemd service:

```
$ sudo cloudflared service install
$ sudo systemctl enable --now cloudflared
```

Pastikan file konfigurasi sudah ada di `/etc/cloudflared/config.yml` atau di path default `~/.cloudflared/config.yml`.

### Tips: Debugging Callback

Saat debugging callback, sangat membantu untuk melihat request yang masuk. Kombinasikan Cloudflare Tunnel dengan tool logging sederhana. Misalnya, tambahkan middleware logging di aplikasi, atau gunakan `cloudflared` dengan flag `--loglevel debug` untuk melihat semua request yang melewati tunnel:

```
$ cloudflared tunnel --url http://localhost:8080 --loglevel debug
```

## Tantangan yang Dihadapi

Tantangan utama saat menggunakan quick tunnel adalah **URL yang berubah setiap restart**. Ini berarti setiap kali `cloudflared` dihentikan dan dijalankan ulang, kita harus update callback URL di service pihak ketiga. Untuk development yang intens, ini cukup mengganggu -- solusinya adalah beralih ke named tunnel.

Tantangan lain adalah **rate limit pada quick tunnel** -- maksimal 200 concurrent request. Untuk menerima callback satu-dua request, ini bukan masalah. Tapi kalau testing load atau simulasi banyak callback sekaligus, bisa kena limit dan mendapat response `429 Too Many Requests`.

Satu hal yang perlu diperhatikan juga -- quick tunnel **tidak mendukung Server-Sent Events (SSE)**. Jika aplikasi menggunakan SSE untuk real-time updates, gunakan named tunnel sebagai gantinya.

## Insight dan Pembelajaran

Beberapa insight setelah menggunakan Cloudflare Tunnel untuk development:

- **Quick tunnel adalah pengganti ngrok yang solid** -- satu perintah, langsung dapat HTTPS URL. Tidak perlu signup atau API key untuk quick tunnel.
- **Named tunnel cocok untuk project jangka panjang** -- investasi 5 menit setup di awal menghemat banyak waktu karena URL tidak perlu diganti-ganti.
- **Ingress rules sangat fleksibel** -- satu tunnel bisa routing ke beberapa service lokal sekaligus. Cocok kalau develop microservices yang butuh beberapa endpoint callback berbeda.
- **Koneksi outbound-only adalah keunggulan keamanan** -- tidak perlu buka port apapun di firewall. Semua koneksi diinisiasi dari mesin kita ke Cloudflare, bukan sebaliknya.
- **Named tunnel butuh domain yang di-manage Cloudflare** -- kalau belum punya, ini bisa jadi blocker. Untuk yang punya domain, quick tunnel bisa jadi alternatif karena langsung jalan tanpa konfigurasi apapun.

## Penutup

Cloudflare Tunnel menjawab kebutuhan klasik developer: mengekspos local server ke internet untuk menerima callback. Quick tunnel cocok untuk kebutuhan cepat dan one-off, sementara named tunnel memberikan URL yang persisten dengan custom domain untuk development jangka panjang. Dibanding solusi sejenis, Cloudflare Tunnel punya keunggulan di sisi keamanan (outbound-only, HTTPS otomatis) dan fleksibilitas (multi-service ingress, bisa jadi systemd service). Untuk yang sering develop integrasi dengan webhook atau callback dari service pihak ketiga, tool ini wajib ada di toolbox.

## Referensi

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) -- Diakses pada 2026-04-05
- [Cloudflare - Create a Local Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/) -- Diakses pada 2026-04-05
- [Cloudflare - TryCloudflare (Quick Tunnels)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/do-more-with-tunnels/trycloudflare/) -- Diakses pada 2026-04-05
- [Arch Wiki - Cloudflared](https://wiki.archlinux.org/title/Cloudflared) -- Diakses pada 2026-04-05
