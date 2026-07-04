+++
draft = false
date = '2026-06-27'
title = 'Migrasi Data PostgreSQL Tanpa Downtime dengan pgcopydb dan Logical Replication'
type = 'blog'
description = 'Memindahkan seluruh database PostgreSQL dari server lama ke server baru tanpa menghentikan layanan menggunakan pgcopydb untuk initial copy dan logical replication untuk sinkronisasi perubahan hingga cutover'
image = ''
tags = ['postgresql', 'pgcopydb', 'logical-replication', 'migration', 'docker', 'ubuntu']
+++

## Latar Belakang

Server database **PostgreSQL** yang sudah jalan selama beberapa tahun mulai menunjukkan tanda-tanda perlu diganti. Bisa karena hardware yang sudah tua, versi OS yang sudah end-of-life, atau kebutuhan upgrade ke spesifikasi yang lebih tinggi. Apapun alasannya, database harus pindah ke server baru.

Masalahnya, database ini melayani aplikasi production yang tidak bisa dihentikan begitu saja. Kalau pakai pendekatan `pg_dump` lalu `pg_restore`, ada window waktu di mana data yang ditulis setelah dump dimulai tidak ikut terbawa. Semakin besar database-nya, semakin lama window itu, dan semakin banyak data yang berpotensi hilang atau harus di-sync manual.

Saya butuh pendekatan yang memungkinkan initial copy berjalan tanpa mengganggu operasi aplikasi, lalu menangkap semua perubahan yang terjadi selama proses copy berlangsung, dan akhirnya melakukan cutover dengan downtime seminimal mungkin.

## Permasalahan

Beberapa tantangan yang harus dijawab sebelum migrasi dimulai:

- **Database berukuran besar**: `pg_dump` dan `pg_restore` bisa memakan waktu berjam-jam untuk database puluhan gigabyte, selama itu aplikasi tetap harus berjalan dan menerima write
- **Data harus tetap sinkron**: perubahan yang terjadi di server lama selama proses copy berlangsung harus ikut terbawa ke server baru, tidak boleh ada data yang hilang
- **Aplikasi tidak boleh berhenti**: layanan harus tetap menerima request baca dan tulis selama migrasi, downtime hanya boleh terjadi saat cutover final
- **Sequence value harus akurat**: sequence di PostgreSQL tidak ter-replicate oleh logical replication, kalau tidak ditangani manual bisa terjadi conflict primary key setelah cutover
- **Extension dan foreign key**: tidak semua extension kompatibel dengan logical replication, dan foreign key constraint bisa menghambat proses initial data copy

## Pendekatan Solusi

Ada beberapa strategi untuk migrasi database PostgreSQL tanpa downtime:

| Pendekatan | Kelebihan | Kekurangan |
| --- | --- | --- |
| **pg_dump + pg_restore** | Simpel, built-in, tidak butuh konfigurasi khusus | Downtime selama proses dump-restore, tidak ada sync perubahan |
| **Streaming Replication** | Real-time sync, built-in | Replica read-only, harus versi dan arsitektur sama persis |
| **pgcopydb + Logical Replication** | Initial copy cepat (paralel), sync perubahan real-time, target bisa read-write | Setup lebih kompleks, perlu konfigurasi WAL dan publication |
| **pglogical** | Fitur lebih lengkap dari logical replication bawaan | Extension tambahan, tidak selalu tersedia di managed database |

Saya pilih kombinasi **pgcopydb** untuk initial copy dan **Logical Replication** bawaan PostgreSQL untuk menangkap perubahan setelahnya. **pgcopydb** jauh lebih cepat dari `pg_dump`/`pg_restore` karena bisa melakukan copy secara paralel, membuat index secara paralel, dan langsung setup logical replication. Setelah initial copy selesai, logical replication mengambil alih untuk menjaga sinkronisasi data sampai cutover.

Arsitektur migrasi yang dibangun:

