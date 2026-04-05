+++
draft = false
date = '2026-04-05'
title = 'Konfigurasi Driver Nvidia Rtx 3050 Di Archlinux Dengan Systemd Boot'
type = 'blog'
description = 'Cara install dan konfigurasi driver NVIDIA RTX 3050 di Archlinux yang menggunakan systemd-boot sebagai bootloader.'
image = ''
tags = ['nvidia', 'driver', 'systemd-boot', 'archlinux']
+++

## Latar Belakang

Setup driver NVIDIA di Archlinux itu bukan hal yang bisa dianggap remeh -- terutama kalau pakai Wayland. Tidak seperti di Ubuntu atau Fedora yang biasanya tinggal klik "Additional Drivers", di Arch kita harus setup semuanya secara manual: install package yang tepat, konfigurasi kernel module, enable DRM modesetting, dan memastikan semuanya dimuat saat boot.

Situasinya makin spesifik kalau menggunakan **systemd-boot** sebagai bootloader -- karena tidak pakai GRUB, kita tidak bisa tinggal edit `/etc/default/grub` untuk menambahkan kernel parameter. Harus langsung edit boot entry.

Artikel ini mendokumentasikan cara setup driver NVIDIA untuk **RTX 3050** di Archlinux dengan **systemd-boot** dan kernel **linux-zen**.

## Permasalahan

Tanpa driver yang dikonfigurasi dengan benar, beberapa masalah yang bisa muncul:

- **Layar hitam saat boot** -- kernel tidak bisa menginisialisasi GPU dengan benar
- **Wayland tidak bisa jalan** -- GNOME atau KDE fallback ke Xorg karena DRM modesetting tidak aktif
- **Tearing dan performa buruk** -- tanpa driver proprietary, nouveau (driver open source default) tidak memberikan performa yang memadai untuk GPU modern
- **Suspend/resume rusak** -- GPU tidak bisa bangun dari sleep dengan benar

RTX 3050 menggunakan arsitektur **Ampere (GA107)**, yang sudah didukung penuh oleh driver NVIDIA proprietary.

## Pendekatan Solusi

Untuk GPU Ampere seperti RTX 3050, ada dua pilihan driver:

| Driver | Keterangan |
|--------|------------|
| `nvidia` | Driver proprietary closed-source, paling stabil dan mature |
| `nvidia-open` | Kernel module open-source dari NVIDIA, direkomendasikan untuk arsitektur Turing ke atas |

Meskipun NVIDIA sudah merekomendasikan `nvidia-open` untuk arsitektur Turing ke atas, saya tetap memilih **nvidia (proprietary)** karena lebih proven dan stabil untuk daily driver. Driver closed-source sudah bertahun-tahun di-maintain dan edge case-nya lebih sedikit dibanding nvidia-open yang relatif masih baru.

Langkah-langkahnya:

1. Install package driver yang sesuai dengan kernel
2. Konfigurasi mkinitcpio untuk early loading module
3. Tambahkan kernel parameter di systemd-boot entry
4. Enable service untuk suspend/resume

## Implementasi Teknis

### Instalasi Package

Karena menggunakan kernel **linux-zen**, kita perlu install package driver yang sesuai:

```
$ sudo pacman -S linux-zen-headers nvidia-dkms nvidia-utils nvidia-settings
```

Penjelasan masing-masing package:

| Package | Fungsi |
|---------|--------|
| `linux-zen-headers` | Header kernel, dibutuhkan oleh DKMS untuk compile module |
| `nvidia-dkms` | Kernel module NVIDIA proprietary, otomatis compile untuk kernel yang terinstall via DKMS |
| `nvidia-utils` | Library userspace NVIDIA (libGL, libEGL, Vulkan ICD, dll) |
| `nvidia-settings` | GUI untuk konfigurasi NVIDIA |

Kenapa `nvidia-dkms` dan bukan `nvidia`? Karena kita pakai kernel **linux-zen**, bukan `linux` standar. Package `nvidia` (tanpa `-dkms`) hanya untuk kernel `linux` default. DKMS memastikan module otomatis di-compile ulang setiap kali kernel di-update.

Untuk dukungan 32-bit (misalnya untuk Steam atau Wine), install juga:

```
$ sudo pacman -S lib32-nvidia-utils
```

### Konfigurasi mkinitcpio

Agar NVIDIA module dimuat sedini mungkin saat boot (early KMS), tambahkan module NVIDIA ke mkinitcpio:

Edit `/etc/mkinitcpio.conf`:

```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

Setelah itu, rebuild initramfs:

```
$ sudo mkinitcpio -P
```

Flag `-P` akan rebuild initramfs untuk semua kernel yang terinstall, termasuk `linux-zen`.

### Pacman Hook untuk Auto-Rebuild

Satu hal yang sering terlupakan -- setiap kali kernel di-update, initramfs perlu di-rebuild agar module NVIDIA yang baru ikut masuk. Buat pacman hook agar proses ini otomatis:

Buat file `/etc/pacman.d/hooks/nvidia.hook`:

```ini
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms
Target=linux-zen

