+++
draft = false
date = '2026-05-14'
title = 'Konfigurasi Redis Replication Master-Slave di Ubuntu Server'
type = 'blog'
description = 'Konfigurasi Redis replication master-slave di Ubuntu Server 22.04 untuk high availability dan menangani trafik tinggi pada aplikasi production'
image = ''
tags = ['redis', 'replication', 'ubuntu', 'server']
+++

## Latar Belakang

Aplikasi production yang saya kelola mulai menunjukkan gejala bottleneck di layer cache. Semua operasi read dan write Redis masuk ke satu instance yang sama. Selama trafik normal tidak ada masalah, tapi begitu peak hour tiba, response time naik drastis dan terkadang ada beberapa request yang timeout karena Redis tidak sanggup menangani semua beban sendirian.

Setelah saya cek, Mayoritas operasi ke Redis adalah read, session lookup, cache hit, rate limiting check. Write hanya terjadi saat ada update data atau cache invalidation. Artinya, kalau beban read bisa didistribusikan ke node lain, master tidak perlu kerja sendirian lagi.

Solusi yang saya ambil adalah setup **Redis replication** dengan skema master-slave. Master fokus handle write, slave fokus handle read. Selain performa lebih baik, slave juga berfungsi sebagai fallback kalau master tiba-tiba down. Setup ini saya lakukan di dua Ubuntu Server 22.04 dalam satu network.

## Permasalahan

Mengandalkan satu instance Redis untuk semua operasi mulai jadi masalah:

- **Semua operasi numpuk di satu node**: read dan write rebutan resource di instance yang sama, latency naik terutama saat peak hour
- **Tidak ada redundansi**: kalau Redis down, semua service yang bergantung padanya ikut terdampak dan request langsung tembus ke database
- **Read scalability mentok**: menambah resource di satu server ada batasnya, dan mayoritas operasi adalah read yang seharusnya bisa didistribusikan
- **Tidak bisa pisah workload**: operasi read-heavy seperti session check dan cache lookup bercampur dengan write operation
- **Performa drop saat trafik tinggi**: Redis yang overloaded bikin response time naik, dan service yang bergantung padanya ikut lambat karena menunggu reply dari Redis

## Pendekatan Solusi

Ada beberapa opsi untuk mendistribusikan beban Redis:

| Pendekatan              | Kelebihan                                           | Kekurangan                                                       |
| ----------------------- | --------------------------------------------------- | ---------------------------------------------------------------- |
| **Master-Slave**        | Setup simpel, slave bisa untuk read dan backup      | Failover manual, slave read-only                                 |
| **Redis Sentinel**      | Automatic failover, monitoring built-in             | Butuh minimal 3 sentinel instance, setup lebih kompleks          |
| **Redis Cluster**       | Sharding otomatis, horizontal scaling untuk write   | Butuh minimal 6 node, aplikasi harus support cluster protocol    |
| **Twemproxy / Envoy**   | Proxy layer untuk distribusi koneksi                | Nambah layer infrastruktur, single point of failure baru         |

Saya pilih **Master-Slave Replication** karena paling straightforward untuk kebutuhan saat ini. Cukup untuk memisahkan beban read dan write, dan slave bisa sekaligus jadi node cadangan. Di sisi aplikasi, tinggal arahkan read ke slave dan write ke master.

Arsitektur yang dibangun:

| Komponen   | IP Address     | Peran                                        |
| ---------- | -------------- | -------------------------------------------- |
| **Master** | `192.168.1.10` | Handle write, kirim data replication ke slave |
| **Slave**  | `192.168.1.11` | Terima replication dari master, handle read   |

## Implementasi Teknis

### Instalasi Redis

Install Redis di kedua server (master dan slave):

```
$ sudo apt update
$ sudo apt install redis-server -y
```

Pastikan Redis sudah jalan:

```
$ sudo systemctl status redis-server
```

Output yang diharapkan menunjukkan service active dan running. Redis secara default listen di port `6379`.

Cek versi Redis yang terinstall:

```
$ redis-server --version
```

```
Redis server v=6.0.16 sha=00000000:0 malloc=jemalloc-5.2.1 bits=64 build=...
```

Ubuntu 22.04 menyediakan Redis 6.x dari repository default yang sudah mendukung fitur replication dengan baik.

### Konfigurasi Redis Master

Buka file konfigurasi Redis di server master:

```
$ sudo nano /etc/redis/redis.conf
```

Ubah parameter berikut:

```conf
bind 0.0.0.0
protected-mode no
port 6379
requirepass master123
masterauth master123
```

Penjelasan tiap parameter:

| Parameter        | Fungsi                                                                       |
| ---------------- | ---------------------------------------------------------------------------- |
| `bind`           | Izinkan koneksi dari semua IP supaya slave bisa connect                      |
| `protected-mode` | Dimatikan karena sudah pakai password authentication                         |
| `port`           | Port default Redis, bisa diganti kalau mau                                   |
| `requirepass`    | Password untuk autentikasi client yang connect ke instance ini               |
| `masterauth`     | Password untuk autentikasi antar node replication, diisi sama dengan master  |

> **Penting:** Jangan matikan `protected-mode` tanpa mengatur `requirepass`. Kombinasi keduanya tanpa password berarti Redis terbuka untuk siapa saja yang bisa reach IP-nya.

Restart Redis:

```
$ sudo systemctl restart redis-server
```

Verifikasi Redis master sudah menerima koneksi dengan password:

```
$ redis-cli -a master123 ping
```

```
PONG
```

Kalau output-nya `PONG`, berarti master sudah siap.

### Konfigurasi Redis Slave

Buka file konfigurasi Redis di server slave:

```
$ sudo nano /etc/redis/redis.conf
```

Ubah parameter berikut:

```conf
bind 0.0.0.0
protected-mode no
port 6379
requirepass slave123
masterauth master123
replicaof 192.168.1.10 6379
replica-read-only yes
```

Parameter tambahan di slave:

| Parameter           | Fungsi                                                                    |
| ------------------- | ------------------------------------------------------------------------- |
| `replicaof`         | Menentukan IP dan port master yang akan di-replicate                      |
| `replica-read-only` | Slave hanya bisa menerima read, write ditolak kecuali dari replication    |
| `masterauth`        | Password untuk autentikasi ke master saat proses replication              |

Restart Redis:

```
$ sudo systemctl restart redis-server
```

Setelah restart, slave akan otomatis mencoba connect ke master dan memulai proses sinkronisasi data.

### Verifikasi Replication

Cek status replication di server master:

```
$ redis-cli -a master123 info replication
```

```
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.1.11,port=6379,state=online,offset=1234,lag=0
master_failover_state:no-failover
master_replid:abc123def456...
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1234
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1234
```

Yang perlu diperhatikan:

- **role:master**: konfirmasi bahwa node ini berjalan sebagai master
- **connected_slaves:1**: ada satu slave yang terhubung
- **state:online**: slave dalam kondisi aktif dan sinkron
- **lag:0**: tidak ada delay replication

Cek juga dari sisi slave:

```
$ redis-cli -a slave123 info replication
```

```
# Replication
role:slave
master_host:192.168.1.10
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
```

Nilai `master_link_status:up` menandakan koneksi ke master dalam keadaan baik. Kalau nilainya `down`, cek koneksi network dan password authentication.

### Testing Replication

Buat data di master, lalu verifikasi apakah muncul di slave.

Di server master:

```
$ redis-cli -a master123
```

```
127.0.0.1:6379> SET test:replication "it works"
OK
127.0.0.1:6379> SET test:counter 42
OK
127.0.0.1:6379> HSET test:user name "john" role "admin"
(integer) 2
```

Di server slave:

```
$ redis-cli -a slave123
```

```
127.0.0.1:6379> GET test:replication
"it works"
127.0.0.1:6379> GET test:counter
"42"
127.0.0.1:6379> HGETALL test:user
1) "name"
2) "john"
3) "role"
4) "admin"
```

Data muncul di slave, replication sudah jalan. Coba juga pastikan slave memang read-only:

```
127.0.0.1:6379> SET test:write "should fail"
(error) READONLY You can't write against a read only replica.
```

Error ini menandakan `replica-read-only` sudah aktif dan slave tidak bisa menerima write dari client.

### Pengujian Failover Sederhana

Simulasikan master down untuk melihat perilaku slave:

```
$ sudo systemctl stop redis-server
```

Di slave, cek status replication:

```
$ redis-cli -a slave123 info replication
```

```
# Replication
role:slave
master_host:192.168.1.10
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
master_link_down_since_seconds:15
```

Slave mendeteksi master down (`master_link_status:down`), tapi data yang sudah ter-replicate masih bisa dibaca:

```
127.0.0.1:6379> GET test:replication
"it works"
```

Untuk melakukan manual failover, promote slave menjadi master:

```
127.0.0.1:6379> REPLICAOF NO ONE
OK
```

Sekarang slave berubah jadi standalone master yang bisa menerima write:

```
127.0.0.1:6379> SET failover:test "slave promoted"
OK
```

> **Warning:** Failover manual seperti ini berarti ada window time di mana write baru di master lama yang belum ter-replicate akan hilang. Untuk production yang butuh automatic failover, gunakan **Redis Sentinel**.

