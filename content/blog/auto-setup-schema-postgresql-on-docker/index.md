+++
draft = false
date = '2026-06-03'
title = 'Auto Setup Schema PostgreSQL on Docker'
type = 'blog'
description = 'Setup database PostgreSQL otomatis saat container pertama kali jalan menggunakan docker-entrypoint-initdb.d supaya tidak perlu migrasi manual tiap bikin environment baru'
image = ''
tags = ['postgresql', 'docker', 'docker-compose']
+++

## Latar Belakang

Setiap kali mulai project baru atau onboard developer lain, langkah yang paling sering terlewat adalah setup database. Pull repo, jalankan `docker compose up`, lalu baru sadar tabel belum ada karena belum ada yang jalankan migration script. Akhirnya harus masuk ke container, konek ke **PostgreSQL**, lalu jalankan SQL satu per satu secara manual.

Saya ingin setup yang benar-benar zero effort -- `docker compose up` dan database langsung siap pakai lengkap dengan schema, extension, bahkan seed data kalau perlu. Ternyata **Docker** image resmi PostgreSQL sudah menyediakan mekanisme untuk ini lewat direktori `/docker-entrypoint-initdb.d/`.

## Permasalahan

Beberapa masalah yang saya hadapi saat mengandalkan setup database manual:

- **Developer lupa jalankan migration** -- setelah clone repo dan `docker compose up`, aplikasi langsung error karena tabel belum ada dan developer baru bingung harus ngapain
- **Schema tidak konsisten antar developer** -- ada yang sudah apply migration terbaru, ada yang belum, debugging jadi susah karena skema-nya beda-beda
- **Rebuild container berarti setup ulang** -- setiap kali volume dihapus atau environment di-reset, harus jalankan migration manual lagi dari awal
- **Tidak ada single source of truth** -- migration script ada tapi tidak otomatis dijalankan, jadi developer harus tahu urutannya dan menjalankan manual

## Pendekatan Solusi

Ada beberapa cara untuk mengotomasi setup database di Docker:

| Pendekatan                                  | Kelebihan                                          | Kekurangan                                   |
| ------------------------------------------- | -------------------------------------------------- | -------------------------------------------- |
| **docker-entrypoint-initdb.d**              | Built-in di image resmi, tidak perlu tool tambahan | Hanya jalan saat volume kosong (fresh init)  |
| **Migration tool (Flyway, golang-migrate)** | Idempotent, bisa jalankan berkali-kali             | Butuh tool tambahan dan konfigurasi di CI/CD |
| **Custom entrypoint script**                | Fleksibel, bisa handle logic apapun                | Harus maintain script sendiri, rawan error   |
| **Init container (Kubernetes)**             | Terpisah dari database container                   | Overkill untuk local development             |

Saya pilih **docker-entrypoint-initdb.d** karena paling simpel dan sudah built-in di image resmi PostgreSQL. Tidak perlu install tool tambahan, cukup taruh file SQL atau shell script di direktori yang tepat. Untuk kebutuhan local development dan environment yang sering di-reset, pendekatan ini sudah lebih dari cukup.

## Implementasi Teknis

### Menyiapkan Struktur Project

Buat struktur direktori project seperti berikut:

```
project/
├── docker-compose.yml
└── migrations/
    ├── 01-init.sql
    ├── 02-create-tables.sql
    └── 03-data-seeder.sql
```

Direktori `migrations/` berisi file SQL yang akan di-mount ke container. File dieksekusi secara **alfanumeris** -- makanya pakai prefix angka untuk mengontrol urutan eksekusi.

### Membuat File SQL Migrasi

File pertama untuk mengaktifkan extension yang dibutuhkan:

```sql
-- migrations/01-init.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

File kedua untuk membuat tabel:

```sql
-- migrations/02-create-tables.sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

File ketiga untuk seed data awal:

```sql
-- migrations/03-data-seeder.sql
INSERT INTO users (username, email, password_hash)
VALUES ('admin', 'admin@example.com', crypt('password123', gen_salt('bf')))
ON CONFLICT (username) DO NOTHING;
```

Setiap file menggunakan `IF NOT EXISTS` dan `ON CONFLICT` supaya aman kalau dijalankan ulang -- meskipun dalam konteks `docker-entrypoint-initdb.d` script hanya dijalankan sekali. Perhatikan juga bahwa `03-data-seeder.sql` menggunakan fungsi `crypt()` dari extension `pgcrypto` -- makanya `01-init.sql` harus dieksekusi lebih dulu supaya extension sudah aktif saat seed data dijalankan.

### Menyiapkan Docker Compose

