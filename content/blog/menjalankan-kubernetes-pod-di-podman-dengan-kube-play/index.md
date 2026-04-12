+++
draft = false
date = '2026-04-11'
title = 'Menjalankan Kubernetes Pod di Podman Dengan Kube Play'
type = 'blog'
description = 'Cara menjalankan Kubernetes YAML manifest langsung di Podman menggunakan perintah podman kube play tanpa perlu cluster Kubernetes'
image = ''
tags = ['podman', 'kubernetes', 'container']
+++

## Latar Belakang

Kubernetes sudah jadi standar untuk container orchestration, tapi tidak semua situasi butuh full cluster untuk menjalankan pod. Kadang kita cuma mau test YAML manifest yang baru ditulis, atau jalankan beberapa container di local tanpa harus setup minikube, kind, atau cluster sungguhan. Rasanya overkill kalau harus spin up cluster cuma untuk validasi satu pod.

Ternyata Podman punya fitur yang bisa menyelesaikan masalah ini: **`podman kube play`**. Command ini bisa membaca Kubernetes YAML manifest dan langsung membuat pod beserta container-nya di local machine -- tanpa cluster, tanpa daemon, tanpa ribet.

## Permasalahan

Beberapa masalah yang sering ditemui saat ingin test atau jalankan Kubernetes manifest di local:

- **Setup cluster yang overkill** -- minikube atau kind butuh resource yang tidak sedikit, dan setup-nya memakan waktu padahal cuma mau test satu pod
- **Perbedaan environment** -- Docker Compose formatnya berbeda dengan Kubernetes manifest, jadi kalau develop di local pakai Compose lalu deploy ke cluster pakai YAML, ada proses konversi yang rawan error
- **Dependency pada daemon** -- Docker butuh daemon yang jalan di background dengan root privilege, yang tidak selalu ideal untuk local development

Yang dibutuhkan adalah cara untuk langsung jalankan Kubernetes YAML di local tanpa overhead setup cluster dan tanpa butuh daemon.

## Implementasi Teknis

### Instalasi Podman

Untuk yang belum install Podman:

```bash
$ sudo pacman -S podman
```

Verifikasi instalasi:

```bash
$ podman --version
```

### Kubernetes YAML Manifest yang Didukung

Sebelum mulai, perlu tahu dulu jenis resource apa saja yang bisa dijalankan dengan `podman kube play`:

| Kind | Keterangan |
|------|------------|
| `Pod` | Menjalankan pod langsung |
| `Deployment` | Membuat pod sesuai jumlah replicas |
| `DaemonSet` | Menjalankan pod di setiap "node" (dalam konteks Podman, satu instance) |
| `PersistentVolumeClaim` | Membuat volume untuk persistent storage |
| `ConfigMap` | Konfigurasi yang bisa di-mount ke pod |
| `Secret` | Data sensitif yang di-inject ke pod |

### Menjalankan Pod Sederhana

Buat file YAML seperti biasa. Contoh sederhana untuk pod Nginx:

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: docker.io/library/nginx:alpine
      ports:
        - containerPort: 80
          hostPort: 8080
```

Jalankan dengan satu command:

```bash
$ podman kube play nginx-pod.yaml
```

Output-nya kurang lebih seperti ini:

```
Pod:
e3b0c44298fc1c14-nginx-pod
Container:
a1b2c3d4e5f6-nginx
```

Pod sudah jalan dan bisa diakses di `http://localhost:8080`.

Untuk cek status:

```bash
# Lihat pod yang sedang jalan
$ podman pod ls

# Lihat container di dalam pod
$ podman ps --pod
```

Untuk menghentikan dan menghapus pod beserta container-nya:

```bash
$ podman kube play nginx-pod.yaml --down
```

### Multi-Container Pod: Backend Service + PostgreSQL

Sekarang contoh yang lebih real-world. Misalnya mau jalankan backend service dengan PostgreSQL dalam satu pod -- setup yang umum untuk API development:

```yaml
# backend-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-stack
  labels:
    app: backend
spec:
  containers:
    - name: postgres
      image: docker.io/library/postgres:16-alpine
      env:
        - name: POSTGRES_DB
          value: "appdb"
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_PASSWORD
          value: "secretpassword"
      volumeMounts:
        - name: pg-data
          mountPath: /var/lib/postgresql/data

    - name: api
      image: docker.io/library/node:22-alpine
      command: ["sh", "-c"]
      args:
        - |
          cat <<'APP' > /tmp/server.js
          const http = require('http');
          const server = http.createServer((req, res) => {
            res.writeHead(200, {'Content-Type': 'application/json'});
            res.end(JSON.stringify({
              status: 'ok',
              message: 'Backend service is running',
              db_host: process.env.DATABASE_HOST
            }));
          });
          server.listen(3000, () => console.log('API running on port 3000'));
          APP
          node /tmp/server.js
      env:
        - name: DATABASE_HOST
          value: "127.0.0.1"
        - name: DATABASE_PORT
          value: "5432"
        - name: DATABASE_NAME
          value: "appdb"
        - name: DATABASE_USER
          value: "appuser"
        - name: DATABASE_PASSWORD
          value: "secretpassword"
      ports:
        - containerPort: 3000
          hostPort: 3000

  volumes:
    - name: pg-data
      emptyDir: {}
```