Setelah master lama kembali online, kembalikan topologi:

Di master lama (setelah di-start ulang):

```
$ sudo systemctl start redis-server
```

Di slave (kembalikan jadi replica):

```
$ redis-cli -a slave123 REPLICAOF 192.168.1.10 6379
```

### Optimasi Redis untuk High Traffic

Untuk menangani trafik tinggi, ada beberapa parameter yang perlu di-tune di kedua server. Buka `/etc/redis/redis.conf`:

```conf
# Memory management
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence tuning
save ""
appendonly no

# Connection
maxclients 10000
tcp-backlog 511
tcp-keepalive 300

# Replication tuning
repl-backlog-size 64mb
repl-diskless-sync yes
repl-diskless-sync-delay 5

# Performance
hz 100
dynamic-hz yes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

Penjelasan parameter optimasi:

| Parameter                  | Fungsi                                                                      |
| -------------------------- | --------------------------------------------------------------------------- |
| `maxmemory`                | Batas penggunaan memory, sesuaikan dengan RAM server                        |
| `maxmemory-policy`         | `allkeys-lru` menghapus key yang paling lama tidak diakses saat memory penuh|
| `save ""`                  | Matikan RDB snapshot untuk mengurangi disk I/O                              |
| `appendonly no`            | Matikan AOF persistence kalau Redis dipakai murni sebagai cache             |
| `repl-backlog-size`        | Buffer untuk menampung data replication, naikkan kalau sering partial resync|
| `repl-diskless-sync`       | Full sync langsung via socket tanpa tulis ke disk dulu, lebih cepat         |
| `repl-diskless-sync-delay` | Delay sebelum diskless sync dimulai, beri waktu slave lain untuk ikut sync  |
| `hz`                       | Frekuensi internal timer Redis, naikkan untuk respon lebih cepat            |
| `lazyfree-lazy-eviction`   | Hapus key secara asynchronous supaya tidak block main thread                |

> **Penting:** Mematikan persistence (`save ""` dan `appendonly no`) berarti data hilang kalau Redis restart. Ini cocok kalau Redis dipakai murni sebagai cache, bukan sebagai primary data store. Kalau butuh durabilitas, tetap aktifkan minimal AOF.

Selain konfigurasi Redis, optimasi di level OS juga penting:

```
$ sudo sysctl -w vm.overcommit_memory=1
$ sudo sysctl -w net.core.somaxconn=65535
$ echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

Supaya setting ini persisten setelah reboot, tambahkan di `/etc/sysctl.conf`:

```conf
vm.overcommit_memory=1
net.core.somaxconn=65535
```

Dan untuk transparent hugepage, buat systemd service atau tambahkan di `/etc/rc.local`.

### Monitoring Replication

Monitoring replication Redis perlu dijalankan secara berkala. Buat script sederhana yang bisa dijadwalkan lewat cron:

```bash
#!/bin/bash
TELEGRAM_BOT_TOKEN="your_bot_token"
TELEGRAM_CHAT_ID="your_chat_id"
REDIS_PASSWORD="slave123"

send_telegram() {
    local message="$1"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d chat_id="${TELEGRAM_CHAT_ID}" \
        -d text="${message}" \
        -d parse_mode="Markdown" > /dev/null
}

REPL_INFO=$(redis-cli -a "$REDIS_PASSWORD" --no-auth-warning info replication 2>/dev/null)
ROLE=$(echo "$REPL_INFO" | grep "role:" | cut -d: -f2 | tr -d '\r')

if [ "$ROLE" = "slave" ]; then
    LINK_STATUS=$(echo "$REPL_INFO" | grep "master_link_status:" | cut -d: -f2 | tr -d '\r')
    SYNC_PROGRESS=$(echo "$REPL_INFO" | grep "master_sync_in_progress:" | cut -d: -f2 | tr -d '\r')

    if [ "$LINK_STATUS" != "up" ]; then
        send_telegram "*Redis Replication Alert*
Link to master is DOWN
Host: $(hostname)"
    fi

    if [ "$SYNC_PROGRESS" = "1" ]; then
        send_telegram "*Redis Replication Warning*
Full sync in progress - expect CPU/IO spike
Host: $(hostname)"
    fi
fi

if [ "$ROLE" = "master" ]; then
    CONNECTED=$(echo "$REPL_INFO" | grep "connected_slaves:" | cut -d: -f2 | tr -d '\r')
    if [ "$CONNECTED" -eq 0 ] 2>/dev/null; then
        send_telegram "*Redis Replication Alert*
No slaves connected to master
Host: $(hostname)"
    fi

    LAG=$(echo "$REPL_INFO" | grep "slave0:" | grep -oP 'lag=\K[0-9]+')
    if [ -n "$LAG" ] && [ "$LAG" -gt 10 ] 2>/dev/null; then
        send_telegram "*Redis Replication Lag*
Slave lag: ${LAG}s
Host: $(hostname)"
    fi
fi
```

