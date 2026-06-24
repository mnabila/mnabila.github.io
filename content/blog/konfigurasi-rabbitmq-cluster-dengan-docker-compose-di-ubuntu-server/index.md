+++
draft = false
date = '2026-06-20'
title = 'Konfigurasi RabbitMQ Cluster dengan Docker Compose di Ubuntu Server'
type = 'blog'
description = 'Konfigurasi RabbitMQ 4.2 cluster 3 node dengan quorum queue menggunakan Docker Compose di 3 VM Ubuntu Server 22.04 untuk high availability pada message broker microservices'
image = ''
tags = ['rabbitmq', 'docker', 'docker-compose', 'high-availability', 'ubuntu']
+++

## Latar Belakang

Beberapa service microservices yang saya kelola menggunakan **RabbitMQ** sebagai message broker. Selama ini RabbitMQ hanya berjalan di satu server, satu instance, satu node, tanpa redundansi apapun. Selama trafik normal dan server stabil, tidak ada masalah. Tapi begitu server mengalami gangguan, entah restart mendadak, update OS, atau network issue, semua service yang bergantung pada RabbitMQ ikut terdampak. Queue tidak bisa diakses, message menumpuk di producer, dan consumer berhenti total.

Situasi ini klasik: **single point of failure**. Satu komponen down, seluruh pipeline messaging lumpuh. Saya perlu setup RabbitMQ yang punya redundansi, kalau satu node mati, message tetap aman dan service tetap bisa beroperasi.

## Permasalahan

Mengandalkan satu instance RabbitMQ untuk seluruh messaging microservices mulai jadi risiko:

- **Single point of failure**: satu server down berarti seluruh message broker tidak bisa diakses, semua service yang bergantung padanya ikut berhenti
- **Tidak ada replikasi message**: queue dan message hanya ada di satu node, kalau node itu mati sebelum consumer sempat proses, message hilang
- **Tidak ada failover otomatis**: ketika server bermasalah, harus manual restart dan pastikan semua service reconnect
- **Kapasitas terbatas di satu server**: connection dari banyak service numpuk di satu node, berpotensi bottleneck saat trafik tinggi
- **Monitoring terpusat di satu titik**: **Management UI** hanya ada di satu server, kalau server itu down tidak bisa monitoring sama sekali

## Pendekatan Solusi

Ada beberapa cara untuk mencapai high availability pada RabbitMQ:

| Pendekatan | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **RabbitMQ Cluster + Quorum Queue** | Replikasi message otomatis, toleran terhadap kegagalan node, built-in di RabbitMQ | Butuh minimal 3 node, konsumsi resource lebih tinggi |
| **RabbitMQ dengan Mirrored Queue (Classic)** | Sudah mature, banyak dokumentasi | Deprecated sejak RabbitMQ 3.13, performa lebih rendah dari quorum queue |
| **Federation / Shovel** | Cocok untuk multi-datacenter, loose coupling | Bukan clustering sesungguhnya, tidak ada shared state |
| **Managed Service (CloudAMQP, AWS MQ)** | Zero maintenance, otomatis scaling | Biaya recurring, data di pihak ketiga, latency network |

Saya pilih **RabbitMQ Cluster dengan Quorum Queue** karena ini adalah mekanisme replikasi yang direkomendasikan RabbitMQ untuk high availability.

**Quorum Queue** adalah tipe queue di RabbitMQ yang mereplikasi data ke beberapa node sekaligus menggunakan **Raft consensus protocol**. Cara kerjanya: setiap message yang masuk tidak langsung dianggap **tersimpan** sampai majority node (lebih dari setengah) mengonfirmasi bahwa mereka sudah menerima salinan message tersebut. Dengan 3 node, majority berarti minimal 2 node harus mengonfirmasi. Kalau 1 node mati, 2 node yang tersisa masih memenuhi majority sehingga queue tetap bisa menerima dan mengirim message tanpa gangguan.

Berbeda dengan classic queue yang hanya menyimpan message di satu node (dan hilang kalau node itu mati), quorum queue menjamin message tetap ada selama majority node masih hidup. Ini yang membuat quorum queue cocok untuk skenario high availability.

Arsitektur yang dibangun, 1 node per VM, masing-masing VM terpisah sehingga kehilangan VM manapun tidak mengganggu operasional cluster:

