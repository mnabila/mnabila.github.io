+++
draft = false
date = '2026-04-07'
title = 'Setup Systemd Boot Di Archlinux Dengan Kernel Linux Zen'
type = 'blog'
description = 'Panduan setup systemd-boot sebagai bootloader di Archlinux menggunakan kernel linux-zen, mulai dari instalasi hingga konfigurasi boot entry.'
image = ''
tags = ['systemd-boot', 'bootloader', 'linux-zen', 'archlinux', 'uefi']
+++

## Latar Belakang

Bootloader adalah komponen pertama yang jalan saat komputer dinyalakan tugasnya memuat kernel ke memori dan menyerahkan kontrol ke sistem operasi. Di Archlinux, kebanyakan orang langsung pakai GRUB karena familiar. Tapi sebenarnya ada alternatif yang jauh lebih simpel: **systemd-boot**. Bootloader ini sudah bundled di dalam `systemd`, konfigurasinya plain text, dan untuk kebutuhan single OS seperti setup saya, ini lebih dari cukup.

## Permasalahan

Beberapa alasan kenapa saya beralih dari GRUB ke systemd-boot:

- **GRUB terlalu complex untuk single OS** -- fitur seperti theming, submenu, dan scripting di `grub.cfg` itu tidak terpakai kalau cuma boot satu OS
- **Config auto-generated susah dibaca** -- setiap perubahan kernel parameter harus lewat `grub-mkconfig` yang generate file config ratusan baris yang susah di-debug manual
- **Overhead yang tidak perlu** -- GRUB punya layer abstraksi tambahan yang sebenarnya tidak dibutuhkan untuk setup sederhana
- **Maintenance ribet** -- setiap update kernel atau perubahan parameter, harus regenerate config dan berharap hasilnya benar

Yang dibutuhkan adalah bootloader yang simpel, cepat, dan konfigurasinya bisa diedit langsung tanpa tool perantara.

## Pendekatan Solusi

Ada beberapa pilihan bootloader untuk sistem UEFI:

| Bootloader | Kelebihan | Kekurangan |
|------------|-----------|------------|
| **GRUB** | Fitur lengkap, support BIOS dan UEFI, theming | Complex, config auto-generated susah dibaca |
| **systemd-boot** | Simpel, cepat, sudah bundled di systemd | Hanya UEFI, tidak ada theming |
| **rEFInd** | Auto-detect OS, tampilan bagus | Perlu install terpisah, konfigurasi lebih rumit |
| **EFISTUB** | Paling minimal, boot langsung dari firmware | Tidak ada menu, manage via efibootmgr |

Saya memilih **systemd-boot** karena:

1. **Sudah ada di sistem** -- tidak perlu install package apapun, sudah bawaan dari `systemd`
2. **Konfigurasi readable** -- file config berupa plain text yang mudah diedit dan dipahami
3. **Update otomatis** -- bisa di-setup untuk auto-update lewat systemd service
4. **Cukup untuk kebutuhan** -- boot satu OS dengan opsi fallback, itu saja yang dibutuhkan

> **Note:** systemd-boot hanya bisa digunakan pada sistem **UEFI**. Kalau masih pakai BIOS/Legacy boot, GRUB tetap jadi pilihan utama.

## Implementasi Teknis

### Cek Mode UEFI

Pastikan sistem sudah boot dalam mode UEFI:

```
$ ls /sys/firmware/efi/efivars
```

Kalau direktori ini ada dan berisi file, berarti sistem sudah dalam mode UEFI. Tanpa ini, systemd-boot tidak bisa digunakan.

### Identifikasi ESP (EFI System Partition)

ESP adalah partisi khusus berformat FAT32 yang menyimpan bootloader dan kernel. Cek apakah sudah ada:

```
$ fdisk -l | grep "EFI System"
```

Biasanya ESP sudah ada kalau install Archlinux dengan mode UEFI. Kalau belum ada, buat partisi baru dengan tipe `EFI System` (kode `ef00` di gdisk).

Saya mount ESP langsung di `/boot` supaya kernel dan initramfs otomatis tersimpan di ESP saat install atau update:

```
$ mount /dev/sdXY /boot
```

Pastikan entry ini juga ada di `/etc/fstab`:

```
/dev/sdXY  /boot  vfat  defaults  0 2
```

