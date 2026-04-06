+++
draft = false
date = '2026-04-06'
title = 'Konfigurasi KVM Di Archlinux'
type = 'blog'
description = 'Panduan lengkap instalasi dan konfigurasi KVM/QEMU di Archlinux menggunakan libvirt dan virt-manager untuk virtualisasi dengan performa mendekati bare metal.'
image = ''
tags = ['kvm', 'qemu', 'libvirt', 'virtualisasi', 'archlinux']
+++

## Latar Belakang

Virtualisasi sudah jadi kebutuhan saya sebagai developer -- entah untuk testing environment, menjalankan OS lain, atau simulasi infrastruktur. Di Linux, **KVM (Kernel-based Virtual Machine)** adalah hypervisor yang langsung terintegrasi di kernel, artinya performanya mendekati bare metal tanpa overhead besar seperti VirtualBox.

## Permasalahan

Beberapa alasan kenapa virtualisasi di local machine itu penting:

- **Testing environment yang terisolasi** -- butuh environment bersih untuk testing deployment, konfigurasi server, atau eksperimen tanpa merusak host system
- **Menjalankan OS lain** -- kadang butuh Windows untuk aplikasi tertentu, atau distro Linux lain untuk compatibility testing
- **VirtualBox performanya kurang** -- virtualBox punya overhead yang cukup terasa terutama untuk I/O intensive workload
- **Simulasi infrastruktur** -- butuh beberapa VM sekaligus untuk simulasi cluster, networking, atau multi-tier architecture
- **Reproducible environment** -- VM bisa di-snapshot, di-clone, dan di-share ke tim untuk memastikan environment yang konsisten

Yang dibutuhkan adalah solusi virtualisasi yang performanya bagus, fleksibel, dan terintegrasi baik dengan Linux.

## Pendekatan Solusi

Ada beberapa opsi virtualisasi di Linux:

| Pendekatan             | Kelebihan                                                         | Kekurangan                                          |
| ---------------------- | ----------------------------------------------------------------- | --------------------------------------------------- |
| **KVM/QEMU + libvirt** | Performa bare metal, terintegrasi di kernel, GUI via virt-manager | Setup awal lebih banyak langkah                     |
| **VirtualBox**         | Mudah diinstall, GUI intuitif                                     | Performa lebih rendah, butuh kernel module terpisah |
| **VMware Workstation** | Fitur enterprise, snapshot bagus                                  | Proprietary, lisensi berbayar                       |
| **Docker/Podman**      | Ringan, cepat spin up                                             | Bukan full virtualisasi, hanya container            |
| **GNOME Boxes**        | Sangat simpel, minimal setup                                      | Fitur terbatas, kurang fleksibel                    |

Saya memilih **KVM/QEMU dengan libvirt** karena:

1. **Performa mumpuni** -- KVM berjalan langsung di kernel sebagai hypervisor type-1, tidak ada overhead translation layer

2. **Virtio driver** -- driver khusus yang dioptimasi untuk VM, jauh lebih cepat dari emulasi hardware biasa
3. **Ecosystem yang mature** -- libvirt, virsh, virt-manager memberikan management layer yang lengkap
4. **Sudah bawaan kernel** -- tidak perlu install kernel module terpisah seperti VirtualBox

> **Note:** Hypervisor type-1 (bare-metal) berjalan langsung di atas hardware, berbeda dengan type-2 (seperti VirtualBox) yang berjalan sebagai aplikasi di atas OS sehingga ada layer tambahan yang mengurangi performa.

## Implementasi Teknis

### Cek Dukungan Hardware Virtualization

Sebelum mulai, pastikan CPU mendukung hardware virtualization (VT-x untuk Intel, AMD-V untuk AMD):

```
$ LC_ALL=C.UTF-8 lscpu | grep Virtualization
```

Output yang diharapkan:

```
Virtualization:                     VT-x
```

Atau untuk AMD:

```
Virtualization:                     AMD-V
```

Jika tidak ada output, cek BIOS/UEFI dan pastikan fitur virtualization sudah di-enable. Bisa juga verifikasi lewat `/proc/cpuinfo`:

```
$ grep -cE 'vmx|svm' /proc/cpuinfo
```

Jika hasilnya lebih dari `0`, CPU mendukung virtualization. `vmx` untuk Intel, `svm` untuk AMD.

### Cek Kernel Module

Pastikan module KVM sudah loaded di kernel:

```
$ lsmod | grep kvm
```

Output yang diharapkan (contoh untuk Intel):

```
kvm_intel             XXX  0
kvm                   XXX  1 kvm_intel
```

Jika module belum loaded, load secara manual:

```
$ sudo modprobe kvm_intel
```

Atau untuk AMD:

```
$ sudo modprobe kvm_amd
```

### Instalasi Paket

Install semua paket yang dibutuhkan:

```
$ sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat
```

Penjelasan masing-masing paket:

- **qemu** -- emulator dan virtualizer utama, menyediakan emulasi hardware dan integrasi KVM
- **virt-manager** -- GUI berbasis GTK untuk membuat dan mengelola VM
- **virt-viewer** -- console viewer untuk mengakses display VM via SPICE/VNC
- **dnsmasq** -- DHCP dan DNS server untuk NAT networking di VM
- **vde2** -- virtual distributed ethernet, untuk konektivitas jaringan antar VM
- **bridge-utils** -- utility untuk membuat dan mengelola bridge network
- **openbsd-netcat** -- network utility yang dibutuhkan libvirt untuk komunikasi

### Konfigurasi Libvirt

#### Aktifkan Service

```
$ sudo systemctl enable --now libvirtd
```

Verifikasi service berjalan:

```
$ sudo systemctl status libvirtd
```

#### Tambahkan User ke Group

Agar bisa mengelola VM tanpa `sudo`, tambahkan user ke group `libvirt`:

```
$ sudo usermod -aG libvirt $(whoami)
```

**Penting:** Logout dan login kembali agar perubahan group berlaku. Verifikasi dengan:

```
$ groups
```

Pastikan `libvirt` muncul di daftar group.

#### Konfigurasi QEMU

Edit file `/etc/libvirt/qemu.conf` untuk menjalankan QEMU sebagai user biasa (bukan root):

```
$ sudo vim /etc/libvirt/qemu.conf
```

Ubah parameter berikut:

```conf
user = "your_username"
group = "libvirt"
```

Ganti `your_username` dengan username kamu. Ini membuat QEMU process berjalan sebagai user biasa, yang lebih aman dan menghindari masalah permission saat mengakses ISO atau disk image dari home directory.

Restart libvirtd setelah perubahan:

```
$ sudo systemctl restart libvirtd
```

### Konfigurasi Network

#### Default NAT Network

Libvirt menyediakan default NAT network (`192.168.122.0/24`) yang memungkinkan VM mengakses internet melalui host. Aktifkan dan set autostart:

```
$ sudo virsh net-start default
$ sudo virsh net-autostart default
```

Verifikasi network aktif:

```
$ sudo virsh net-list --all
```

Output yang diharapkan:

```
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

NAT network sudah cukup untuk kebanyakan use case -- VM bisa akses internet dan berkomunikasi satu sama lain lewat bridge `virbr0`.


### Membuat Virtual Machine

#### Via Virt-Manager (GUI)

Buka virt-manager:

```
$ virt-manager
```

Langkah membuat VM baru:

1. Klik **Create a new virtual machine**
2. Pilih **Local install media (ISO image)**
3. Browse ke file ISO yang sudah didownload
4. Set alokasi **RAM** dan jumlah **CPU**
5. Buat disk virtual -- pilih format **qcow2** (mendukung snapshot dan thin provisioning)
6. Di bagian **Network**, pastikan menggunakan **Virtual network 'default': NAT**
7. Centang **Customize configuration before install** untuk tuning tambahan

#### Tuning Performa di Virt-Manager

Sebelum klik install, lakukan beberapa optimasi:

**CPU:** Di tab **CPUs**, set **CPU model** ke `host-passthrough`. Ini mengekspos fitur CPU host secara langsung ke VM, memberikan performa terbaik.

**Disk:** Di tab **Disk**, ubah **Disk bus** ke `VirtIO`. Virtio jauh lebih cepat dari emulasi IDE atau SATA karena menggunakan para-virtualization.

**Network:** Di tab **NIC**, ubah **Device model** ke `virtio`. Sama seperti disk, virtio network adapter performanya jauh lebih baik dari emulasi e1000.

**Firmware:** Di tab **Overview** > **Firmware**, pilih `UEFI x86_64: /usr/share/edk2/x64/OVMF_CODE.4m.fd` untuk boot mode UEFI. Dibutuhkan untuk Windows 11 atau jika ingin secure boot.

#### Via Command Line (virsh)

Untuk yang lebih suka CLI, bisa membuat VM dengan `virt-install`:

```
$ virt-install \
  --name archlinux-vm \
  --ram 4096 \
  --vcpus 4 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/archlinux-vm.qcow2,size=40,bus=virtio,format=qcow2 \
  --cdrom /var/lib/libvirt/images/archlinux.iso \
  --network network=default,model=virtio \
  --os-variant archlinux \
  --graphics spice \
  --boot uefi
```

## Tantangan yang Dihadapi

**Default network yang tidak otomatis aktif** setelah reboot. Meskipun libvirtd sudah di-enable, default network perlu di-set autostart secara terpisah dengan `virsh net-autostart default`. Tanpa ini, VM tidak bisa akses internet setelah host di-reboot.

## Insight dan Pembelajaran

Beberapa insight setelah menggunakan KVM di Archlinux:

- **Perbedaan performa KVM vs VirtualBox sangat terasa** -- terutama untuk disk I/O. VM yang sama terasa jauh lebih responsif di KVM dengan virtio dibanding VirtualBox, bahkan dengan konfigurasi resource yang identik.
- **Fitur Snapshot** -- sebelum melakukan update besar atau eksperimen, buat snapshot dulu. Kalau ada yang salah, revert dalam hitungan detik. Ini jauh lebih cepat dari backup manual.
- **`host-passthrough` CPU mode wajib diaktifkan** -- default CPU model di virt-manager biasanya generic yang tidak mengekspos semua fitur CPU. Dengan `host-passthrough`, VM mendapat akses penuh ke fitur CPU host termasuk instruksi AES-NI, AVX, dan lainnya.
- **Nested virtualization** -- jika butuh menjalankan VM di dalam VM (misalnya untuk belajar Kubernetes), untuk mengaktifkan nested virtualization cukup dengan menambahkan `options kvm_intel nested=1` di `/etc/modprobe.d/kvm_intel.conf` di host kvmnya.

## Penutup

KVM dengan libvirt dan virt-manager adalah solusi virtualisasi terbaik di Linux -- performa mendekati bare metal karena terintegrasi langsung di kernel, management yang fleksibel lewat GUI maupun CLI, dan ecosystem yang mature. Setup awalnya memang lebih banyak langkah dibanding VirtualBox, tapi hasilnya sepadan: VM yang lebih cepat, lebih stabil, dan lebih production-ready. 

## Referensi

- [Arch Wiki - KVM](https://wiki.archlinux.org/title/KVM) -- Diakses pada 2026-04-06
- [Arch Wiki - QEMU](https://wiki.archlinux.org/title/QEMU) -- Diakses pada 2026-04-06
- [Arch Wiki - Libvirt](https://wiki.archlinux.org/title/Libvirt) -- Diakses pada 2026-04-06