| Komponen | Hostname | VM | IP Address | Peran |
|----------|----------|----|------------|-------|
| **Node 1** | `rabbitmq-01` | VM 1 | `10.10.10.11` | Cluster node + Management UI |
| **Node 2** | `rabbitmq-02` | VM 2 | `10.10.10.12` | Cluster node + Management UI |
| **Node 3** | `rabbitmq-03` | VM 3 | `10.10.10.13` | Cluster node + Management UI |

Port yang digunakan (sama di ketiga VM):

| Port | Fungsi |
|------|--------|
| `5672` | AMQP, protokol utama untuk producer dan consumer |
| `15672` | Management UI, web dashboard untuk monitoring dan administrasi |
| `25672` | Cluster Communication, komunikasi internal antar node cluster |
| `4369` | EPMD (Erlang Port Mapper Daemon), service discovery antar node Erlang |

## Implementasi Teknis

### Prasyarat

Sebelum mulai, pastikan ketiga VM sudah memenuhi kondisi berikut:

- Ubuntu Server 22.04 LTS sudah terinstall
- Docker Engine dan Docker Compose v2 sudah terinstall di ketiga VM
- Ketiga VM bisa saling berkomunikasi melalui jaringan internal
- Hostname antar VM bisa saling resolve

Pembagian IP:

| VM | IP Address | Hostname |
|----|------------|----------|
| VM 1 | `10.10.10.11` | `rabbitmq-01` |
| VM 2 | `10.10.10.12` | `rabbitmq-02` |
| VM 3 | `10.10.10.13` | `rabbitmq-03` |

### Konfigurasi Hostname Resolution

Supaya setiap node RabbitMQ bisa resolve hostname node lainnya, tambahkan entry di `/etc/hosts` di **ketiga VM**:

```
$ sudo nano /etc/hosts
```

Tambahkan baris berikut:

```
10.10.10.11 rabbitmq-01
10.10.10.12 rabbitmq-02
10.10.10.13 rabbitmq-03
```

Verifikasi dari setiap VM:

```
$ ping -c 2 rabbitmq-01
$ ping -c 2 rabbitmq-02
$ ping -c 2 rabbitmq-03
```

Semua harus bisa reply. Kalau tidak, cek konfigurasi network interface di masing-masing VM terlebih dahulu.

### Menyiapkan Erlang Cookie

**Erlang Cookie** adalah shared secret yang digunakan node RabbitMQ untuk saling autentikasi dalam cluster. Semua node **wajib** menggunakan cookie yang sama, kalau berbeda node akan menolak bergabung ke cluster.

Generate cookie di salah satu VM:

```
$ openssl rand -hex 32
```

```
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2
```

Simpan nilai ini, akan digunakan di Docker Compose semua node.

> **Penting:** Erlang cookie bukan password untuk client. Ini khusus untuk autentikasi antar node Erlang/OTP dalam cluster. Jaga kerahasiaannya karena siapa pun yang punya cookie ini bisa join ke cluster.

### Docker Compose untuk Setiap VM

Konfigurasi Docker Compose di ketiga VM **identik** kecuali `container_name`, `hostname`, dan `RABBITMQ_NODENAME` yang disesuaikan dengan masing-masing node.

Buat direktori kerja di setiap VM:

```
$ mkdir -p /opt/rabbitmq && cd /opt/rabbitmq
```

Berikut file `docker-compose.yml` untuk **VM 1**:

```yaml
services:
  rabbitmq:
    image: rabbitmq:4.2-management
    container_name: rabbitmq-01
    hostname: rabbitmq-01
    restart: unless-stopped
    environment:
      RABBITMQ_ERLANG_COOKIE: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
      RABBITMQ_NODENAME: rabbit@rabbitmq-01
    ports:
      - "5672:5672"
      - "15672:15672"
      - "25672:25672"
      - "4369:4369"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    extra_hosts:
      - "rabbitmq-01:10.10.10.11"
      - "rabbitmq-02:10.10.10.12"
      - "rabbitmq-03:10.10.10.13"
    networks:
      - rabbitmq_network

volumes:
  rabbitmq_data:

networks:
  rabbitmq_network:
    driver: bridge
```