| Komponen | IP Address | Port | Peran |
| --- | --- | --- | --- |
| **Source (server lama)** | `10.10.10.11` | `5432` | Database production aktif, menjadi publisher |
| **Target (server baru)** | `10.10.10.12` | `5432` | Database tujuan migrasi, menjadi subscriber |

Kedua server menjalankan PostgreSQL 16 di Docker container pada Ubuntu Server 22.04.

## Implementasi Teknis

### Prasyarat

Sebelum mulai, pastikan kondisi berikut terpenuhi:

- Ubuntu Server 22.04 LTS terinstall di kedua server
- Docker Engine dan Docker Compose v2 terinstall di kedua server
- Kedua server bisa saling berkomunikasi melalui jaringan internal di port `5432`
- PostgreSQL di server lama sudah berjalan dan melayani production
- Akses superuser ke PostgreSQL di kedua server

### Konfigurasi Source (Server Lama)

Logical replication membutuhkan `wal_level` diset ke `logical`. Ini adalah perubahan yang memerlukan restart PostgreSQL, jadi perlu direncanakan.

Masuk ke container PostgreSQL di server lama dan cek konfigurasi WAL saat ini:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SHOW wal_level;"
```

```
 wal_level
-----------
 replica
```

Kalau hasilnya `replica` atau `minimal`, perlu diubah ke `logical`. Tambahkan konfigurasi berikut ke `postgresql.conf` atau melalui Docker Compose environment:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: pg-source
    restart: unless-stopped
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pg_source_data:/var/lib/postgresql/data
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_replication_slots=5"
      - "-c"
      - "max_wal_senders=5"

volumes:
  pg_source_data:
```

Restart container source untuk menerapkan perubahan:

```
$ docker compose up -d
```

Verifikasi `wal_level` sudah berubah:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SHOW wal_level;"
```

```
 wal_level
-----------
 logical
```

> **Penting:** Mengubah `wal_level` dari `replica` ke `logical` memerlukan restart PostgreSQL. Ini adalah satu-satunya momen restart yang dibutuhkan di server lama. Setelah ini, semua proses migrasi berjalan tanpa mengganggu operasi database.

Penjelasan parameter yang ditambahkan:

| Parameter | Fungsi |
| --- | --- |
| `wal_level` | Set ke `logical` supaya WAL mencatat informasi yang cukup untuk logical decoding |
| `max_replication_slots` | Jumlah maksimal replication slot, pgcopydb dan subscription masing-masing butuh satu slot |
| `max_wal_senders` | Jumlah maksimal proses yang bisa mengirim WAL secara bersamaan |

Selanjutnya, buat user khusus untuk replication dan beri akses yang dibutuhkan:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
CREATE ROLE migrator WITH REPLICATION LOGIN PASSWORD 'migrator123';
GRANT CONNECT ON DATABASE appdb TO migrator;
GRANT USAGE ON SCHEMA public TO migrator;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO migrator;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO migrator;
```

User `migrator` butuh privilege `REPLICATION` untuk membuat replication slot dan `SELECT` pada semua tabel untuk membaca data yang akan di-copy.

Buat juga **publication** di source yang mencakup semua tabel:

```sql
CREATE PUBLICATION migration_pub FOR ALL TABLES;
```

Verifikasi publication sudah terbuat:

```sql
SELECT * FROM pg_publication;
```

```
    pubname     | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate
----------------+----------+--------------+-----------+-----------+-----------+-------------
 migration_pub  |       10 | t            | t         | t         | t         | t
```

Publication `migration_pub` akan mem-publish semua perubahan (insert, update, delete, truncate) dari semua tabel di database `appdb`.

### Konfigurasi Target (Server Baru)

Buat direktori kerja di server baru:

```
$ mkdir -p /opt/pg-target && cd /opt/pg-target
```

