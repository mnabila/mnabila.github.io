+++
draft = false
date = '2026-04-11'
title = 'Instalasi Kubernetes Dengan k0s di Debian Server'
type = 'blog'
description = 'Panduan instalasi Kubernetes cluster di Debian server menggunakan k0s dengan setup 1 controller dan 1 worker node tanpa ribet konfigurasi'
image = ''
tags = ['kubernetes', 'k0s', 'debian', 'container']
+++

## Latar Belakang

Kalau pernah setup Kubernetes pakai kubeadm, pasti tahu betapa banyaknya langkah yang harus dilakukan -- disable swap, load kernel modules, set sysctl parameters, install containerd, konfigurasi cgroup driver, baru install kubeadm/kubelet/kubectl. Belum lagi kalau ada satu langkah yang terlewat, debugging-nya bisa memakan waktu berjam-jam.

**k0s** hadir sebagai alternatif yang jauh lebih simpel. k0s adalah distribusi Kubernetes yang dikemas dalam **single binary** -- semua komponen yang dibutuhkan (containerd, etcd, CoreDNS, kube-proxy, Metrics Server) sudah dibundel di dalamnya. Tidak perlu install dependency satu per satu, tidak perlu konfigurasi container runtime manual, dan cluster bisa jalan dalam hitungan menit. Yang paling penting, k0s tetap **CNCF certified** -- artinya 100% upstream Kubernetes, bukan versi modifikasi.

## Permasalahan

Beberapa masalah yang sering ditemui saat setup Kubernetes secara tradisional di Debian:

- **Banyak dependency yang harus diinstall manual** -- container runtime, kubelet, kubeadm, kubectl, CNI plugin, semuanya harus dikonfigurasi satu per satu dan versinya harus compatible
- **Persiapan sistem yang rawan terlewat** -- disable swap, load kernel modules `overlay` dan `br_netfilter`, set sysctl parameters. Kalau satu terlewat, cluster tidak akan jalan tapi error-nya sering tidak jelas
- **Konfigurasi container runtime yang tricky** -- containerd harus di-set `SystemdCgroup = true` supaya match dengan kubelet, dan ini sering jadi sumber masalah
- **Proses yang panjang untuk setup minimal** -- bahkan untuk cluster 1 controller + 1 worker, butuh belasan langkah sebelum cluster ready

Yang dibutuhkan adalah cara install Kubernetes yang lebih straightforward tanpa mengorbankan kompatibilitas dengan upstream Kubernetes.

## Pendekatan Solusi

Ada beberapa opsi untuk install Kubernetes dengan cara yang lebih simpel:

| Pendekatan | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **kubeadm** | Tool resmi dari Kubernetes, kontrol penuh | Banyak langkah manual, dependency management ribet |
| **k3s** | Ringan, single binary, cocok untuk edge/IoT | Pakai SQLite default (bukan etcd), beberapa komponen di-strip |
| **k0s** | Single binary, zero dependency, CNCF certified, etcd built-in | Komunitas lebih kecil dibanding kubeadm/k3s |
| **minikube** | Mudah untuk local development | Tidak cocok untuk multi-node atau production |

Saya memilih **k0s** karena:

1. **Zero friction** -- tidak perlu install dependency apapun selain binary k0s itu sendiri, semua sudah dibundel
2. **CNCF certified** -- 100% upstream Kubernetes, manifest dan tooling yang sudah ada tetap bisa dipakai tanpa modifikasi
3. **Etcd built-in** -- berbeda dengan k3s yang default pakai SQLite, k0s sudah include etcd sebagai datastore
4. **System requirements rendah** -- minimal 1 vCPU dan 1 GB RAM untuk controller, bahkan worker cuma butuh 0.5 GB RAM

## Implementasi Teknis

### Arsitektur Cluster

Berikut gambaran arsitektur yang akan di-setup:

| Node | Hostname | IP (contoh) | Role |
|------|----------|-------------|------|
| Server 1 | `k0s-controller` | `192.168.1.10` | Controller |
| Server 2 | `k0s-worker` | `192.168.1.11` | Worker |

Minimum system requirements:

| Komponen | CPU | RAM | Storage |
|----------|-----|-----|---------|
| Controller | 1 vCPU | 1 GB | SSD recommended |
| Worker | 1 vCPU | 0.5 GB | SSD recommended, min 15% free disk |

> **Note:** Pastikan kedua server sudah bisa saling komunikasi via network dan punya akses internet untuk pull image.

