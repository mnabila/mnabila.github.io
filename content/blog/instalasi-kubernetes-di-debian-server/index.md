+++
draft = false
date = '2026-04-08'
title = 'Instalasi Kubernetes di Debian Server Dengan 1 Control Plane dan 1 Worker Node'
type = 'blog'
description = 'Panduan instalasi Kubernetes cluster di Debian server menggunakan kubeadm dengan setup minimal 1 control plane dan 1 worker node'
image = ''
tags = ['kubernetes', 'debian']
+++

## Latar Belakang

Kubernetes sudah menjadi standar container orchestration untuk saat ini. Kalau sebelumnya cukup pakai Docker Compose untuk menjalankan beberapa container, begitu kebutuhan mulai bertambah seperti scaling, self-healing, rolling update, service discovery sehigga kubernetes jadi pilihan yang sulit dihindari. Masalahnya, setup Kubernetes dari nol itu terkesan ribet dan belibet karena banyak komponen yang harus dikonfigurasi dengan benar.

Pada artikel ini akan membahas cara instalasi Kubernetes cluster di Debian server dengan setup paling minimal: **1 control plane dan 1 worker node**. Setup ini cocok untuk belajar, development, atau staging environment sebelum scale ke production.

## Permasalahan

Beberapa tantangan yang sering ditemui saat instalasi kubernetes di Debian:

- **Banyak komponen yang saling bergantung** -- container runtime, kubelet, kubeadm, kubectl, dan CNI plugin harus dikonfigurasi dengan benar dan saling compatible
- **Dokumentasi tersebar** -- ada yang untuk Ubuntu, ada yang untuk RHEL, dan tidak semua langkah applicable untuk Debian sehingga menyusahkan user
- **Konfigurasi sistem yang sering terlewat** -- seperti disable swap, load kernel modules, dan set sysctl parameters. Kalau terlewat satu saja, cluster tidak akan jalan 

Yang dibutuhkan adalah panduan yang fokus ke Debian dengan langkah-langkah yang jelas dari awal sampai cluster ready.

## Arsitektur Cluster

Sebelum mulai, berikut gambaran arsitektur yang akan di-setup:

| Node | Hostname | IP (contoh) | Role |
|------|----------|-------------|------|
| Server 1 | `k8s-cp` | `192.168.1.10` | Control Plane |
| Server 2 | `k8s-worker` | `192.168.1.11` | Worker Node |

Komponen yang akan diinstall:

| Komponen | Fungsi |
|----------|--------|
| `containerd` | Container runtime (CRI-compatible) |
| `kubeadm` | Tool untuk bootstrap cluster Kubernetes |
| `kubelet` | Agent yang jalan di setiap node, mengelola container |
| `kubectl` | CLI untuk interaksi dengan Kubernetes API |
| `Calico` | CNI plugin untuk networking antar pod |

> **Note:** Pastikan kedua server sudah bisa saling komunikasi via network dan punya akses internet untuk pull image dan download package.

## Implementasi Teknis

### Persiapan Sistem (Semua Node)

Langkah-langkah berikut dilakukan di **kedua node** -- control plane dan worker.

#### Update Sistem

```bash
$ sudo apt update && sudo apt upgrade -y
```

#### Disable Swap

Kubernetes membutuhkan swap dalam keadaan mati supaya scheduler bisa mengelola resource dengan akurat:

```bash
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Command pertama mematikan swap untuk session saat ini, command kedua meng-comment baris swap di `/etc/fstab` supaya swap tidak aktif lagi setelah reboot.

Verifikasi swap sudah mati:

```bash
$ free -h
```

Kolom `Swap` harus menunjukkan `0B` di semua field.

#### Load Kernel Modules

Kubernetes dan containerd membutuhkan module `overlay` dan `br_netfilter`:

```bash
$ sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```

Penjelasan masing-masing module:

| Module | Fungsi |
|--------|--------|
| `overlay` | Filesystem driver untuk container layer -- dibutuhkan containerd untuk manage image layers |
| `br_netfilter` | Memungkinkan iptables melihat traffic yang melewati Linux bridge -- dibutuhkan untuk network policy dan service routing |

#### Set Sysctl Parameters

```bash
$ sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

$ sudo sysctl --system
```

Parameter ini memastikan traffic antar pod bisa di-route dengan benar melalui iptables. Tanpa ini, pod di node berbeda tidak akan bisa komunikasi.

### Install Container Runtime -- containerd (Semua Node)

Kubernetes butuh container runtime yang CRI-compatible. Kita pakai **containerd** karena sudah mature, ringan, dan jadi default di banyak distribusi Kubernetes.

#### Install Dependencies

```bash
$ sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

#### Tambah Repository Docker