Buat file `docker-compose.yml` untuk target:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: pg-target
    restart: unless-stopped
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pg_target_data:/var/lib/postgresql/data
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_replication_slots=5"
      - "-c"
      - "max_wal_senders=5"

volumes:
  pg_target_data:
```

> **Tip:** Target juga perlu `wal_level=logical` kalau di kemudian hari server ini akan menjadi source untuk migrasi berikutnya. Kalau tidak dibutuhkan, bisa diset ke `replica` atau dibiarkan default.

Jalankan target:

```
$ docker compose up -d
```

Verifikasi target sudah berjalan:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT 1;"
```

### Initial Copy dengan pgcopydb

**pgcopydb** adalah tool yang dirancang khusus untuk menyalin database PostgreSQL secara efisien. Dibanding `pg_dump`/`pg_restore`, pgcopydb melakukan copy data dan pembuatan index secara paralel, yang bisa mempercepat proses secara signifikan untuk database besar.

Jalankan pgcopydb dari container terpisah yang bisa connect ke kedua server. Buat file `docker-compose.pgcopydb.yml` di server target:

```yaml
services:
  pgcopydb:
    image: ghcr.io/dimitri/pgcopydb:latest
    container_name: pgcopydb
    environment:
      PGCOPYDB_SOURCE_PGURI: "postgresql://migrator:migrator123@10.10.10.11:5432/appdb"
      PGCOPYDB_TARGET_PGURI: "postgresql://devuser:devpassword@10.10.10.12:5432/appdb"
    volumes:
      - pgcopydb_work:/tmp/pgcopydb
    entrypoint: ["sleep", "infinity"]

volumes:
  pgcopydb_work:
```

Container ini sengaja dibuat dengan `entrypoint: sleep infinity` supaya bisa masuk ke dalamnya dan menjalankan perintah pgcopydb secara interaktif. Ini lebih aman daripada langsung menjalankan migrasi, karena bisa mengecek koneksi dulu sebelum mulai.

Jalankan container pgcopydb:

```
$ docker compose -f docker-compose.pgcopydb.yml up -d
```

Masuk ke container dan verifikasi koneksi ke kedua server:

```
$ docker compose -f docker-compose.pgcopydb.yml exec pgcopydb bash
```

Di dalam container, test koneksi:

```
$ pgcopydb ping
```

```
SOURCE: ok (10.10.10.11:5432)
TARGET: ok (10.10.10.12:5432)
```

Kalau koneksi ke kedua server berhasil, jalankan proses copy. pgcopydb punya perintah `clone` yang melakukan copy schema dan data sekaligus:

```
$ pgcopydb clone --skip-extensions --table-jobs 4 --index-jobs 4
```

Penjelasan flag:

| Flag | Fungsi |
| --- | --- |
| `--skip-extensions` | Tidak meng-copy extension, harus diinstall manual di target karena tidak semua extension bisa di-copy |
| `--table-jobs` | Jumlah worker paralel untuk copy data tabel |
| `--index-jobs` | Jumlah worker paralel untuk membuat index |

> **Warning:** Flag `--skip-extensions` penting karena beberapa extension seperti `pg_stat_statements` atau extension yang membutuhkan shared library mungkin tidak terinstall di target. Install extension yang dibutuhkan secara manual di target sebelum menjalankan pgcopydb.

Sebelum menjalankan `pgcopydb clone`, pastikan extension yang dibutuhkan sudah ada di target:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

Proses clone akan menampilkan progress:

```
 step   |    conn    |    duration    |    transfer    |  errors
--------+------------+----------------+----------------+---------
 schema |          1 |        2s 450ms|                |       0
  data  |          4 |     1m 23s 100ms|    8.5 GB     |       0
 index  |          4 |       45s 200ms|                |       0
 vacuum |          1 |       12s 800ms|                |       0
--------+------------+----------------+----------------+---------
 total  |            |     2m 23s 550ms|    8.5 GB     |       0
```