### Install k0s Binary (Semua Node)

Langkah ini dilakukan di **kedua node** -- controller dan worker.

Download k0s binary menggunakan script resmi:

```bash
$ curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
```

Script ini akan mendeteksi arsitektur sistem dan download binary yang sesuai. Kalau mau install versi spesifik:

```bash
$ curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo K0S_VERSION=v1.32.3+k0s.0 sh
```

Verifikasi instalasi:

```bash
$ k0s version
```

Sebelum lanjut, jalankan system check untuk memastikan sistem memenuhi requirements:

```bash
$ sudo k0s sysinfo
```

Command ini akan mengecek kernel version, cgroup support, dan komponen lain yang dibutuhkan. Pastikan semua item menunjukkan status OK.

### Generate Konfigurasi (Controller Node Only)

Langkah-langkah berikut **hanya dilakukan di controller node**.

Generate default config:

```bash
$ mkdir -p /etc/k0s
$ k0s config create > /etc/k0s/k0s.yaml
```

Beberapa field penting di konfigurasi:

| Field | Default | Keterangan |
|-------|---------|------------|
| `spec.api.address` | IP pertama non-localhost | IP untuk komunikasi antar komponen cluster |
| `spec.api.port` | `6443` | Port Kubernetes API server |
| `spec.storage.type` | `etcd` | Backend datastore, bisa juga `kine` untuk external DB |
| `spec.network.provider` | `kuberouter` | CNI plugin, opsi lain: `calico` atau `custom` |
| `spec.network.podCIDR` | `10.244.0.0/16` | Range IP untuk pod |
| `spec.network.serviceCIDR` | `10.96.0.0/12` | Range IP untuk service |

Untuk setup standar, default config sudah cukup. Tapi kalau mau ganti CNI ke Calico misalnya, edit bagian `spec.network`:

```yaml
spec:
  network:
    provider: calico
    calico:
      mode: vxlan
```

> **Note:** k0s support partial config -- field yang tidak didefinisikan akan otomatis pakai default value. Jadi tidak perlu tulis semua field kalau cuma mau ubah beberapa.

### Install dan Start Controller (Controller Node Only)

Install k0s sebagai service dan langsung start:

```bash
$ sudo k0s install controller -c /etc/k0s/k0s.yaml --start
```

Tunggu beberapa saat sampai controller ready, lalu cek status:

```bash
$ sudo k0s status
```

Output yang diharapkan:

```
Version: v1.32.3+k0s.0
Process ID: 12345
Role: controller
Workloads: false
```

Verifikasi node sudah terdaftar:

```bash
$ sudo k0s kubectl get nodes
```

Pada tahap ini belum ada node yang muncul karena controller secara default tidak menjalankan workload -- ini by design untuk isolasi control plane.

### Generate Token untuk Worker (Controller Node Only)

Buat join token untuk worker node:

```bash
$ sudo k0s token create --role=worker --expiry=24h > ~/worker-token
```

Parameter `--expiry` menentukan masa berlaku token. Default-nya tidak expire, tapi untuk keamanan sebaiknya set expiry.

Transfer file token ke worker node:

```bash
$ scp ~/worker-token user@192.168.1.11:~/worker-token
```

### Join Worker Node (Worker Node Only)

Langkah ini **hanya dilakukan di worker node**.

Pastikan k0s binary sudah terinstall (langkah pertama), lalu join ke cluster:

```bash
$ sudo k0s install worker --token-file ~/worker-token --start
```

Cek status worker:

```bash
$ sudo k0s status
```

Output yang diharapkan:

```
Version: v1.32.3+k0s.0
Process ID: 23456
Role: worker
Workloads: true
```

### Verifikasi Cluster (Controller Node)

Kembali ke controller node, cek semua node sudah terdaftar:

```bash
$ sudo k0s kubectl get nodes
```

Output seharusnya:

```
NAME           STATUS   ROLES    AGE     VERSION
k0s-worker     Ready    <none>   2m      v1.32.3+k0s.0
```

Cek semua system pod jalan dengan normal:

```bash
$ sudo k0s kubectl get pods -n kube-system
```

Semua pod seharusnya dalam status `Running`.

### Setup kubectl dari External Machine (Opsional)

Kalau mau akses cluster dari laptop atau machine lain, ambil kubeconfig dari controller:

```bash
# Di controller node
$ sudo cat /var/lib/k0s/pki/admin.conf
```

