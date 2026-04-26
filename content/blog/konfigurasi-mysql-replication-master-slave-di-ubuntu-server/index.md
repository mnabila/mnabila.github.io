+++
draft = false
date = '2026-04-23'
title = 'Konfigurasi MySQL Replication Master-Slave di Ubuntu Server'
type = 'blog'
description = 'Konfigurasi MySQL replication master-slave di Ubuntu Server 22.04 untuk memisahkan beban read dan write pada backend dengan traffic tinggi'
image = ''
tags = ['mysql', 'replication', 'database', 'ubuntu', 'server']
+++

## Latar Belakang

Punya backend service yang traffic-nya makin tinggi dan yang paling merasakan dampaknya adalah database dikarenakan semua query baik read maupun write masuk ke satu instance MySQL yang sama. Makin banyak user, response time makin lambat. Padahal kalau kalau dilihat di log servicenya mayoritas querynya adalah proses read bukan write.

Solusi yang paling masuk akal untuk kasus ini adalah **MySQL replication** dengan skema master-slave. Idenya simpel master khusus handle write, slave khusus handle read. Selain performa jadi lebih baik, slave juga bisa jadi semacam backup kalau sewaktu-waktu master bermasalah.

Setup ini saya lakukan di dua Ubuntu Server 22.04 dalam satu network. Satu jadi master, satu lagi jadi slave.

## Permasalahan

Pakai satu instance MySQL untuk semuanya mulai terasa limitasinya:

- **Semua query numpuk di satu server** -- read dan write rebutan resource di tempat yang sama, lock contention makin sering, response time naik
- **Tidak ada fallback** -- kalau server database down, ya wassalam seluruh aplikasi ikut mati dan user tidak bisa akses aplikasinya
- **Scaling mentok** -- nambah RAM dan CPU ada batasnya dan costnya mahal. Horizontal scaling tidak bisa tanpa replication
- **Backup ganggu production** -- jalankan `mysqldump` di server yang sama berarti table lock dan itu terasa banget di aplikasi
- **Read-heavy tapi tidak terdistribusi** -- rasio read vs write di aplikasi ini sekitar 80:20, tapi semua tetap masuk ke satu node

## Pendekatan Solusi

Ada beberapa cara untuk mendistribusikan beban database:

| Pendekatan                    | Kelebihan                                              | Kekurangan                                                         |
| ----------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------ |
| **Master-Slave Replication**  | Setup relatif simpel, slave bisa untuk read dan backup | Slave read-only, failover manual                                   |
| **Master-Master Replication** | Dua node bisa handle write                             | Risiko conflict tinggi, setup lebih ribet                          |
| **MySQL Group Replication**   | Built-in fault tolerance, automatic failover           | Butuh minimal 3 node, ada overhead consensus                       |
| **ProxySQL / MySQL Router**   | Query routing otomatis, load balancing                 | Nambah layer infrastruktur, bisa jadi single point of failure baru |

Pada kasus ini saya pilih **Master-Slave Replication** karena paling straightforward untuk kebutuhan ini. Cukup untuk misahin beban antara read dan write, dan slave bisa sekaligus jadi backup node. Di sisi aplikasi, tinggal arahkan query read ke slave dan write ke master.

Arsitektur yang dibangun:

| Komponen   | IP Address     | Peran                                      |
| ---------- | -------------- | ------------------------------------------ |
| **Master** | `192.168.1.10` | Handle write, kirim binary log ke slave    |
| **Slave**  | `192.168.1.11` | Terima binary log dari master, handle read |

## Implementasi Teknis

### Instalasi MySQL Server

Install MySQL di kedua server (master dan slave):

```
$ sudo apt update
$ sudo apt install mysql-server -y
```

Pastikan MySQL sudah jalan:

```
$ sudo systemctl status mysql
```

Lalu jalankan `mysql_secure_installation` untuk setup keamanan dasar:

```
$ sudo mysql_secure_installation
```