Setelah clone selesai, keluar dari container pgcopydb.

### Disable Foreign Key di Target

Sebelum mengaktifkan logical replication, ada langkah penting yang sering terlewat. Foreign key constraint di target bisa menyebabkan error saat subscription melakukan initial sync, karena urutan insert data antar tabel tidak dijamin sesuai dependency.

Masuk ke PostgreSQL target dan disable semua foreign key sementara:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN (
        SELECT conname, conrelid::regclass AS table_name
        FROM pg_constraint
        WHERE contype = 'f'
    ) LOOP
        EXECUTE format('ALTER TABLE %s DISABLE TRIGGER ALL', r.table_name);
    END LOOP;
END $$;
```

> **Penting:** Jangan lupa enable kembali trigger setelah subscription selesai sync dan sebelum cutover. Foreign key tetap ada di schema, hanya trigger enforcement-nya yang dinonaktifkan sementara.

### Mengaktifkan Logical Replication

Setelah initial copy selesai, aktifkan logical replication untuk menangkap perubahan yang terjadi di source sejak copy dimulai. Proses ini membuat **subscription** di target yang terhubung ke **publication** di source.

Masuk ke PostgreSQL target:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

Buat subscription yang terhubung ke publication di source:

```sql
CREATE SUBSCRIPTION migration_sub
    CONNECTION 'postgresql://migrator:migrator123@10.10.10.11:5432/appdb'
    PUBLICATION migration_pub
    WITH (
        copy_data = false,
        create_slot = true,
        enabled = true
    );
```

Penjelasan parameter subscription:

| Parameter | Fungsi |
| --- | --- |
| `copy_data` | Set `false` karena data sudah di-copy oleh pgcopydb, tidak perlu copy ulang |
| `create_slot` | Set `true` untuk membuat replication slot baru di source |
| `enabled` | Set `true` supaya subscription langsung aktif setelah dibuat |

Verifikasi subscription sudah aktif:

```sql
SELECT subname, subenabled, subconninfo FROM pg_subscription;
```

```
    subname     | subenabled |                          subconninfo
----------------+------------+--------------------------------------------------------------
 migration_sub  | t          | postgresql://migrator:migrator123@10.10.10.11:5432/appdb
```

Cek status replication dari sisi source:

```
$ docker compose exec postgres psql -U devuser -d appdb -c \
    "SELECT slot_name, active, confirmed_flush_lsn FROM pg_replication_slots;"
```

```
    slot_name    | active | confirmed_flush_lsn
-----------------+--------+---------------------
 migration_sub   | t      |           0/1A3B4C8
```

Slot aktif dan `confirmed_flush_lsn` menunjukkan posisi WAL terakhir yang sudah dikonfirmasi oleh target.

### Verifikasi Konsistensi Data

Sebelum cutover, pastikan data di source dan target sudah konsisten. Cek row count setiap tabel di kedua sisi.

Di source:

```
$ docker compose exec postgres psql -U devuser -d appdb -c \
    "SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables ORDER BY relname;"
```

Di target, jalankan query yang sama dan bandingkan hasilnya. Kalau ada perbedaan, tunggu beberapa saat karena logical replication bersifat asynchronous dan ada delay.

Untuk verifikasi yang lebih ketat, bandingkan checksum data:

```sql
-- Jalankan di kedua sisi, bandingkan hasilnya
SELECT md5(string_agg(t::text, '')) FROM (SELECT * FROM users ORDER BY id) t;
```

### Sinkronisasi Sequence

Ini bagian yang paling sering terlewat. **Logical replication tidak me-replicate sequence value**. Artinya sequence di target masih menunjuk ke nilai awal, bukan nilai terakhir yang digunakan di source. Kalau tidak ditangani, insert pertama di target setelah cutover akan mendapat conflict primary key.

Jalankan script berikut di target untuk menyinkronkan semua sequence dari source. Pertama, generate perintah `setval` dari source:

```
$ docker compose exec postgres psql -U devuser -d appdb -t -A -c \
    "SELECT 'SELECT setval(''' || schemaname || '.' || sequencename || ''', ' || last_value || ');' FROM pg_sequences WHERE schemaname = 'public';"