Script ini mengecek status link replication, proses full sync, jumlah slave yang terhubung, dan replication lag. Simpan di `/usr/local/bin/check_redis_replication.sh`, beri permission execute, dan jadwalkan di cron tiap 5 menit:

```
$ sudo chmod +x /usr/local/bin/check_redis_replication.sh
$ sudo crontab -e
```

Tambahkan baris:

```
*/5 * * * * /usr/local/bin/check_redis_replication.sh
```

## Tantangan yang Dihadapi

Tantangan pertama yang saya temui adalah **replication bersifat asynchronous**. Artinya setelah write berhasil di master, tidak ada jaminan data langsung tersedia di slave. Ada jeda waktu, meskipun biasanya dalam hitungan milidetik, tapi di saat trafik sangat tinggi, jeda ini bisa membesar. Ini jadi masalah kalau aplikasi langsung baca dari slave setelah write ke master. Misal user update profile, lalu redirect ke halaman profile yang baca dari slave, data lama yang muncul. Solusinya, untuk operasi yang butuh konsistensi ketat, arahkan read ke master juga. Pisahkan mana read yang boleh stale dan mana yang harus fresh.

**Full sync** juga jadi masalah tersendiri. Saat slave pertama kali connect atau setelah putus cukup lama, Redis melakukan full synchronization. Master membuat RDB snapshot dan mengirimnya ke slave. Proses ini menyebabkan spike CPU dan disk I/O di master yang cukup signifikan, dan selama proses berlangsung performa master bisa turun. Dengan mengaktifkan `repl-diskless-sync`, overhead disk I/O berkurang karena data dikirim langsung via socket. Tapi tetap saja, full sync di jam sibuk sebaiknya dihindari.

Satu hal lagi yang tidak langsung obvious, **scaling write tetap terbatas pada master**. Mau tambah slave berapapun, semua write tetap masuk ke satu node. Kalau bottleneck-nya ada di write, master-slave replication tidak akan menyelesaikan masalah. Untuk kasus itu, perlu pertimbangkan **Redis Cluster** yang mendukung sharding. Dan soal failover, tanpa **Redis Sentinel**, promosi slave ke master harus dilakukan manual, dan ada window time di mana service tidak bisa menerima write sama sekali. Untuk production yang critical, Sentinel atau solusi orchestration lain hampir wajib.

## Insight dan Pembelajaran

- **Asynchronous replication punya trade-off**: performa tinggi tapi konsistensi tidak dijamin. Pahami mana read yang boleh stale dan mana yang harus konsisten, lalu route query-nya sesuai
- **Full sync itu mahal**: hindari kondisi yang memicu full sync di jam sibuk. Perbesar `repl-backlog-size` supaya partial resync lebih sering berhasil dan tidak perlu fallback ke full sync
- **Persistence dan performa itu trade-off**: mematikan RDB dan AOF bikin Redis lebih cepat tapi data hilang saat restart. Putuskan berdasarkan apakah Redis dipakai sebagai cache atau data store
- **Monitoring bukan opsional**: replication bisa putus tanpa warning. Tanpa monitoring aktif, slave bisa serving data stale berjam-jam tanpa ada yang sadar
- **Master-slave bukan solusi untuk semua masalah**: kalau bottleneck di write, tambah slave tidak membantu. Kenali dulu profil workload sebelum memilih arsitektur
- **Tuning OS sama pentingnya dengan tuning Redis**: `vm.overcommit_memory`, `somaxconn`, dan transparent hugepage bisa jadi pembeda signifikan di production

## Penutup

Redis replication master-slave cukup efektif untuk mendistribusikan beban read dan meningkatkan availability, terutama di workload yang read-heavy. Konfigurasinya lebih simpel dibanding MySQL replication karena tidak perlu dump data manual. Redis menangani sinkronisasi awal secara otomatis. Yang perlu diperhatikan adalah sifat asynchronous-nya, batasan write scaling di master, dan pentingnya monitoring yang jalan terus supaya masalah replication bisa terdeteksi sebelum berdampak ke user.

## Referensi

- [Redis Master Slave](https://www.educba.com/redis-master-slave/), diakses pada2026-05-14
- [Redis Replication Documentation](https://redis.io/docs/management/replication/), diakses pada2026-05-14