Jalankan:

```bash
$ podman kube play backend-pod.yaml
```

Karena container dalam satu pod berbagi network namespace yang sama, backend service bisa akses PostgreSQL via `127.0.0.1:5432` -- sama persis seperti behaviour di Kubernetes. Test dengan curl:

```bash
$ curl http://localhost:3000
{"status":"ok","message":"Backend service is running","db_host":"127.0.0.1"}
```

### Menggunakan Deployment

Selain Pod, `podman kube play` juga support Deployment:

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: docker.io/library/nginx:alpine
          ports:
            - containerPort: 80
```

```bash
$ podman kube play nginx-deployment.yaml
```

Ini akan membuat 3 pod sesuai jumlah replicas yang didefinisikan.

### Menggunakan ConfigMap

`podman kube play` juga support ConfigMap yang bisa didefinisikan di file yang sama:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "development"
  APP_DEBUG: "true"
---
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: docker.io/library/nginx:alpine
      envFrom:
        - configMapRef:
            name: app-config
```

### Generate YAML dari Container yang Sudah Ada

Ini fitur yang sangat berguna. Kalau sudah punya container yang jalan dan ingin generate YAML-nya:

```bash
# Generate dari pod
$ podman kube generate my-pod > my-pod.yaml

# Generate dari container
$ podman kube generate my-container > my-container.yaml
```

Jadi bisa bikin container manual dulu, test sampai jalan, baru generate YAML-nya untuk dipakai ulang.

### Flag yang Sering Dipakai

Beberapa flag penting yang perlu diketahui:

| Flag | Fungsi |
|------|--------|
| `--down` | Menghentikan dan menghapus pod yang dibuat dari manifest |
| `--replace` | Replace pod yang sudah ada tanpa harus `--down` dulu |
| `--build` | Build image dari Containerfile/Dockerfile sebelum jalankan pod |
| `--network` | Tentukan network yang dipakai pod |
| `--configmap` | Load ConfigMap dari file terpisah |
| `--annotation` | Tambah annotation ke pod |

## Tantangan yang Dihadapi

Tantangan pertama adalah **port mapping yang berbeda dari Kubernetes**. Di Kubernetes ada Service dan Ingress untuk expose port, tapi di Podman harus pakai `hostPort` langsung di manifest. Ini berarti manifest yang dibuat untuk Podman perlu sedikit penyesuaian kalau mau deploy ke cluster sungguhan, atau sebaliknya.

Tantangan kedua adalah **tidak semua field di Kubernetes YAML di-support**. Resource seperti Service, Ingress, dan HPA tidak bisa dijalankan di Podman karena komponen-komponen tersebut butuh cluster. Jadi `podman kube play` cocoknya untuk test pod dan container, bukan untuk simulasi full cluster.

Satu hal lagi -- **image path harus lengkap**. Berbeda dengan Kubernetes yang punya default registry (`docker.io`), Podman lebih strict soal image reference. Kalau cuma tulis `nginx:alpine` tanpa prefix registry, Podman bisa bingung mau pull dari mana.

## Insight dan Pembelajaran

Beberapa hal yang bisa diambil dari penggunaan `podman kube play`:

- **Selalu pakai full image path** -- gunakan `docker.io/library/nginx:alpine` daripada cuma `nginx:alpine` untuk menghindari ambiguitas registry.
- **Gunakan `--down` untuk cleanup** -- jangan manual `podman pod rm`, pakai `podman kube play --down` supaya semua resource yang dibuat dari manifest tersebut ter-cleanup dengan benar.
- **Manfaatkan `kube generate`** -- kalau tidak hafal format YAML Kubernetes, bikin container biasa dulu lalu generate YAML-nya. Ini shortcut yang sangat membantu.
- **Satu manifest bisa dipakai dua arah** -- develop dan test di local dengan `podman kube play`, kalau sudah oke tinggal `kubectl apply` ke cluster. Manifest-nya sama, tidak perlu konversi.
- **Cocok untuk CI/CD pipeline** -- di environment CI yang tidak punya Kubernetes, `podman kube play` bisa jadi alternatif untuk test container dari manifest yang sama.

## Penutup

`podman kube play` adalah solusi yang praktis untuk menjalankan Kubernetes pod di local tanpa overhead setup cluster. Kelebihannya ada di tiga hal: daemonless sehingga tidak butuh service yang jalan di background, rootless by default sehingga lebih aman, dan kompatibel dengan Kubernetes YAML sehingga manifest yang sama bisa dipakai di local maupun di cluster. Untuk workflow development yang melibatkan Kubernetes manifest, fitur ini worth untuk dicoba.

## Referensi

- [Podman - podman-kube-play(1)](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html) -- Diakses pada 2026-04-11