```

Output-nya berupa daftar perintah SQL seperti:

```sql
SELECT setval('public.users_id_seq', 15234);
SELECT setval('public.posts_id_seq', 89012);
SELECT setval('public.comments_id_seq', 234567);
```

Copy perintah tersebut dan jalankan di target:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
SELECT setval('public.users_id_seq', 15234);
SELECT setval('public.posts_id_seq', 89012);
SELECT setval('public.comments_id_seq', 234567);
```

> **Tip:** Tambahkan margin pada nilai sequence, misalnya tambah 100 atau 1000 dari nilai terakhir di source. Ini untuk mengantisipasi write yang masih masuk ke source antara saat query dijalankan dan cutover dilakukan.

### Enable Kembali Foreign Key

Setelah subscription selesai sync dan data sudah konsisten, enable kembali foreign key trigger di target:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN (
        SELECT conname, conrelid::regclass AS table_name
        FROM pg_constraint
        WHERE contype = 'f'
    ) LOOP
        EXECUTE format('ALTER TABLE %s ENABLE TRIGGER ALL', r.table_name);
    END LOOP;
END $$;
```

Verifikasi foreign key constraint masih valid:

```sql
SELECT conname, conrelid::regclass AS table_name, confrelid::regclass AS ref_table
FROM pg_constraint
WHERE contype = 'f';
```

### Cutover ke Database Baru

Proses cutover adalah momen di mana downtime terjadi. Targetnya adalah meminimalkan window ini. Urutan langkah cutover:

1. **Stop write ke source**: ubah konfigurasi aplikasi untuk menghentikan operasi write ke database lama. Operasi read masih bisa berjalan

2. **Tunggu replication catch up**: pastikan semua perubahan terakhir sudah ter-replicate ke target

```
$ docker compose exec postgres psql -U devuser -d appdb -c \
    "SELECT slot_name, confirmed_flush_lsn FROM pg_replication_slots WHERE slot_name = 'migration_sub';"