[Action]
Description=Rebuilding initramfs after NVIDIA driver update...
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux-zen) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

Hook ini memastikan initramfs di-rebuild setiap kali package `nvidia-dkms` atau `linux-zen` di-update.

### Konfigurasi Systemd-Boot Entry

Ini bagian yang spesifik untuk systemd-boot. Edit boot entry di `/boot/loader/entries/`:

```
title Arch Linux
linux /vmlinuz-linux-zen
initrd /initramfs-linux-zen.img
options root=LABEL=archlinux rw quiet acpi_backlight=native rd.udev.log_level=3 v4l2loopback preempt=full nvidia nvidia_modeset nvidia_uvm nvidia_drm
```

Penjelasan masing-masing kernel parameter:

| Parameter | Fungsi |
|-----------|--------|
| `quiet` | Menyembunyikan pesan boot verbose |
| `acpi_backlight=native` | Menggunakan driver backlight native dari ACPI -- memperbaiki kontrol brightness di laptop |
| `rd.udev.log_level=3` | Membatasi log level udev saat initramfs agar boot lebih bersih |
| `v4l2loopback` | Memuat module virtual camera -- berguna untuk OBS virtual camera atau aplikasi sejenis |
| `preempt=full` | Mengaktifkan full kernel preemption -- memprioritaskan responsivitas sistem, cocok untuk desktop |
| `nvidia` | Memuat kernel module NVIDIA utama |
| `nvidia_modeset` | Memuat module modesetting NVIDIA -- **wajib** untuk Wayland dan menghindari tearing |
| `nvidia_uvm` | Memuat module Unified Virtual Memory -- dibutuhkan untuk CUDA |
| `nvidia_drm` | Memuat module DRM NVIDIA -- mengaktifkan kernel mode setting |

Module `nvidia_modeset` dan `nvidia_drm` adalah yang paling krusial. Tanpa keduanya, Wayland compositor seperti Mutter (GNOME) atau KWin (KDE) tidak akan bisa menggunakan GPU NVIDIA.

### NVIDIA PRIME (Hybrid GPU)

Laptop dengan RTX 3050 biasanya punya konfigurasi hybrid GPU -- ada **Intel iGPU** untuk daily use dan **NVIDIA dGPU** untuk aplikasi yang butuh performa grafis tinggi. Pendekatan ini jauh lebih hemat daya dibanding menjalankan NVIDIA terus-menerus.

Konsepnya sederhana: sehari-hari pakai Intel iGPU (hemat battery, tidak panas), lalu saat butuh NVIDIA tinggal jalankan aplikasi tertentu dengan `prime-run`.

Install package yang dibutuhkan:

```
$ sudo pacman -S nvidia-prime
```

Sekarang untuk menjalankan aplikasi menggunakan GPU NVIDIA, cukup tambahkan `prime-run` di depan perintah:

```
$ prime-run glxinfo | grep "OpenGL renderer"
OpenGL renderer string: NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2
```

Tanpa `prime-run`, aplikasi akan menggunakan Intel iGPU:

```
$ glxinfo | grep "OpenGL renderer"
OpenGL renderer string: Mesa Intel(R) ...
```

Beberapa contoh penggunaan:

```
$ prime-run blender                    # 3D rendering pakai NVIDIA
$ prime-run obs                        # Recording/streaming pakai NVIDIA encoder
$ prime-run steam                      # Gaming pakai NVIDIA
$ prime-run vkcube                     # Test Vulkan di NVIDIA
```

Untuk aplikasi berbasis Electron atau Chromium yang ingin memanfaatkan hardware acceleration NVIDIA:

```
$ prime-run code                       # VS Code dengan GPU NVIDIA
$ prime-run google-chrome-stable       # Chrome dengan GPU NVIDIA
```

Keuntungan pendekatan PRIME render offload ini:

- **Battery life lebih panjang** -- NVIDIA GPU hanya aktif saat dibutuhkan
- **Temperatur lebih rendah** -- Intel iGPU jauh lebih hemat daya untuk tugas sehari-hari
- **Fleksibel** -- tidak perlu logout atau restart untuk switching GPU, cukup pilih per-aplikasi

### Enable Suspend/Resume Service

NVIDIA menyediakan systemd service untuk menangani suspend, hibernate, dan resume agar GPU bisa tidur dan bangun dengan benar:

```
$ sudo systemctl enable nvidia-suspend
$ sudo systemctl enable nvidia-resume
$ sudo systemctl enable nvidia-hibernate
```

Tanpa service ini, ada kemungkinan layar hitam atau artefak setelah laptop bangun dari sleep.

### Verifikasi

Setelah semua konfigurasi selesai, reboot:

```
$ sudo reboot
```