Ganti `/dev/sdXY` dengan partisi ESP yang sebenarnya. Bisa juga pakai `UUID=xxxx` untuk identifikasi yang lebih reliable.

### Install Kernel linux-zen

Kalau belum install kernel linux-zen, install beserta headers-nya:

```
$ sudo pacman -S linux-zen linux-zen-headers
```

Penjelasan masing-masing package:

| Package | Fungsi |
|---------|--------|
| `linux-zen` | Kernel yang sudah di-patch untuk responsiveness lebih baik di desktop -- include scheduling optimization dan lower latency |
| `linux-zen-headers` | Header kernel, dibutuhkan untuk compile module pihak ketiga (misalnya NVIDIA DKMS) |

Jangan lupa install microcode sesuai CPU:

```
# Untuk Intel
$ sudo pacman -S intel-ucode

# Untuk AMD
$ sudo pacman -S amd-ucode
```

### Instalasi systemd-boot

Install bootloader ke ESP:

```
$ sudo bootctl install
```

Command ini akan:

- Copy file EFI binary ke `esp/EFI/systemd/systemd-bootx64.efi`
- Copy fallback binary ke `esp/EFI/BOOT/BOOTX64.EFI`
- Membuat entry boot UEFI bernama "Linux Boot Manager"
- Set entry tersebut sebagai boot pertama di urutan boot UEFI

Verifikasi instalasi:

```
$ bootctl status
```

Output akan menampilkan informasi firmware, secure boot status, dan boot entry yang terdeteksi.

### Konfigurasi Loader

Buat atau edit file `/boot/loader/loader.conf`:

```
default arch-zen.conf
timeout 3
console-mode max
editor  no
```

Penjelasan tiap parameter:

| Parameter | Nilai | Deskripsi |
|-----------|-------|-----------|
| `default` | `arch-zen.conf` | Entry yang di-boot secara default |
| `timeout` | `3` | Tampilkan menu selama 3 detik sebelum auto-boot |
| `console-mode` | `max` | Gunakan resolusi konsol tertinggi yang tersedia |
| `editor` | `no` | Nonaktifkan editing kernel parameter di menu boot -- mencegah siapa saja dengan akses fisik mengubah parameter boot |

Set `timeout 0` kalau mau langsung boot tanpa menu. Menu tetap bisa diakses dengan menekan **Space** saat boot. Bisa juga set `default @saved` kalau ingin systemd-boot mengingat entry terakhir yang di-boot.

Template loader.conf juga tersedia di `/usr/share/systemd/bootctl/loader.conf` sebagai referensi.

### Konfigurasi Boot Entry

Buat file entry untuk kernel linux-zen di `/boot/loader/entries/arch-zen.conf`:

```
title   Arch Linux (zen)
linux   /vmlinuz-linux-zen
initrd  /amd-ucode.img
initrd  /initramfs-linux-zen.img
options root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rw
```

Penjelasan tiap baris:

| Field | Deskripsi |
|-------|-----------|
| `title` | Teks yang ditampilkan di menu boot |
| `linux` | Path ke kernel image, relatif terhadap root ESP |
| `initrd` (pertama) | Microcode CPU -- **harus sebelum** initramfs |
| `initrd` (kedua) | Initramfs image |
| `options` | Kernel parameter -- minimal butuh `root=` untuk menunjuk partisi root |

Untuk mendapatkan UUID partisi root:

```
$ blkid /dev/sdXY
```

Ganti `amd-ucode.img` dengan `intel-ucode.img` kalau pakai Intel. Path file (`/vmlinuz-linux-zen`, `/initramfs-linux-zen.img`) itu relatif terhadap root ESP -- jadi kalau ESP di-mount di `/boot`, file `/boot/vmlinuz-linux-zen` ditulis sebagai `/vmlinuz-linux-zen` di entry config.

Template entry juga tersedia di `/usr/share/systemd/bootctl/arch.conf` sebagai referensi.

### Buat Entry Fallback

Buat juga entry fallback di `/boot/loader/entries/arch-zen-fallback.conf`:

```
title   Arch Linux (zen, fallback)
linux   /vmlinuz-linux-zen
initrd  /amd-ucode.img
initrd  /initramfs-linux-zen-fallback.img
options root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rw
```

Entry fallback menggunakan initramfs yang berisi **semua** module -- berguna kalau initramfs default gagal boot karena module yang hilang.