```

Bandingkan `confirmed_flush_lsn` dengan `pg_current_wal_lsn()` di source:

```
$ docker compose exec postgres psql -U devuser -d appdb -c "SELECT pg_current_wal_lsn();"
```

Kalau keduanya sama atau sangat dekat, replication sudah catch up.

3. **Sinkronkan sequence terakhir kali**: ulangi langkah sinkronisasi sequence seperti di atas untuk menangkap perubahan terakhir

4. **Arahkan aplikasi ke target**: ubah connection string aplikasi dari `10.10.10.11:5432` ke `10.10.10.12:5432`

5. **Verifikasi aplikasi berjalan normal**: pastikan aplikasi bisa read dan write ke database baru

### Cleanup Setelah Cutover

Setelah cutover berhasil dan aplikasi sudah berjalan normal di target, bersihkan konfigurasi replication.

Di target, hapus subscription:

```
$ docker compose exec postgres psql -U devuser -d appdb
```

```sql
ALTER SUBSCRIPTION migration_sub DISABLE;
ALTER SUBSCRIPTION migration_sub SET (slot_name = NONE);
DROP SUBSCRIPTION migration_sub;
```

> **Penting:** Urutan perintah di atas penting. Langsung `DROP SUBSCRIPTION` tanpa `DISABLE` dan `SET (slot_name = NONE)` terlebih dahulu akan mencoba menghapus replication slot di source. Kalau koneksi ke source sudah terputus, perintah DROP akan hang.

Di source, hapus publication dan replication slot:

```sql
DROP PUBLICATION migration_pub;
SELECT pg_drop_replication_slot('migration_sub');
```

Hapus juga user migrator kalau sudah tidak dibutuhkan:

```sql
DROP ROLE migrator;
```

## Tantangan yang Dihadapi

Tantangan terbesar ada di konfigurasi awal **logical replication**. Mengubah `wal_level` ke `logical` membutuhkan restart PostgreSQL, yang berarti ada downtime singkat di awal. Ini harus dikomunikasikan dan dijadwalkan, meskipun restart PostgreSQL biasanya hanya memakan waktu beberapa detik. Yang sering bikin kaget adalah kalau lupa menambah `max_replication_slots` atau `max_wal_senders`, subscription akan gagal terbuat tanpa error message yang informatif.

**Sequence** adalah gotcha terbesar dalam migrasi dengan logical replication. Logical replication hanya me-replicate perubahan data (DML), bukan perubahan pada sequence counter. Saya baru sadar ini setelah cutover pertama gagal karena insert pertama di target mendapat error duplicate key. Solusinya memang manual, query nilai terakhir dari source dan set di target, tapi harus dilakukan di momen yang tepat yaitu setelah semua write ke source dihentikan dan sebelum aplikasi mulai write ke target.

**Extension** juga perlu perhatian khusus. Tidak semua extension bisa di-copy oleh pgcopydb. Extension yang membutuhkan shared library seperti `pg_stat_statements` atau `timescaledb` harus diinstall manual di target sebelum proses copy dimulai. Kalau ada extension yang tidak tersedia di target, pgcopydb akan gagal saat mencoba membuat objek yang bergantung pada extension tersebut. Pendekatan yang aman adalah list semua extension di source dengan `\dx`, lalu install satu per satu di target sebelum migrasi.

## Insight dan Pembelajaran

- **pgcopydb jauh lebih cepat dari pg_dump/pg_restore**: kemampuan copy data dan build index secara paralel membuat perbedaan yang signifikan untuk database berukuran besar, proses yang biasanya berjam-jam bisa selesai dalam hitungan menit
- **Logical replication tidak menangani sequence**: ini adalah knowledge yang harus dimiliki sebelum memulai migrasi, bukan setelah cutover gagal. Selalu sync sequence sebagai langkah terpisah
- **copy_data = false menghemat waktu**: kalau sudah melakukan initial copy dengan pgcopydb, set `copy_data = false` pada subscription supaya tidak mengulang copy data yang sudah ada
- **Foreign key bisa menghambat initial sync**: disable trigger sementara di target selama proses sync, enable kembali sebelum cutover
- **Cutover window bisa sangat singkat**: dengan pgcopydb dan logical replication, downtime hanya terjadi saat stop write, sync sequence, dan switch connection string, biasanya kurang dari satu menit untuk database yang sudah fully synced
- **Urutan DROP SUBSCRIPTION itu penting**: langsung DROP tanpa DISABLE dan SET slot_name = NONE bisa menyebabkan hang kalau koneksi ke source sudah terputus

## Penutup

Kombinasi pgcopydb dan logical replication adalah pendekatan yang solid untuk migrasi database PostgreSQL tanpa downtime. pgcopydb menangani initial copy dengan cepat berkat paralelisme, sementara logical replication menjaga sinkronisasi data sampai cutover. Yang perlu diperhatikan secara khusus adalah sinkronisasi sequence, kompatibilitas extension, dan urutan langkah cutover supaya tidak ada data yang hilang atau conflict saat aplikasi mulai write ke database baru.

## Referensi

- [pgcopydb Documentation](https://pgcopydb.readthedocs.io/), diakses pada 2026-06-27
- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html), diakses pada 2026-06-27
- [PostgreSQL CREATE SUBSCRIPTION](https://www.postgresql.org/docs/current/sql-createsubscription.html), diakses pada 2026-06-27
- [PostgreSQL CREATE PUBLICATION](https://www.postgresql.org/docs/current/sql-createpublication.html), diakses pada 2026-06-27
