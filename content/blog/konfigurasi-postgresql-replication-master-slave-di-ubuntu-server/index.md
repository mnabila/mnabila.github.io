+++
draft = false
date = '2026-06-22'
title = 'Konfigurasi PostgreSQL Replication Master-Slave di Ubuntu Server'
type = 'blog'
description = 'Konfigurasi PostgreSQL 16 replication master-slave menggunakan Docker Compose di 2 VM Ubuntu Server 22.04 untuk meningkatkan availability dan mendistribusikan beban read query'
image = ''
tags = ['postgresql', 'replication', 'docker', 'docker-compose', 'ubuntu', 'server']
+++

## Latar Belakang

Aplikasi yang saya kelola menggunakan **PostgreSQL** sebagai database utama. Seiring pertumbuhan user, query ke database makin banyak dan mayoritas adalah operasi read: laporan, dashboard, listing data, pencarian. Semua query ini masuk ke satu instance PostgreSQL yang sama, bersaing resource dengan write operation seperti insert dan update transaksi.

Selama trafik masih rendah, satu instance cukup. Tapi begitu trafik naik, response time mulai merangkak dan beberapa query berat mulai mengantri. Saya perlu cara untuk mendistribusikan beban read tanpa harus overhaul arsitektur aplikasi secara menyeluruh.

## Permasalahan

Mengandalkan satu instance PostgreSQL untuk semua operasi mulai terasa limitasinya:

- **Semua query numpuk di satu node**: read dan write rebutan resource di instance yang sama, I/O disk dan CPU bersaing terutama saat ada query reporting yang berat
- **Tidak ada redundansi**: kalau server database down, seluruh aplikasi ikut mati dan tidak ada node cadangan yang bisa langsung ambil alih
- **Read scalability mentok**: nambah resource di satu server ada batasnya, padahal mayoritas operasi adalah read yang seharusnya bisa didistribusikan
- **Backup ganggu production**: menjalankan `pg_dump` di server yang sama berarti ada tambahan beban I/O yang terasa di aplikasi
- **Tidak bisa pisah workload**: query reporting yang berat berjalan bersamaan dengan query transaksional, bikin keduanya saling menghambat

## Pendekatan Solusi

Ada beberapa cara untuk mendistribusikan beban database PostgreSQL:

| Pendekatan                   | Kelebihan                                                                        | Kekurangan                                |
| ---------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------- |
| **Streaming Replication**    | Built-in di PostgreSQL, setup relatif simpel, replica bisa untuk read dan backup | Replica read-only, failover manual        |
| **Logical Replication**      | Bisa replicate per tabel, subscriber bisa read-write                             | Tidak replicate DDL, setup lebih kompleks |
| **PgBouncer + Read Replica** | Connection pooling + distribusi read otomatis                                    | Nambah layer infrastruktur                |

Saya pilih **Streaming Replication** karena paling straightforward untuk kebutuhan saat ini. Mekanisme ini menggunakan **WAL (Write-Ahead Log)**, setiap perubahan data yang terjadi di primary dicatat ke WAL, lalu dikirim secara streaming ke replica yang kemudian me-replay perubahan tersebut. Hasilnya, replica punya salinan data yang nyaris identik dengan primary secara real-time.

Setup yang saya bangun menggunakan **Docker Compose** di 2 VM terpisah. Primary node handle semua write, replica node handle read. Masing-masing jalan di VM sendiri supaya kehilangan satu server tidak membuat database sepenuhnya mati. Selain performa lebih baik karena beban terdistribusi, replica juga berfungsi sebagai standby kalau primary bermasalah.

Arsitektur yang dibangun, 1 instance per VM, masing-masing VM terpisah sehingga beban read dan write terdistribusi secara fisik:

| Komponen    | IP Address     | Port   | Peran                                |
| ----------- | -------------- | ------ | ------------------------------------ |
| **Primary** | `10.10.10.11` | `5432` | Handle write, kirim WAL ke replica   |
| **Replica** | `10.10.10.12` | `5432` | Terima WAL dari primary, handle read |