Kemudian Ikuti prompt-nya -- set root password, hapus anonymous user, disable remote root login, hapus test database. Ini dilakukan di kedua server.

### Konfigurasi Master

Buka file konfigurasi MySQL di server master `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```
$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Tambahkan atau ubah parameter berikut di bagian `[mysqld]`:

```ini
[mysqld]
server-id              = 1
log_bin                = /var/log/mysql/mysql-bin.log
binlog_do_db           = myapp_db
bind-address           = 0.0.0.0
```

Penjelasan tiap parameter:

| Parameter      | Fungsi                                                                            |
| -------------- | --------------------------------------------------------------------------------- |
| `server-id`    | ID unik tiap node, harus beda di setiap server                                    |
| `log_bin`      | Aktifkan binary logging dan tentukan lokasi file-nya                              |
| `binlog_do_db` | Database mana yang mau di-replicate, bisa tambah baris lagi kalau lebih dari satu |
| `bind-address` | Izinkan koneksi dari IP manapun, supaya slave bisa connect                        |

> **Tip:** Kalau mau replicate semua database, hapus baris `binlog_do_db`. Tapi untuk production, lebih baik spesifik biar binary log tidak membengkak.

Restart MySQL:

```
$ sudo systemctl restart mysql
```

### Membuat User Replication di Master

Login ke MySQL di master dan buat user khusus untuk replication:

```
$ sudo mysql -u root -p
```

```sql
CREATE USER 'slave'@'192.168.1.11' IDENTIFIED BY 'slave123';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'192.168.1.11';
FLUSH PRIVILEGES;
```

User `slave` cuma punya privilege `REPLICATION SLAVE` -- cukup untuk nerima binary log, tidak bisa ngubah data apapun. IP yang didaftarkan adalah IP slave server.

> **Penting:** Pakai password yang kuat untuk user replication. Meskipun privilege-nya terbatas, akses ke binary log artinya bisa lihat seluruh perubahan data.

### Mengambil Binary Log Position

Masih di MySQL shell master, lock table dulu untuk dapat posisi binary log yang konsisten:

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Output-nya kurang lebih seperti ini:

```
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      857 | myapp_db     |                  |
+------------------+----------+--------------+------------------+
```

Catat nilai `File` dan `Position` -- dua nilai ini nanti dipakai saat konfigurasi slave. Jangan tutup session ini dulu, table masih ke-lock.

### Dump Data dari Master

Buka terminal baru di server master, jalankan `mysqldump`:

```
$ sudo mysqldump -u root -p --opt myapp_db > myapp_db_dump.sql
```

Flag `--opt` mengaktifkan beberapa optimasi seperti `--quick`, `--add-drop-table`, dan `--add-locks` yang bikin import di slave lebih cepat.

Setelah dump selesai, balik ke MySQL shell yang tadi dan unlock table:

```sql
UNLOCK TABLES;
EXIT;
```

Transfer file dump ke slave:

```
$ scp myapp_db_dump.sql user@192.168.1.11:/tmp/
```

### Import Data ke Slave

Di server slave, buat database yang sama lalu import dump-nya:

```
$ sudo mysql -u root -p -e "CREATE DATABASE myapp_db;"
$ sudo mysql -u root -p myapp_db < /tmp/myapp_db_dump.sql
```

Ini memastikan data awal di slave sama persis dengan master sebelum replication mulai jalan.

### Konfigurasi Slave

Buka `/etc/mysql/mysql.conf.d/mysqld.cnf` di server slave:

```
$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Tambahkan atau ubah parameter berikut:

```ini
[mysqld]
server-id              = 2
relay-log              = /var/log/mysql/mysql-relay-bin.log
log_bin                = /var/log/mysql/mysql-bin.log
binlog_do_db           = myapp_db
read_only              = 1
```

| Parameter      | Fungsi                                                                      |
| -------------- | --------------------------------------------------------------------------- |
| `server-id`    | Harus beda dari master, di sini pakai `2`                                   |
| `relay-log`    | Tempat slave nyimpan event yang diterima dari master                        |
| `log_bin`      | Binary log di slave juga diaktifkan untuk keperluan chaining atau backup    |
| `binlog_do_db` | Sama dengan master, cuma replicate database yang ditentukan                 |
| `read_only`    | Slave cuma bisa nerima write dari replication thread, bukan dari user biasa |

Restart MySQL:

```
$ sudo systemctl restart mysql
```

### Menghubungkan Slave ke Master

Login ke MySQL di slave dan jalankan `CHANGE MASTER TO`:

```
$ sudo mysql -u root -p
```

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='slave',
  MASTER_PASSWORD='slave123',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=857;
```

Nilai `MASTER_LOG_FILE` dan `MASTER_LOG_POS` harus sesuai dengan output `SHOW MASTER STATUS` yang dicatat tadi. Ini yang nentuin dari mana slave mulai baca binary log.

Mulai replication:

```sql
START SLAVE;
```

### Verifikasi Status Replication

Cek apakah replication sudah jalan:

```sql
SHOW SLAVE STATUS\G
```

Yang perlu diperhatikan:

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
Master_Log_File: mysql-bin.000001
Read_Master_Log_Pos: 857
```

Dua hal yang harus bernilai `Yes`:

- **Slave_IO_Running** -- thread yang ngambil binary log dari master
- **Slave_SQL_Running** -- thread yang njalanin event dari relay log ke database slave

`Seconds_Behind_Master` menunjukkan seberapa jauh slave ketinggalan dari master. Idealnya `0`, tapi pas beban tinggi bisa naik sementara.

> **Tip:** Kalau `Slave_IO_Running` atau `Slave_SQL_Running` nilainya `No`, cek kolom `Last_IO_Error` atau `Last_SQL_Error` di output yang sama untuk tahu penyebabnya.

### Testing Replication

Buat perubahan di master, lalu cek di slave apakah datanya ikut masuk.

Di server master:

```sql
USE myapp_db;
CREATE TABLE test_replication (id INT PRIMARY KEY, message VARCHAR(100));
INSERT INTO test_replication VALUES (1, 'replication works');
```

Di server slave:

```sql
USE myapp_db;
SELECT * FROM test_replication;
```

```
+----+-------------------+
| id | message           |
+----+-------------------+
|  1 | replication works |
+----+-------------------+
```

Kalau data muncul di slave, berarti replication sudah jalan.

### Monitoring Replication

Replication bisa putus kapan saja tanpa warning, jadi monitoring itu wajib. Buat script sederhana yang bisa dijadwalkan via cron:

```bash
#!/bin/bash
TELEGRAM_BOT_TOKEN="your_bot_token"
TELEGRAM_CHAT_ID="your_chat_id"

send_telegram() {
    local message="$1"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d chat_id="${TELEGRAM_CHAT_ID}" \
        -d text="${message}" \
        -d parse_mode="Markdown" > /dev/null
}

SLAVE_STATUS=$(mysql -u root -p'your_password' -e "SHOW SLAVE STATUS\G" 2>/dev/null)
IO_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_IO_Running:" | awk '{print $2}')
SQL_RUNNING=$(echo "$SLAVE_STATUS" | grep "Slave_SQL_Running:" | awk '{print $2}')
LAG=$(echo "$SLAVE_STATUS" | grep "Seconds_Behind_Master:" | awk '{print $2}')

if [ "$IO_RUNNING" != "Yes" ] || [ "$SQL_RUNNING" != "Yes" ]; then
    send_telegram "*MySQL Replication Alert*
Replication is broken!
IO Running: $IO_RUNNING
SQL Running: $SQL_RUNNING
Host: $(hostname)"
fi

if [ "$LAG" -gt 60 ] 2>/dev/null; then
    send_telegram "*MySQL Replication Lag*