### Verifikasi Konfigurasi

Setelah semua selesai, jalankan `bootctl` tanpa argument untuk validasi:

```
$ bootctl
```

Atau list semua entry yang terdeteksi:

```
$ bootctl list
```

Pastikan entry `arch-zen.conf` muncul dan tidak ada error parsing. Kalau ada kesalahan di config, `bootctl` akan menampilkan warning yang cukup jelas.

### Setup Auto-Update

Supaya systemd-boot otomatis di-update saat package `systemd` di-upgrade, enable service berikut:

```
$ sudo systemctl enable systemd-boot-update.service
```

Service ini menjalankan `bootctl update` setiap boot, memastikan EFI binary selalu sinkron dengan versi systemd yang terinstall. Tanpa ini, ada kemungkinan EFI binary tertinggal versinya setelah update systemd.

## Tantangan yang Dihadapi

Tantangan pertama adalah soal **path relatif di boot entry**. Semua path di file entry config itu relatif terhadap root ESP, bukan root filesystem. Ini sering bikin bingung di awal -- kalau ESP di-mount di `/boot`, maka file `/boot/vmlinuz-linux-zen` ditulis sebagai `/vmlinuz-linux-zen`, bukan `/boot/vmlinuz-linux-zen`. Salah tulis path berarti kernel tidak ditemukan dan boot gagal.

Tantangan kedua adalah **UUID yang harus tepat**. Tidak seperti GRUB yang punya `grub-mkconfig` untuk auto-detect partisi, di systemd-boot kita harus manual memasukkan UUID partisi root. Salah ketik satu karakter saja berarti kernel panic saat boot. Selalu double-check dengan `blkid` dan pastikan UUID yang dimasukkan benar.

Satu hal lagi -- **urutan `initrd` itu penting**. Microcode harus di-load sebelum initramfs. Kalau urutannya terbalik, microcode tidak akan ter-apply meskipun tidak menyebabkan boot failure. Ini silent issue yang baru ketahuan kalau cek manual.

## Insight dan Pembelajaran

Setelah migrasi ke systemd-boot, beberapa insight yang didapat:

- **Simplicity is a feature** -- tidak semua hal butuh tool yang complex. Untuk single-boot system, systemd-boot jauh lebih straightforward dibanding GRUB. Satu file entry berisi semua yang perlu diketahui.
- **Plain text config itu powerful** -- bisa di-track di dotfiles, mudah di-backup, dan gampang di-debug kalau ada masalah. Tidak ada auto-generated config yang misterius.
- **Keyboard shortcut di menu berguna untuk troubleshooting** -- tekan **d** untuk set default entry, **t/T** untuk adjust timeout, dan **e** untuk edit kernel parameter (kalau editor diaktifkan).
- **`editor no` wajib di-set untuk keamanan** -- secara default, siapa saja dengan akses fisik bisa edit kernel parameter di menu boot. Ini bisa dieksploitasi untuk bypass security, jadi selalu nonaktifkan.
- **Auto-detect Windows sudah built-in** -- meskipun tidak se-canggih GRUB yang punya `os-prober`, systemd-boot otomatis mendeteksi Windows Boot Manager kalau file `EFI/Microsoft/Boot/Bootmgfw.efi` ada di ESP.
- **`systemctl reboot --firmware-setup`** -- command ini berguna untuk reboot langsung ke UEFI firmware setup tanpa harus spam tombol F2/Del saat boot.

## Penutup

Migrasi dari GRUB ke systemd-boot itu straightforward dan hasilnya worth it -- terutama untuk setup single OS. Konfigurasinya minimal (satu file loader.conf dan satu file entry per kernel), maintenance-nya hampir zero dengan `systemd-boot-update.service`, dan sudah terintegrasi dengan systemd ecosystem. Kuncinya ada di tiga hal: mount ESP di `/boot` agar kernel otomatis tersimpan di tempat yang benar, pastikan UUID dan path di boot entry sudah tepat, dan set `editor no` untuk keamanan.

## Referensi

- [Arch Wiki - Systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) -- Diakses pada 2026-04-07
- [The Boot Loader Specification - systemd.io](https://systemd.io/BOOT/) -- Diakses pada 2026-04-07
- [Arch Wiki - Kernel](https://wiki.archlinux.org/title/Kernel#Officially_supported_kernels) -- Diakses pada 2026-04-07