## Implementasi Teknis

### Prasyarat

Sebelum mulai, pastikan kedua VM sudah memenuhi kondisi berikut:

- Ubuntu Server 22.04 LTS sudah terinstall
- Docker Engine dan Docker Compose v2 sudah terinstall di kedua VM
- Kedua VM bisa saling berkomunikasi melalui jaringan internal
- Port `5432` tidak digunakan service lain

Pembagian IP:

| VM   | IP Address     | Peran   |
| ---- | -------------- | ------- |
| VM 1 | `10.10.10.11` | Primary |
| VM 2 | `10.10.10.12` | Replica |

### Konfigurasi Primary (VM 1)

Buat direktori kerja di server primary:

```
$ mkdir -p /opt/pg-primary && cd /opt/pg-primary
```

Struktur file yang akan dibuat di VM 1:

```
pg-primary/
├── docker-compose.yml
├── pg_hba.conf
└── init-replication.sh
```

Pertama, buat file `pg_hba.conf` untuk mengizinkan koneksi replication dari replica:

```conf
# pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust
host    all             all             0.0.0.0/0               md5
host    replication     replicator      10.10.10.12/32         md5
```

Baris terakhir adalah yang paling penting yakni mengizinkan user `replicator` untuk melakukan koneksi replication khusus dari IP replica (`10.10.10.12`). Method `md5` memastikan koneksi tetap memerlukan password.

> **Tip:** Kalau berencana menambah replica di kemudian hari, bisa ganti `10.10.10.12/32` menjadi `10.10.10.0/24` supaya semua IP di subnet tersebut bisa melakukan replication. Tapi untuk setup awal, lebih aman membatasi ke IP spesifik.

Kemudian buat script inisialisasi yang membuat user replication dan mengatur parameter WAL:

```bash
#!/bin/bash
# init-replication.sh
set -e

# Buat user replication
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator123';
EOSQL

# Konfigurasi WAL untuk streaming replication
cat >> "$PGDATA/postgresql.conf" <<EOF

# Replication
wal_level = replica
max_wal_senders = 5
wal_keep_size = 256MB
hot_standby = on
listen_addresses = '*'
EOF

# Ganti pg_hba.conf dengan konfigurasi custom
cp /etc/postgresql/pg_hba.conf "$PGDATA/pg_hba.conf"

# Reload konfigurasi
pg_ctl reload -D "$PGDATA"
```

Penjelasan parameter WAL:

| Parameter          | Fungsi                                                                                                         |
| ------------------ | -------------------------------------------------------------------------------------------------------------- |
| `wal_level`        | Set ke `replica` untuk mengaktifkan WAL yang cukup untuk streaming replication                                 |
| `max_wal_senders`  | Jumlah maksimal koneksi replication yang bisa berjalan bersamaan                                               |
| `wal_keep_size`    | Jumlah WAL yang disimpan di primary, supaya replica yang sempat tertinggal bisa catch up tanpa perlu full sync |
| `hot_standby`      | Izinkan replica menerima read query saat dalam mode standby                                                    |
| `listen_addresses` | Set ke `*` supaya PostgreSQL menerima koneksi dari semua interface, termasuk dari replica di VM lain           |

Beri permission execute pada script:

```
$ chmod +x init-replication.sh
```