Containerd diambil dari repository Docker karena package `containerd.io` di sana lebih up-to-date:

```bash
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/debian.gpg

$ sudo add-apt-repository "deb [arch=$(dpkg --print-architecture)] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```

#### Install dan Konfigurasi containerd

```bash
$ sudo apt update
$ sudo apt install -y containerd.io
```

Generate default config dan aktifkan **SystemdCgroup**:

```bash
$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart dan enable containerd:

```bash
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```

> **Penting:** Setting `SystemdCgroup = true` itu wajib. Kubelet dan containerd harus pakai cgroup driver yang sama (systemd). Kalau tidak match, kubelet akan gagal start atau pod akan random crash.

### Install kubeadm, kubelet, kubectl (Semua Node)

#### Tambah Repository Kubernetes

```bash
$ sudo apt install -y apt-transport-https ca-certificates curl gpg

$ K8S_VERSION="v1.29"

$ sudo mkdir -p -m 755 /etc/apt/keyrings
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

$ echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

> **Note:** Repository `pkgs.k8s.io` adalah repository resmi yang baru. Repository lama `apt.kubernetes.io` sudah deprecated. Sesuaikan `K8S_VERSION` dengan versi yang diinginkan.

#### Install Package

```bash
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

`apt-mark hold` mencegah package ter-upgrade secara tidak sengaja saat `apt upgrade`. Upgrade Kubernetes harus dilakukan secara manual mengikuti prosedur resmi karena ada urutan yang harus diikuti (control plane dulu, baru worker).

Enable kubelet:

```bash
$ sudo systemctl enable --now kubelet
```

### Inisialisasi Control Plane (Control Plane Only)

Langkah-langkah berikut **hanya dilakukan di node control plane**.

#### Pull Container Images

Sebelum init, pull dulu semua image yang dibutuhkan supaya proses init tidak tergantung kecepatan download:

```bash
$ sudo kubeadm config images pull
```

#### Init Cluster

```bash
$ sudo kubeadm init --pod-network-cidr=10.10.0.0/16
```

Parameter `--pod-network-cidr` menentukan range IP yang akan digunakan untuk pod. Value ini harus match dengan konfigurasi CNI plugin yang akan diinstall nanti. Sesuaikan CIDR dengan CNI yang dipilih -- `10.244.0.0/16` untuk Flannel atau `10.10.0.0/16` untuk Calico.

Kalau init berhasil, output akan menampilkan:

1. **Perintah untuk setup kubectl** -- simpan dan jalankan ini
2. **Perintah `kubeadm join`** -- simpan ini untuk join worker node nanti

#### Setup kubectl

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verifikasi cluster sudah jalan:

```bash
$ kubectl get nodes
```

Output akan menunjukkan control plane dengan status `NotReady` -- ini normal karena CNI plugin belum diinstall.

### Install CNI Plugin (Control Plane Only)

CNI (Container Network Interface) plugin dibutuhkan supaya pod antar node bisa saling berkomunikasi. Tanpa CNI, semua node akan tetap `NotReady`. Kubernetes tidak menyediakan CNI default, jadi harus pilih dan install manual.

Pilih salah satu dari opsi berikut:

| CNI | Pod CIDR | Kelebihan | Kekurangan |
|-----|----------|-----------|------------|
| **Flannel** | `10.244.0.0/16` | Paling simpel, config minimal, cocok untuk belajar | Tidak support Network Policy |
| **Calico** | `10.10.0.0/16` (atau custom) | Support Network Policy, performa bagus, banyak dokumentasi | Sedikit lebih complex |

> **Penting:** CIDR yang dipakai harus match dengan `--pod-network-cidr` saat `kubeadm init`. Kalau sudah terlanjur init dengan CIDR yang berbeda, harus `kubeadm reset` dan init ulang.

#### Opsi A: Flannel

Flannel adalah pilihan paling simpel dan cocok untuk yang baru belajar Kubernetes atau cluster yang tidak butuh Network Policy.

Pastikan saat `kubeadm init` menggunakan `--pod-network-cidr=10.244.0.0/16`, lalu apply manifest:

```bash
$ kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Tunggu sampai pod Flannel running:

```bash
$ kubectl get pods -n kube-flannel -w
```

#### Opsi B: Calico

Calico lebih banyak fiturnya dan support Network Policy untuk mengontrol traffic antar pod, dan performanya bagus untuk cluster yang lebih besar.

Pastikan saat `kubeadm init` menggunakan `--pod-network-cidr=10.10.0.0/16`.

Install Tigera Operator:

```bash
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
```

Download dan edit custom resources:

```bash
$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml -O
```

Edit CIDR supaya match dengan `--pod-network-cidr`:

```bash
$ sed -i 's|192.168.0.0/16|10.10.0.0/16|g' custom-resources.yaml
```

Apply konfigurasi:

```bash
$ kubectl apply -f custom-resources.yaml
```

Tunggu sampai semua pod Calico running:

```bash
$ kubectl get pods -n calico-system -w
```

#### Verifikasi CNI

Setelah semua pod CNI `Running`, cek status node:

```bash
$ kubectl get nodes
```

Control plane seharusnya sudah `Ready`.

### Join Worker Node (Worker Node Only)

Langkah ini **hanya dilakukan di worker node**.

Jalankan perintah `kubeadm join` yang didapat dari output `kubeadm init` tadi:

```bash
$ sudo kubeadm join 192.168.1.10:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Ganti `<token>` dan `<hash>` dengan value yang sebenarnya dari output init.

Kalau token sudah expired (default 24 jam), generate ulang dari control plane:

```bash
$ kubeadm token create --print-join-command
```

#### Verifikasi Cluster

Kembali ke control plane, cek semua node:

```bash
$ kubectl get nodes
```

Output seharusnya:

```bash
NAME         STATUS   ROLES           AGE     VERSION
k8s-cp       Ready    control-plane   10m     v1.29.x
k8s-worker   Ready    <none>          2m      v1.29.x
```

Kedua node `Ready` berarti cluster sudah siap digunakan.

## Tantangan yang Dihadapi

Tantangan pertama adalah **urutan instalasi yang harus tepat**. Kernel modules dan sysctl harus di-set sebelum install containerd, containerd harus jalan sebelum kubelet, dan CNI harus diinstall setelah `kubeadm init`. Kalau urutan ini tidak diikuti, error yang muncul sering tidak jelas dan menyesatkan.

Tantangan kedua adalah **CIDR yang harus konsisten**. Value `--pod-network-cidr` saat `kubeadm init` harus match dengan konfigurasi CNI plugin. Kalau berbeda, pod akan bisa dibuat tapi tidak bisa komunikasi antar node dan ini silent failure yang baru ketahuan saat testing.

Satu hal lagi -- **`SystemdCgroup = true` di containerd itu critical**. Default config containerd menggunakan `cgroupfs`, tapi kubelet di Debian menggunakan `systemd`. Kalau tidak match, kubelet akan restart loop atau pod random crash tanpa error message yang jelas.

## Insight dan Pembelajaran

Beberapa hal yang bisa diambil dari pengalaman setup ini:

- **Jangan skip persiapan sistem** -- disable swap, load kernel modules, dan set sysctl parameters itu bukan opsional. Satu yang terlewat bisa bikin debugging berjam-jam.
- **Pin versi Kubernetes** -- gunakan `apt-mark hold` dan tentukan versi spesifik di repository. Upgrade Kubernetes harus dilakukan secara terkontrol, bukan melalui `apt upgrade`.
- **Simpan output `kubeadm init`** -- perintah join yang ditampilkan di akhir itu penting. Kalau hilang, bisa di-generate ulang tapi lebih baik disimpan dari awal.
- **Pilih CNI sesuai kebutuhan** -- Flannel untuk setup simpel tanpa Network Policy, Calico kalau butuh kontrol traffic antar pod. Keduanya production-ready, tapi fitur yang ditawarkan berbeda.
- **Test connectivity setelah CNI** -- setelah install CNI plugin, deploy simple pod di masing-masing node dan pastikan bisa saling ping. Jangan tunggu sampai deploy aplikasi baru menemukan network tidak jalan.
- **Repository `pkgs.k8s.io` adalah yang resmi** -- jangan pakai `apt.kubernetes.io` yang sudah deprecated. Package di repository lama tidak akan di-update lagi.

## Penutup

Setup Kubernetes cluster di Debian dengan 1 control plane dan 1 worker node itu straightforward kalau langkah-langkahnya diikuti dengan urutan yang benar. Kuncinya ada di tiga hal: persiapan sistem yang lengkap (swap, modules, sysctl), container runtime yang dikonfigurasi dengan cgroup driver yang match, dan CIDR yang konsisten antara `kubeadm init` dan CNI plugin. Dari setup minimal ini, cluster bisa di-scale dengan menambah worker node menggunakan perintah `kubeadm join` yang sama.

## Referensi

- [Kubernetes - Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) -- Diakses pada 2026-04-08
- [Computing For Geeks - Install Kubernetes Cluster on Debian 12](https://computingforgeeks.com/install-kubernetes-cluster-on-debian-12-bookworm/) -- Diakses pada 2026-04-08
- [Steve Notes - Kubernetes Provisioning](https://github.com/steve-notes/Kubernetes-Provisioning) -- Diakses pada 2026-04-08