Setelah masuk kembali, verifikasi driver sudah aktif:

```
$ nvidia-smi
```

Output-nya akan menampilkan informasi GPU:

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.xx.xx    Driver Version: 570.xx.xx    CUDA Version: 12.x                |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3050 ...    Off | 00000000:01:00.0  On  |                  N/A |
| N/A   45C    P8              5W /  60W  |     256MiB /   4096MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

Verifikasi DRM modesetting aktif:

```
$ cat /sys/module/nvidia_drm/parameters/modeset
Y
```

Cek apakah module dimuat dengan benar:

```
$ lsmod | grep nvidia
```

Output seharusnya menampilkan `nvidia`, `nvidia_modeset`, `nvidia_uvm`, dan `nvidia_drm`.

## Tantangan yang Dihadapi

Tantangan pertama adalah memahami hubungan antara package driver dan kernel. Di Arch, setiap varian kernel punya package NVIDIA tersendiri. Kalau pakai kernel `linux` standar, cukup install `nvidia`. Tapi untuk kernel lain seperti `linux-zen`, `linux-lts`, atau `linux-cachyos`, harus pakai versi `-dkms` agar module di-compile khusus untuk kernel tersebut. Salah install package bisa berujung ke layar hitam saat boot.

Tantangan kedua adalah soal **systemd-boot**. Karena tidak ada tool seperti `update-grub` yang otomatis mengurus kernel parameter, kita harus manual menambahkan parameter NVIDIA di boot entry. Ini sebenarnya lebih transparan -- kita tahu persis apa yang dikonfigurasi -- tapi juga berarti kita yang bertanggung jawab memastikan parameter-nya benar dan tidak ada yang terlewat.

Satu hal yang perlu diingat -- jika ingin menggunakan `nvidia-dkms`, pastikan `linux-zen-headers` sudah terinstall **sebelum** install driver. Tanpa header, DKMS tidak bisa compile module dan driver tidak akan tersedia saat boot.

## Insight dan Pembelajaran

Setelah menjalani proses setup ini, beberapa insight:

- **`nvidia-dkms` adalah pilihan terbaik untuk kernel non-standar** -- satu package yang otomatis compile untuk semua kernel yang terinstall. Tidak perlu khawatir soal kompatibilitas versi.
- **Early loading via mkinitcpio itu penting** -- tanpa ini, module NVIDIA dimuat terlambat dan bisa menyebabkan flickering atau resolusi salah saat boot. Dengan early loading, transisi dari boot ke display manager jauh lebih mulus.
- **Pacman hook menyelamatkan dari lupa rebuild** -- tanpa hook, setiap update kernel harus manual rebuild initramfs. Hook mengotomasi proses ini dan mencegah situasi boot gagal setelah update.
- **Suspend/resume service sering terlupakan** -- banyak yang setup driver NVIDIA tapi lupa enable service untuk power management. Akibatnya, laptop mengalami layar hitam setelah bangun dari sleep.
- **PRIME render offload adalah sweet spot untuk laptop** -- daily use pakai Intel iGPU yang hemat daya, lalu `prime-run` saat butuh performa NVIDIA. Tidak perlu logout atau restart untuk switching.
- **Systemd-boot sebenarnya lebih straightforward dari GRUB** -- meskipun manual, konfigurasinya minimal dan mudah dipahami. Satu file boot entry berisi semua yang perlu diketahui.

## Penutup

Setup driver NVIDIA di Archlinux memang butuh beberapa langkah manual, tapi hasilnya worth it -- performa GPU optimal, Wayland berjalan lancar, dan suspend/resume yang reliable. Dengan NVIDIA PRIME, kita bisa mendapatkan yang terbaik dari dua dunia: hemat daya pakai Intel iGPU untuk sehari-hari, dan performa penuh NVIDIA saat dibutuhkan lewat `prime-run`. Kuncinya ada di empat hal: install package yang sesuai kernel (`nvidia-dkms` untuk kernel non-standar), konfigurasi early loading di mkinitcpio, menambahkan kernel parameter `nvidia_drm.modeset=1` di boot entry systemd-boot, dan setup NVIDIA PRIME untuk hybrid GPU. Dengan pacman hook yang sudah di-setup, proses update kernel ke depannya juga tidak perlu khawatir karena initramfs otomatis di-rebuild.

## Referensi

- [Arch Wiki - NVIDIA](https://wiki.archlinux.org/title/NVIDIA) -- Diakses pada 2026-04-05
- [Arch Wiki - NVIDIA/Tips and tricks](https://wiki.archlinux.org/title/NVIDIA/Tips_and_tricks) -- Diakses pada 2026-04-05
- [Arch Wiki - Systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) -- Diakses pada 2026-04-05
- [NVIDIA Open GPU Kernel Modules](https://github.com/NVIDIA/open-gpu-kernel-modules) -- Diakses pada 2026-04-05