Buat file `docker-compose.yml` untuk primary:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: pg-primary
    restart: unless-stopped
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pg_primary_data:/var/lib/postgresql/data
      - ./init-replication.sh:/docker-entrypoint-initdb.d/init-replication.sh
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pg_primary_data:
```

Penjelasan konfigurasi penting:

| Source | Destination | Fungsi |
|--------|-------------|--------|
| `./init-replication.sh` | `/docker-entrypoint-initdb.d/init-replication.sh` | Script otomatis dijalankan saat primary pertama kali inisialisasi |
| `./pg_hba.conf` | `/etc/postgresql/pg_hba.conf` | File HBA custom yang akan di-copy ke `$PGDATA` oleh init script |
| `pg_primary_data` | `/var/lib/postgresql/data` | Named volume untuk persist data PostgreSQL supaya tidak hilang saat container restart |

> **Penting:** Jangan lupa `listen_addresses = '*'` di konfigurasi PostgreSQL. Tanpa ini, PostgreSQL hanya listen di `localhost` dan replica di VM lain tidak bisa connect.

Jalankan primary:

```
$ docker compose up -d
```

Cek status container:

```
$ docker compose ps
```

```
NAME          IMAGE                COMMAND                  SERVICE    STATUS                    PORTS
pg-primary    postgres:16-alpine   "docker-entrypoint.s…"   postgres   Up (healthy)              0.0.0.0:5432->5432/tcp
```

Verifikasi primary sudah menerima koneksi dengan password:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT 1;"
```

```
 ?column?
----------
        1
```

Pastikan juga user replication sudah terbuat:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT usename, userepl FROM pg_user WHERE usename = 'replicator';"
```

```
  usename   | userepl
------------+---------
 replicator | t
```

Kolom `userepl: t` mengonfirmasi user `replicator` sudah punya privilege replication. Primary siap menerima koneksi dari replica.

### Konfigurasi Replica (VM 2)

Buat direktori kerja di server replica:

```
$ mkdir -p /opt/pg-replica && cd /opt/pg-replica
```

Struktur file yang akan dibuat di VM 2:

```
pg-replica/
├── docker-compose.yml
└── start-replica.sh
```

Buat script startup untuk replica yang akan melakukan base backup dari primary lalu memulai PostgreSQL dalam mode standby:

```bash
#!/bin/bash
# start-replica.sh
set -e

PRIMARY_HOST=10.10.10.11
PRIMARY_PORT=5432

# Tunggu primary siap menerima koneksi
until PGPASSWORD=replicator123 pg_isready -h "$PRIMARY_HOST" -p "$PRIMARY_PORT" -U replicator; do
    echo "Menunggu primary siap..."
    sleep 2
done