Buat file `docker-compose.yml` di root project:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: project-db
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: projectdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d projectdb"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

Yang penting di sini adalah baris volume `./migrations:/docker-entrypoint-initdb.d:ro`. Baris ini me-mount direktori `migrations/` lokal ke `/docker-entrypoint-initdb.d/` di dalam container dengan mode **read-only**. PostgreSQL akan otomatis mengeksekusi semua file `.sql` dan `.sh` di direktori ini saat inisialisasi database pertama kali.

### Menjalankan Container

Jalankan container dengan **Docker Compose**:

```
$ docker compose up -d
```

Cek log untuk memastikan migration script berhasil dijalankan:

```
$ docker compose logs postgres
```

Output yang diharapkan kurang lebih seperti ini:

```
project-db  | /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/01-init.sql
project-db  | CREATE EXTENSION
project-db  | /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/02-create-tables.sql
project-db  | CREATE TABLE
project-db  | CREATE TABLE
project-db  | CREATE INDEX
project-db  | CREATE INDEX
project-db  | /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/03-data-seeder.sql
project-db  | INSERT 0 1
```

Kalau semua file tereksekusi tanpa error, database sudah siap pakai.

### Verifikasi Hasil Migrasi

Masuk ke container dan cek apakah tabel sudah terbuat:

```
$ docker compose exec postgres psql -U devuser -d projectdb
```

Kemudian jalankan query untuk melihat daftar tabel:

```
projectdb=# \dt
```

Output-nya harusnya menampilkan tabel `users` dan `posts` yang sudah dibuat oleh migration script.

## Tantangan yang Dihadapi

Hal yang paling penting untuk dipahami dari mekanisme `docker-entrypoint-initdb.d` adalah script hanya dijalankan saat database di-initialize pertama kali, yaitu ketika volume `pgdata` masih kosong. Kalau volume sudah terisi data dari eksekusi sebelumnya, PostgreSQL akan skip semua script di direktori tersebut. Ini by design, bukan bug.

Konsekuensinya, kalau ada perubahan schema di file SQL, perubahan itu tidak akan otomatis terapply ke database yang sudah jalan. Saya harus hapus volume dulu dengan `docker compose down -v` lalu jalankan ulang `docker compose up -d` supaya PostgreSQL membaca ulang script dari awal. Untuk local development ini tidak masalah, tapi untuk production tentu tidak bisa begini, di production tetap butuh migration tool yang proper seperti **Flyway** atau **golang-migrate**.

Satu hal lagi yang sempat bikin bingung adalah urutan eksekusi file. PostgreSQL mengeksekusi file berdasarkan urutan alfanumerik nama file. Jadi kalau ada file bernama `create-tables.sql` dan `init-extensions.sql`, yang dijalankan duluan justru `create-tables.sql` karena huruf "c" lebih kecil dari "i". Solusinya simpel cukup pakai prefix angka seperti `01-`, `02-`, `03-` untuk memastikan urutan yang benar.

## Insight dan Pembelajaran

- **Built-in feature sering sudah cukup** -- sebelum install tool tambahan, cek dulu apakah image resmi sudah menyediakan mekanisme yang dibutuhkan, `docker-entrypoint-initdb.d` adalah contoh fitur yang sering terlewat
- **Prefix angka untuk kontrol urutan** -- jangan andalkan nama file untuk urutan eksekusi, pakai prefix numerik supaya urutannya eksplisit dan predictable
- **Pahami lifecycle volume Docker** -- banyak masalah debugging bisa dihindari kalau paham bahwa init script hanya jalan saat volume kosong, bukan setiap container restart
- **Pisahkan concern di file terpisah** -- satu file besar untuk semua migration memang bisa, tapi file terpisah per concern (extension, schema, seed) lebih mudah di-maintain dan di-debug
- **Read-only mount untuk safety** -- mount migration file dengan flag `:ro` supaya tidak ada proses di container yang bisa mengubah file migration

## Penutup

Mekanisme `docker-entrypoint-initdb.d` adalah cara paling simpel untuk mengotomasi setup database PostgreSQL di Docker. Cukup taruh file SQL di direktori yang tepat, mount ke container, dan database langsung siap pakai tanpa langkah manual. Untuk local development dan environment yang sering di-rebuild, pendekatan ini menghilangkan satu langkah yang sering terlewat dan menghemat waktu setup.

## Referensi

- [Docker Official Images - PostgreSQL](https://github.com/docker-library/docs/blob/master/postgres/README.md) -- Diakses pada 2026-06-03
- [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres) -- Diakses pada 2026-06-03