Copy output-nya ke machine yang mau dipakai, simpan sebagai `~/.kube/config`. Jangan lupa ganti `localhost` di field `server` dengan IP controller:

```bash
# Di local machine
$ mkdir -p ~/.kube
$ scp user@192.168.1.10:/var/lib/k0s/pki/admin.conf ~/.kube/config

# Ganti localhost dengan IP controller
$ sed -i 's/localhost/192.168.1.10/g' ~/.kube/config

# Test koneksi
$ kubectl get nodes
```

### Menghapus Cluster

Kalau mau reset dan mulai dari awal, jalankan di setiap node:

```bash
$ sudo k0s stop
$ sudo k0s reset
$ sudo reboot
```

> **Penting:** Command `k0s reset` akan menghapus semua data cluster termasuk etcd data, certificate, dan konfigurasi. Pastikan sudah backup data yang diperlukan sebelum menjalankan command ini.

## Tantangan yang Dihadapi

Tantangan pertama adalah **memahami perbedaan role controller dan worker di k0s**. Berbeda dengan kubeadm di mana control plane node secara default juga bisa menjalankan workload (dengan taint), di k0s controller secara default **tidak menjalankan workload sama sekali**. Ini by design untuk isolasi, tapi bisa membingungkan di awal karena `kubectl get nodes` tidak menampilkan controller. Kalau mau controller juga jalan sebagai worker, harus pakai flag `--enable-worker --no-taints` saat install.

Tantangan kedua adalah **transfer token ke worker node**. Token yang di-generate cukup panjang dan harus ditransfer secara utuh ke worker. Copy-paste manual lewat terminal bisa rawan terpotong. Cara paling aman adalah pakai `scp` atau simpan ke file lalu transfer.

Satu hal lagi -- **kubeconfig default pakai `localhost`** sebagai server address. Ini hanya bisa diakses dari controller node itu sendiri. Kalau mau akses dari machine lain, harus manual ganti ke IP controller. Langkah ini sering terlewat dan bikin bingung kenapa `kubectl` timeout dari laptop.

## Insight dan Pembelajaran

Beberapa hal yang bisa diambil dari pengalaman setup k0s:

- **Single binary itu game changer** -- tidak perlu install containerd, etcd, atau CNI plugin terpisah. Semua sudah ada di dalam binary k0s, jadi kemungkinan dependency conflict hampir tidak ada.
- **Jalankan `k0s sysinfo` sebelum install** -- command ini mengecek semua prerequisites secara otomatis. Lebih baik tahu masalahnya di awal daripada debugging setelah install.
- **Set expiry pada token** -- secara default token tidak expire, yang bisa jadi security risk. Selalu set `--expiry` saat generate token, terutama di environment yang accessible dari luar.
- **Simpan kubeconfig dengan benar** -- ambil dari `/var/lib/k0s/pki/admin.conf` di controller, ganti `localhost` dengan IP controller, baru simpan di `~/.kube/config` di machine yang mau dipakai.
- **Default CNI sudah cukup untuk kebanyakan kasus** -- Kube-Router yang jadi default di k0s sudah cukup untuk cluster kecil-menengah. Ganti ke Calico kalau butuh Network Policy yang lebih advanced.
- **k0s reset bersifat destruktif** -- command ini menghapus semua data termasuk etcd. Pastikan backup sebelum reset, terutama di environment yang sudah ada workload-nya.

## Penutup

Setup Kubernetes cluster di Debian dengan k0s jauh lebih straightforward dibanding kubeadm. Prosesnya bisa dirangkum dalam empat langkah: install binary, start controller, generate token, join worker. Tidak perlu install dependency tambahan, tidak perlu konfigurasi container runtime manual, dan tidak perlu disable swap atau load kernel modules. Dari setup minimal ini, cluster bisa di-scale dengan menambah worker node menggunakan token yang di-generate dari controller.

## Referensi

- [k0s Documentation - Quick Start Guide](https://docs.k0sproject.io/stable/) -- Diakses pada 2026-04-11
- [k0s Documentation - Manual Install (Multi-node)](https://docs.k0sproject.io/stable/k0s-multi-node/) -- Diakses pada 2026-04-11
- [k0s Documentation - Configuration](https://docs.k0sproject.io/stable/configuration/) -- Diakses pada 2026-04-11
- [k0s Documentation - System Requirements](https://docs.k0sproject.io/stable/system-requirements/) -- Diakses pada 2026-04-11