# Ambil base backup dari primary kalau data directory masih kosong
if [ ! -f "$PGDATA/PG_VERSION" ]; then
    echo "Mengambil base backup dari primary..."
    rm -rf "$PGDATA"/*

    PGPASSWORD=replicator123 pg_basebackup \
        -h "$PRIMARY_HOST" \
        -p "$PRIMARY_PORT" \
        -U replicator \
        -D "$PGDATA" \
        -Fp -Xs -P -R

    echo "Base backup selesai."
fi

# Jalankan PostgreSQL
exec docker-entrypoint.sh postgres
```

Penjelasan flag `pg_basebackup`:

| Flag  | Fungsi                                                                                        |
| ----- | --------------------------------------------------------------------------------------------- |
| `-h`  | Host primary yang akan di-backup, diisi IP VM 1                                               |
| `-U`  | User replication yang sudah dibuat di primary                                                 |
| `-D`  | Direktori tujuan backup, diisi `$PGDATA` supaya langsung jadi data directory PostgreSQL       |
| `-Fp` | Format plain: backup langsung jadi file PostgreSQL, bukan archive                           |
| `-Xs` | WAL method stream: WAL ikut di-stream selama backup supaya hasilnya konsisten               |
| `-P`  | Tampilkan progress backup                                                                     |
| `-R`  | Otomatis buat file `standby.signal` dan `postgresql.auto.conf` dengan konfigurasi replication |

Flag `-R` adalah yang paling penting karena otomatis mengonfigurasi replica untuk connect ke primary tanpa perlu setup manual. PostgreSQL akan membuat file `standby.signal` yang menandakan instance ini berjalan sebagai standby, dan menulis `primary_conninfo` di `postgresql.auto.conf` berisi koneksi ke primary.

Beri permission execute:

```
$ chmod +x start-replica.sh
```

Buat file `docker-compose.yml` untuk replica:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: pg-replica
    restart: unless-stopped
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pg_replica_data:/var/lib/postgresql/data
      - ./start-replica.sh:/start-replica.sh
    entrypoint: ["/bin/bash", "/start-replica.sh"]

volumes:
  pg_replica_data:
```

Replica menggunakan custom `entrypoint` yang menggantikan entrypoint default PostgreSQL. Script `start-replica.sh` menangani base backup dari primary terlebih dahulu, baru kemudian menjalankan `docker-entrypoint.sh postgres` untuk memulai PostgreSQL dalam mode standby.

> **Penting:** Replica tidak menggunakan `docker-entrypoint-initdb.d/` karena data directory-nya diisi oleh `pg_basebackup`, bukan oleh proses inisialisasi PostgreSQL biasa. Kalau pakai entrypoint default, PostgreSQL akan mencoba inisialisasi database baru dan menimpa hasil base backup.

Jalankan replica:

```
$ docker compose up -d
```

Cek log untuk memastikan base backup berhasil dan replication sudah jalan:

```
$ docker compose logs -f postgres
```

Output yang diharapkan kurang lebih:

```
pg-replica  | Menunggu primary siap...
pg-replica  | Mengambil base backup dari primary...
pg-replica  | 30318/30318 kB (100%), 1/1 tablespace
pg-replica  | Base backup selesai.
pg-replica  | LOG:  entering standby mode
pg-replica  | LOG:  redo starts at 0/3000028
pg-replica  | LOG:  started streaming WAL from primary at 0/4000000 on timeline 1
```

Baris `started streaming WAL from primary` mengonfirmasi replica sudah terhubung ke primary dan menerima WAL stream.

### Verifikasi Replication

Cek status replication dari sisi primary di **VM 1**:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT pid, usename, client_addr, state, sync_state FROM pg_stat_replication;"
```

```
 pid | usename    | client_addr   | state     | sync_state
-----+------------+---------------+-----------+------------
  78 | replicator | 10.10.10.12  | streaming | async
```

Yang perlu diperhatikan:

- **usename: replicator**: replica connect menggunakan user replication yang sudah dibuat
- **client_addr: 10.10.10.12**: koneksi berasal dari IP replica di VM 2
- **state: streaming**: WAL sedang di-stream secara aktif ke replica
- **sync_state: async**: replication berjalan secara asynchronous

Cek juga dari sisi replica di **VM 2**:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT pid, status, sender_host, sender_port FROM pg_stat_wal_receiver;"
```

```
 pid | status    | sender_host   | sender_port
-----+-----------+---------------+-------------
  34 | streaming | 10.10.10.11  |        5432
```

Status `streaming` dan `sender_host: 10.10.10.11` menandakan replica aktif menerima WAL dari primary di VM 1.

### Testing Replication

Buat data di primary, lalu verifikasi apakah muncul di replica.

Di **VM 1** (primary):

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
CREATE TABLE test_replication (id SERIAL PRIMARY KEY, message TEXT, created_at TIMESTAMP DEFAULT NOW());
INSERT INTO test_replication (message) VALUES ('replication works');
INSERT INTO test_replication (message) VALUES ('data from primary');
```

Di **VM 2** (replica):

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
SELECT * FROM test_replication;
```

```
 id |      message      |         created_at
----+-------------------+----------------------------
  1 | replication works | 2026-06-22 10:30:15.123456
  2 | data from primary | 2026-06-22 10:30:18.654321
```

Data muncul di replica. Pastikan juga replica memang read-only:

```sql
INSERT INTO test_replication (message) VALUES ('should fail');
```

```
ERROR:  cannot execute INSERT in a read-only transaction
```

Error ini mengonfirmasi replica berjalan dalam mode hot standby sehingga hanya bisa menerima action read  dan untuk action write  akan ditolak.

### Pengujian Failover Sederhana

Simulasikan primary down untuk melihat perilaku replica.

Di **VM 1**, matikan primary:

```
$ docker compose down
```

Di **VM 2**, cek status replication:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT pid, status, sender_host FROM pg_stat_wal_receiver;"
```

```
 pid | status | sender_host
-----+--------+-------------
(0 rows)
```

Replica mendeteksi primary sudah tidak tersedia sehingga `pg_stat_wal_receiver` kosong. Tapi data yang sudah ter-replicate masih bisa dibaca:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT * FROM test_replication;"
```

```
 id |      message      |         created_at
----+-------------------+----------------------------
  1 | replication works | 2026-06-22 10:30:15.123456
  2 | data from primary | 2026-06-22 10:30:18.654321
```

Untuk melakukan manual failover, promote replica menjadi standalone primary. Masuk ke container replica:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
SELECT pg_promote();
```

```
 pg_promote
------------
 t
```

Sekarang replica berubah jadi standalone primary yang bisa menerima write:

```sql
INSERT INTO test_replication (message) VALUES ('replica promoted');
```

```
INSERT 0 1
```

> **Warning:** Failover manual seperti ini berarti ada window time di mana write baru di primary lama yang belum ter-replicate akan hilang. Untuk production yang butuh automatic failover, gunakan **Patroni** atau **repmgr**.

Setelah primary lama kembali online, topologi perlu dibangun ulang. Di **VM 1**, hapus volume lama dan konfigurasi ulang sebagai replica baru, atau kembalikan sebagai primary dan rebuild replica di VM 2 dari awal.

### Monitoring Replication

Monitoring replication perlu dijalankan secara berkala. Buat script sederhana yang bisa dijadwalkan lewat cron di **VM 1** (primary):

```bash
#!/bin/bash
# check_pg_replication.sh
TELEGRAM_BOT_TOKEN="your_bot_token"
TELEGRAM_CHAT_ID="your_chat_id"

send_telegram() {
    local message="$1"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d chat_id="${TELEGRAM_CHAT_ID}" \
        -d text="${message}" \
        -d parse_mode="Markdown" > /dev/null
}

# Cek jumlah replica yang terhubung
REPLICA_COUNT=$(docker compose exec -T postgres psql -U devuser -d appdb -t -c \
    "SELECT COUNT(*) FROM pg_stat_replication;" 2>/dev/null | tr -d ' ')

if [ "$REPLICA_COUNT" -eq 0 ] 2>/dev/null; then
    send_telegram "*PostgreSQL Replication Alert*
No replica connected to primary
Host: $(hostname) (10.10.10.11)"
fi

# Cek replication lag dalam byte
LAG_BYTES=$(docker compose exec -T postgres psql -U devuser -d appdb -t -c \
    "SELECT COALESCE(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn), 0) FROM pg_stat_replication LIMIT 1;" 2>/dev/null | tr -d ' ')

if [ -n "$LAG_BYTES" ] && [ "$LAG_BYTES" -gt 104857600 ] 2>/dev/null; then
    LAG_MB=$((LAG_BYTES / 1048576))
    send_telegram "*PostgreSQL Replication Lag*
Lag: ${LAG_MB}MB
Host: $(hostname) (10.10.10.11)"
fi
```

Script ini mengecek jumlah replica yang terhubung dan replication lag dalam byte. Threshold alert diset 100MB, kalau lag melebihi ini kemungkinan ada masalah network atau replica sedang overloaded. Simpan di `/usr/local/bin/check_pg_replication.sh`, beri permission execute, dan jadwalkan di cron tiap 5 menit:

```
$ sudo chmod +x /usr/local/bin/check_pg_replication.sh
$ sudo crontab -e
```

Tambahkan baris:

```
*/5 * * * * cd /opt/pg-primary && /usr/local/bin/check_pg_replication.sh
```

## Tantangan yang Dihadapi

Tantangan pertama yang cukup membuat pusing adalah **konektivitas antar VM**. Berbeda dengan setup single host di mana container bisa berkomunikasi lewat Docker network internal, setup 2 VM membutuhkan koneksi melalui jaringan fisik. Container PostgreSQL di dalam Docker menggunakan port mapping ke host, jadi replica di VM 2 connect ke IP VM 1 (`10.10.10.11`) lewat port `5432` yang di-expose. Kalau firewall belum buka port ini, `pg_basebackup` gagal tanpa error message yang jelas, hanya timeout. Pastikan `ufw allow 5432/tcp` sudah dijalankan di kedua VM sebelum memulai.

**Replication bersifat asynchronous** secara default. Artinya setelah write berhasil di primary, tidak ada jaminan data langsung tersedia di replica. Ada jeda dalam hitungan milidetik, tapi saat trafik tinggi atau ada query berat di primary jeda ini bisa membesar. Ditambah lagi latency network antar VM yang bisa menambah delay. Ini jadi masalah kalau aplikasi langsung baca dari replica setelah write ke primary. Misalnya user update profil, lalu redirect ke halaman profil yang baca dari replica, data lama yang muncul. Untuk operasi yang butuh konsistensi ketat, arahkan read ke primary juga.

Satu hal lagi yang tidak langsung obvious: **volume Docker menyimpan state**. Kalau sudah pernah menjalankan setup ini dan ingin mengulang dari awal, data di volume lama bisa menyebabkan konflik. Init script di primary tidak jalan lagi karena volume sudah terisi, sementara replica mencoba connect dengan konfigurasi yang mungkin sudah berubah. Solusinya adalah `docker compose down -v` di kedua VM untuk menghapus volume sebelum memulai ulang. Urutan start juga penting: selalu nyalakan primary dulu sampai healthy, baru nyalakan replica. Kalau replica start duluan, `pg_basebackup` akan terus retry sampai primary tersedia.

## Insight dan Pembelajaran

- **WAL adalah jantungnya replication PostgreSQL**: semua perubahan data dicatat ke WAL sebelum di-commit. Replica me-replay WAL ini untuk menjaga sinkronisasi, kalau WAL tertinggal atau terhapus sebelum replica sempat baca, replication putus
- **`pg_basebackup` dengan flag `-R` menghemat banyak konfigurasi manual**: tanpa `-R`, harus buat `standby.signal` dan isi `primary_conninfo` di `postgresql.auto.conf` secara manual. Flag ini mengotomasi semua itu
- **`wal_keep_size` perlu disesuaikan dengan workload**: kalau terlalu kecil, WAL bisa di-recycle sebelum replica sempat baca dan replica perlu full sync ulang. Kalau terlalu besar, makan disk space di primary
- **Monitoring replication bukan opsional**: replication bisa putus tanpa warning. Tanpa monitoring aktif, replica bisa serving data stale berjam-jam tanpa ada yang sadar
- **Pisahkan Docker Compose per VM**: setiap VM menjalankan Docker Compose sendiri-sendiri, tidak ada orchestration lintas host. Urutan start harus diatur manual yakni primary dulu, baru replica
- **Pisahkan connection pool di aplikasi**: pakai pool terpisah untuk read (ke replica `10.10.10.12:5432`) dan write (ke primary `10.10.10.11:5432`), supaya bisa switch node tanpa deploy ulang

## Penutup

PostgreSQL streaming replication dengan Docker Compose di 2 VM terpisah cukup efektif untuk mendistribusikan beban read dan meningkatkan availability database. Primary di VM 1 fokus handle write, replica di VM 2 fokus handle read, dan kehilangan satu VM tidak membuat data hilang karena sudah ter-replicate. Yang perlu diperhatikan adalah sifat asynchronous-nya, konektivitas network antar VM, manajemen volume Docker yang menyimpan state, dan pentingnya monitoring yang jalan terus supaya masalah replication terdeteksi sebelum berdampak ke user.

## Referensi

- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/16/warm-standby.html#STREAMING-REPLICATION), diakses pada2026-06-22
- [PostgreSQL WAL Configuration](https://www.postgresql.org/docs/16/runtime-config-wal.html), diakses pada2026-06-22
- [pg_basebackup Documentation](https://www.postgresql.org/docs/16/app-pgbasebackup.html), diakses pada2026-06-22
- [PostgreSQL Docker Official Image](https://hub.docker.com/_/postgres), diakses pada2026-06-22