Lag: ${LAG}s
Host: $(hostname)"
fi
```

Script ini ngecek `Slave_IO_Running`, `Slave_SQL_Running`, dan replication lag. Kalau ada yang tidak beres, kirim notifikasi ke Telegram. Ganti `TELEGRAM_BOT_TOKEN` dengan token dari BotFather dan `TELEGRAM_CHAT_ID` dengan chat ID tujuan. Simpan di `/usr/local/bin/check_replication.sh` dan jadwalkan di cron tiap 5 menit.

## Tantangan yang Dihadapi

Yang paling bikin pusing adalah **mismatch binary log position**. Kalau lupa dicatat `File` dan `Position` dari `SHOW MASTER STATUS` sebelum dump, atau ada write yang nyelip di antara proses lock dan dump, slave bakal mulai dari posisi yang salah. Hasilnya data tidak konsisten sehingga ada transaksi yang kelewat atau malah terduplikasi. Kalau sudah begini, tidak ada jalan lain selain mengulangi dari awal: lock table, catat posisi, dump, unlock.

Masalah lain yang cukup bikin bingung di awal adalah **server-id yang sama**. Kalau lupa mengganti `server-id` di slave sehingga nilainya sama dengan master (atau dua-duanya masih default `1`), replication tidak mau jalan kemudian memunculkan error message-nya yang tidak jelas. MySQL cuma bilang IO thread gagal connect tanpa kasih tahu kalau penyebabnya duplicate server-id, dan pastikan tiap node punya `server-id` yang beda sebelum mulai.

**Replication lag** juga jadi masalah tersendiri, terutama pas ada bulk operation di master -- `ALTER TABLE` di tabel besar, batch insert jutaan row. Slave memproses event secara single-threaded (di konfigurasi default), jadi operasi berat di master bisa bikin slave ketinggalan jauh. Kalau ini sering terjadi, coba pakai parallel replication lewat parameter `slave_parallel_workers`. Satu lagi yang sering kelewat -- pastikan firewall di kedua server sudah buka port 3306. Koneksi yang putus di tengah replication bikin slave perlu di-resync ulang.

## Insight dan Pembelajaran

- **Urutan langkah itu krusial** -- lock table dulu, catat posisi binary log, dump data, baru unlock. Salah urutan, data di slave bisa inkonsisten dan debugging-nya makan waktu yang cukup lama
- **Binary log itu jantungnya replication** -- semua perubahan data di master tercatat di sini. Kalau binary log corrupt atau terhapus sebelum slave sempat baca bisa dikatakan replication putus
- **Read-only di slave bukan opsional** -- tanpa `read_only = 1`, siapapun yang punya akses write ke slave bisa bikin data drift. Sekali data di slave beda dari master sehingga mengembalikan konsistensinya jadi susah
- **Monitoring itu wajib, bukan nice-to-have** -- replication bisa putus kapan saja. Tanpa monitoring aktif, bisa-bisa slave sudah ketinggalan berjam-jam sebelum ada yang sadar
- **Pisahkan connection pool di aplikasi** -- pakai pool terpisah untuk read (ke slave) dan write (ke master). Jangan hardcode, pakai config supaya bisa switch node tanpa perlu deploy ulang

## Penutup

MySQL replication master-slave sudah cukup untuk meningkatkan performa dan availability database, terutama di workload yang read-heavy. Konfigurasinya butuh ketelitian di setiap langkah dari sinkronisasi binary log position, konsistensi data saat initial dump, sampai monitoring yang jalan terus. Tapi kalau setup-nya benar, beban database terdistribusi dengan baik dan aplikasi punya fallback kalau node utama bermasalah.

## Referensi

- [How To Set Up Replication in MySQL](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql) -- Diakses pada 2026-04-23
- [MySQL Master-Slave Replication](https://phoenixnap.com/kb/mysql-master-slave-replication) -- Diakses pada 2026-04-23