Untuk **VM 2**, ganti bagian berikut:

```yaml
    container_name: rabbitmq-02
    hostname: rabbitmq-02
    ...
      RABBITMQ_NODENAME: rabbit@rabbitmq-02
```

Untuk **VM 3**, ganti menjadi:

```yaml
    container_name: rabbitmq-03
    hostname: rabbitmq-03
    ...
      RABBITMQ_NODENAME: rabbit@rabbitmq-03
```

Sisanya sama persis, port, volume, `extra_hosts`, environment variable lainnya tidak berubah.

Penjelasan konfigurasi penting:

| Parameter | Fungsi |
|-----------|--------|
| `hostname` | Hostname container, harus match dengan `RABBITMQ_NODENAME` dan entry di `/etc/hosts` |
| `RABBITMQ_ERLANG_COOKIE` | Shared secret untuk autentikasi antar node cluster |
| `RABBITMQ_NODENAME` | Nama node dalam format `rabbit@hostname`, digunakan untuk identifikasi di cluster |
| `extra_hosts` | Menambahkan entry DNS di dalam container supaya bisa resolve hostname node lain |
| `rabbitmq_data` | Named volume untuk persist data RabbitMQ: queue, message, konfigurasi cluster |
| `restart: unless-stopped` | Container otomatis restart kecuali di-stop manual |

> **Warning:** Jangan gunakan `restart: always` untuk RabbitMQ cluster. Kalau ada masalah cluster yang perlu investigasi, container yang terus restart bisa memperparah situasi. Misalnya node yang terus join-leave cluster secara berulang.

### Menjalankan RabbitMQ di Semua VM

Jalankan Docker Compose di **ketiga VM**:

```
$ docker compose up -d
```

Cek status container:

```
$ docker compose ps
```

```
NAME          IMAGE                     COMMAND                  SERVICE    STATUS    PORTS
rabbitmq-01   rabbitmq:4.2-management   "docker-entrypoint.s…"   rabbitmq   Up        0.0.0.0:4369->4369/tcp, 0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp, 0.0.0.0:25672->25672/tcp
```

Pastikan status `Up` di ketiga VM sebelum lanjut ke langkah membentuk cluster.

### Membentuk RabbitMQ Cluster

RabbitMQ cluster dibentuk dengan cara node 2 dan node 3 **bergabung ke node 1**.

Di **VM 2**, join node 2 ke cluster:

```
$ docker exec -it rabbitmq-02 bash
```

Di dalam container:

```
$ rabbitmqctl stop_app
$ rabbitmqctl reset
$ rabbitmqctl join_cluster rabbit@rabbitmq-01
$ rabbitmqctl start_app
$ exit
```

Di **VM 3**, join node 3 ke cluster:

```
$ docker exec -it rabbitmq-03 bash
```

```
$ rabbitmqctl stop_app
$ rabbitmqctl reset
$ rabbitmqctl join_cluster rabbit@rabbitmq-01
$ rabbitmqctl start_app
$ exit
```

Penjelasan setiap command:

| Command | Fungsi |
|---------|--------|
| `stop_app` | Menghentikan RabbitMQ application tapi Erlang VM tetap jalan |
| `reset` | Menghapus semua data node supaya bisa join cluster dengan state bersih |
| `join_cluster` | Bergabung ke cluster yang sudah ada di node target |
| `start_app` | Menjalankan kembali RabbitMQ application setelah join cluster |

> **Penting:** `rabbitmqctl reset` akan **menghapus semua data** di node tersebut: queue, message, user, vhost. Pastikan node yang di-reset memang node baru atau node yang tidak punya data penting.

### Verifikasi Cluster

Cek status cluster dari **salah satu node**:

```
$ docker exec -it rabbitmq-01 rabbitmqctl cluster_status
```

```
Cluster status of node rabbit@rabbitmq-01 ...
Basics

Cluster name: rabbit@rabbitmq-01

Disk Nodes

rabbit@rabbitmq-01
rabbit@rabbitmq-02
rabbit@rabbitmq-03

Running Nodes

rabbit@rabbitmq-01
rabbit@rabbitmq-02
rabbit@rabbitmq-03

Maintenance status

Node: rabbit@rabbitmq-01, status: not under maintenance
Node: rabbit@rabbitmq-02, status: not under maintenance
Node: rabbit@rabbitmq-03, status: not under maintenance
```

Yang perlu dicek:

- **Disk Nodes**: ketiga node terdaftar sebagai disk node (data persisten ke disk)
- **Running Nodes**: ketiga node dalam status running
- **Maintenance status**: tidak ada node yang sedang maintenance

Kalau ada node yang tidak muncul di Running Nodes, cek koneksi network dan pastikan Erlang cookie-nya sama.

Management UI bisa diakses dari browser:

- Node 1: `http://10.10.10.11:15672`
- Node 2: `http://10.10.10.12:15672`
- Node 3: `http://10.10.10.13:15672`

Login dengan user `admin` dan password `password`.

### Membuat Quorum Queue

Buat quorum queue lewat `rabbitmqadmin`:

```
$ docker exec -it rabbitmq-01 rabbitmqadmin declare queue \
    name=order.processing \
    queue_type=quorum \
    durable=true \
    --username=admin \
    --password=password
```

```
queue declared
```

Verifikasi queue sudah terbuat dan ter-replicate:

```
$ docker exec -it rabbitmq-01 rabbitmqctl list_queues name type members --formatter=pretty_table
```

```
┌──────────────────┬────────┬─────────────────────────────────────────────────────────────────────┐
│ name             │ type   │ members                                                             │
├──────────────────┼────────┼─────────────────────────────────────────────────────────────────────┤
│ order.processing │ quorum │ [rabbit@rabbitmq-01, rabbit@rabbitmq-02, rabbit@rabbitmq-03]        │
└──────────────────┴────────┴─────────────────────────────────────────────────────────────────────┘
```

Kolom `members` menunjukkan queue ter-replicate di ketiga node. Message yang masuk akan di-commit setelah majority node (minimal 2) mengonfirmasi penerimaan.

Untuk membuat quorum queue sebagai **default** di seluruh cluster:

```
$ docker exec -it rabbitmq-01 rabbitmqctl set_policy quorum-default \
    ".*" \
    '{"queue-mode":"default","queue-version":2}' \
    --priority 0 \
    --apply-to queues
```

### Pengujian Failover

Untuk membuktikan cluster benar-benar high available, kita simulasikan kegagalan node.

Pertama, publish beberapa message ke quorum queue:

```
$ docker exec -it rabbitmq-01 bash -c '
for i in $(seq 1 10); do
    rabbitmqadmin publish \
        exchange=amq.default \
        routing_key=order.processing \
        payload="{\"order_id\": $i, \"status\": \"pending\"}" \
        --username=admin \
        --password=password
done'
```

Cek jumlah message di queue:

```
$ docker exec -it rabbitmq-01 rabbitmqctl list_queues name messages --formatter=pretty_table
```

```
┌──────────────────┬──────────┐
│ name             │ messages │
├──────────────────┼──────────┤
│ order.processing │ 10       │
└──────────────────┴──────────┘
```

Sekarang **matikan node 3** di VM 3:

```
$ docker compose down
```

Cek status cluster dari VM 1:

```
$ docker exec -it rabbitmq-01 rabbitmqctl cluster_status
```

Node 3 seharusnya tidak ada di Running Nodes. Tapi queue tetap beroperasi normal karena 2 node masih aktif (majority terpenuhi):

```
$ docker exec -it rabbitmq-01 rabbitmqctl list_queues name messages --formatter=pretty_table
```

```
┌──────────────────┬──────────┐
│ name             │ messages │
├──────────────────┼──────────┤
│ order.processing │ 10       │
└──────────────────┴──────────┘
```

Producer juga masih bisa publish message baru:

```
$ docker exec -it rabbitmq-01 rabbitmqadmin publish \
    exchange=amq.default \
    routing_key=order.processing \
    payload='{"order_id": 11, "status": "pending"}' \
    --username=admin \
    --password=password
```

```
Message published
```

Cluster tetap menerima write karena majority (2 dari 3 node) masih aktif. Kehilangan 1 VM manapun tidak mengganggu operasional cluster.

Nyalakan kembali node 3 di VM 3:

```
$ docker compose up -d
```

Node 3 akan otomatis rejoin cluster dan menerima data replication dari node lainnya:

```
$ docker exec -it rabbitmq-03 rabbitmqctl cluster_status
```

Ketiga node seharusnya kembali muncul di Running Nodes.

## Tantangan yang Dihadapi

Tantangan pertama yang cukup membingungkan adalah **hostname resolution antar container di VM berbeda**. RabbitMQ cluster bergantung pada Erlang distribution protocol yang menggunakan hostname untuk identifikasi node. Container Docker secara default punya hostname sendiri yang terisolasi dan tidak bisa resolve hostname container di VM lain. Konfigurasi `extra_hosts` di Docker Compose menyelesaikan masalah ini dari sisi container, tapi kalau network interface antar VM belum terhubung dengan benar, `rabbitmqctl join_cluster` tetap gagal dengan error `unable to connect to node`. Yang membingungkan, error ini juga muncul kalau Erlang cookie tidak match, jadi harus cek dua hal sekaligus.

**Erlang cookie mismatch** juga jadi masalah yang tidak langsung obvious. RabbitMQ menyimpan Erlang cookie di `/var/lib/rabbitmq/.erlang.cookie` di dalam container. Kalau volume sudah pernah digunakan dengan cookie yang berbeda, cookie dari environment variable bisa di-ignore karena file cookie yang ada di volume lebih diprioritaskan. Solusinya adalah pastikan volume bersih sebelum deploy pertama kali, atau hapus file `.erlang.cookie` di volume kalau perlu ganti cookie.

Satu hal lagi, **Docker Compose tidak menangani orchestration lintas host**. Setiap VM menjalankan Docker Compose sendiri-sendiri tanpa tahu keberadaan container di VM lain. Proses join cluster harus dilakukan manual setelah semua container running. Kalau ada node yang restart dan gagal rejoin otomatis, perlu masuk ke container dan join ulang secara manual. Untuk production yang butuh self-healing, pertimbangkan Docker Swarm atau Kubernetes.

## Insight dan Pembelajaran

- **RabbitMQ cluster bukan database replication**: cluster RabbitMQ berbagi metadata (exchange, binding, user) secara otomatis, tapi message di classic queue hanya ada di satu node. Untuk replikasi message, harus pakai quorum queue secara eksplisit
- **1 node per VM adalah distribusi ideal**: kehilangan VM manapun tidak mengganggu cluster karena 2 node yang tersisa masih memenuhi majority. Menaruh lebih dari 1 node di VM yang sama mengurangi efektivitas fault tolerance
- **Erlang cookie itu krusial tapi mudah terlewat**: satu karakter berbeda di cookie dan cluster gagal terbentuk. Selalu generate cookie sekali dan distribusikan ke semua node, jangan biarkan masing-masing node generate sendiri
- **Persistent volume bukan opsional**: tanpa volume, restart container berarti kehilangan semua data, queue definition, message, cluster membership. Node yang restart tanpa volume akan dianggap node baru dan perlu join cluster ulang
- **Management UI bukan mekanisme replikasi**: Management UI hanya untuk monitoring dan administrasi. Replikasi data ditangani oleh Erlang distribution protocol dan Raft consensus, bukan oleh Management plugin
- **Konfigurasi identik di semua node**: dengan 1 node per VM, Docker Compose di setiap VM nyaris sama. Yang berbeda hanya hostname dan node name, sehingga mudah di-template dan di-automate

## Penutup

RabbitMQ cluster 3 node dengan quorum queue di 3 VM terpisah memberikan high availability yang solid untuk message broker microservices. Kehilangan 1 VM manapun tidak berdampak pada operasional cluster karena 2 node yang tersisa masih memenuhi majority untuk Raft consensus. Yang perlu diingat adalah cluster ini bergantung pada konsistensi Erlang cookie, hostname resolution yang benar antar VM, dan persistent volume supaya data tidak hilang saat container restart.

## Referensi

- [RabbitMQ Clustering Guide](https://www.rabbitmq.com/docs/clustering), diakses pada2026-06-20
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues), diakses pada2026-06-20
- [RabbitMQ Docker Official Image](https://hub.docker.com/_/rabbitmq), diakses pada2026-06-20
- [Deploying RabbitMQ with Docker](https://www.rabbitmq.com/docs/docker), diakses pada2026-06-20
