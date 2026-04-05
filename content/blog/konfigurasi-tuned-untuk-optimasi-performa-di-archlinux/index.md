+++
draft = false
date = '2026-04-05'
title = 'Konfigurasi Tuned Untuk Optimasi Performa Di Archlinux'
type = 'blog'
description = 'Cara menggunakan tuned untuk optimasi performa sistem secara otomatis di Archlinux berdasarkan profil workload yang sedang berjalan.'
image = ''
tags = ['tuned', 'performance', 'power-management', 'archlinux']
+++

## Latar Belakang

Salah satu hal yang sering diabaikan di Linux desktop adalah **tuning performa sistem**. Kebanyakan dari kita langsung pakai default setting setelah install -- CPU governor di `schedutil`, I/O scheduler default, dan parameter kernel apa adanya. Padahal, kebutuhan setiap workload itu berbeda. Saat ngoding, kita butuh responsivitas. Saat build project, kita butuh throughput maksimal. Saat di battery, kita butuh efisiensi daya.

Di sinilah **tuned** masuk. Tool ini awalnya dikembangkan oleh Red Hat untuk RHEL, tapi karena open source, kita bisa menggunakannya di Archlinux juga. Tuned adalah daemon yang secara dinamis mengoptimasi parameter sistem berdasarkan profil yang kita pilih -- tanpa perlu tweaking manual satu per satu.

## Permasalahan

Melakukan tuning performa secara manual itu melelahkan dan error-prone. Bayangkan harus mengatur semua ini setiap kali kebutuhan berubah:

- CPU governor (`performance`, `powersave`, `schedutil`)
- I/O scheduler (`mq-deadline`, `bfq`, `none`)
- Kernel parameter seperti `vm.swappiness`, `vm.dirty_ratio`
- Power management untuk disk dan network adapter
- SATA link power management

Tanpa tool khusus, kita harus menulis script sendiri atau mengubah parameter satu per satu via `sysctl`, `cpupower`, dan file di `/sys/`. Selain ribet, mudah lupa mengembalikan setting saat berpindah skenario penggunaan.

## Pendekatan Solusi

Tuned menyelesaikan masalah ini dengan pendekatan **profile-based tuning**. Konsepnya sederhana:

1. **Pilih profil** yang sesuai dengan workload
2. **Tuned daemon** menerapkan semua parameter yang didefinisikan di profil tersebut
3. **Switching** antar profil bisa dilakukan kapan saja dengan satu perintah

Beberapa profil bawaan yang tersedia:

| Profil | Kegunaan |
|--------|----------|
| `balanced` | Keseimbangan antara performa dan power saving -- cocok untuk daily use |
| `throughput-performance` | Maksimalkan throughput, cocok untuk server atau saat build project |
| `latency-performance` | Minimalkan latency, cocok untuk real-time workload |
| `powersave` | Hemat daya semaksimal mungkin -- ideal untuk laptop saat di battery |
| `desktop` | Optimasi untuk penggunaan desktop interaktif |
| `virtual-guest` | Optimasi untuk VM guest |
| `virtual-host` | Optimasi untuk VM host |
| `network-latency` | Minimalkan network latency |
| `network-throughput` | Maksimalkan network throughput |

Saya memilih tuned karena tidak perlu reinvent the wheel -- profil-profil ini sudah dioptimasi oleh engineer Red Hat dan komunitas, jadi tinggal pakai.

## Implementasi Teknis

### Instalasi

Tuned tersedia di repository resmi Archlinux (extra):

```
$ sudo pacman -S tuned
```

Setelah terinstall, aktifkan dan jalankan daemon-nya:

```
$ sudo systemctl enable --now tuned
```

Verifikasi bahwa service berjalan:

```
$ sudo systemctl status tuned
```

### Melihat Profil yang Tersedia

Untuk melihat semua profil yang tersedia di sistem:

```
$ tuned-adm list
```

Output-nya kurang lebih seperti ini:

```
Available profiles:
- balanced                    - General non-specialized tuned profile
- desktop                     - Optimize for the desktop use-case
- latency-performance         - Optimize for deterministic performance at the cost of increased power consumption
- network-latency             - Optimize for deterministic performance at the cost of increased power consumption, focused on low latency network performance
- network-throughput          - Optimize for streaming network throughput, generally only necessary on older CPUs or 40G+ networks
- powersave                   - Optimize for low power consumption
- throughput-performance      - Broadly applicable tuning that provides excellent performance across a variety of common server workloads
- virtual-guest               - Optimize for running inside a virtual guest
- virtual-host                - Optimize for running KVM guests
Current active profile: balanced
```

### Mengecek Profil Aktif

```
$ tuned-adm active
Current active profile: balanced
```

### Mengganti Profil

Misalnya ingin beralih ke profil `throughput-performance` saat mau build project:

```
$ sudo tuned-adm profile throughput-performance
```

Atau saat mau hemat battery di laptop:

```
$ sudo tuned-adm profile powersave
```

### Rekomendasi Profil Otomatis

Tuned bisa merekomendasikan profil yang cocok untuk hardware kita:

```
$ tuned-adm recommend
```

Biasanya untuk laptop akan merekomendasikan `balanced`, dan untuk desktop atau server akan merekomendasikan `throughput-performance`.

### Verifikasi Profil

Untuk memastikan semua parameter dari profil aktif sudah diterapkan dengan benar:

```
$ sudo tuned-adm verify
Verification succeeded, current system settings match the preset profile.
```

### Membuat Profil Custom

Jika profil bawaan kurang sesuai, kita bisa membuat profil sendiri. Buat direktori dan file konfigurasi:

```
$ sudo mkdir /etc/tuned/balanced-no-turbo
```

Buat file `/etc/tuned/balanced-no-turbo/tuned.conf`:

```ini
[main]
summary=Balanced profile with turbo boost disabled
include=balanced

[cpu]
no_turbo=1
```

Profil di atas menggunakan `balanced` sebagai base, lalu menonaktifkan turbo boost lewat parameter `no_turbo=1`. Kenapa mau mematikan turbo? Beberapa alasannya:

- **Temperatur lebih terkontrol** -- turbo boost mendorong clock speed di atas base frequency, yang berarti panas lebih tinggi. Tanpa turbo, suhu CPU lebih stabil dan fan tidak berisik.
- **Konsumsi daya lebih rendah** -- cocok untuk laptop yang ingin battery life lebih panjang tanpa harus masuk full `powersave`.
- **Performa tetap konsisten** -- turbo boost bisa menyebabkan thermal throttling di workload yang sustained, sehingga performa naik-turun. Tanpa turbo, performa lebih predictable meskipun sedikit lebih rendah.

Directive `include` sangat berguna supaya tidak perlu menulis ulang semua parameter dari nol -- cukup override yang ingin diubah saja.

Aktifkan profil custom:

```
$ sudo tuned-adm profile balanced-no-turbo
```

Verifikasi turbo boost sudah nonaktif:

```
$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
1
```

## Tantangan yang Dihadapi

Beberapa profil bawaan tuned dirancang untuk RHEL/CentOS, jadi ada parameter yang mungkin tidak relevan atau tidak tersedia di kernel Archlinux yang lebih baru. Misalnya, beberapa setting terkait `tuned-adm` plugin tertentu mungkin membutuhkan package tambahan yang belum terinstall. Jalankan `tuned-adm verify` setelah mengganti profil untuk memastikan semua setting berhasil diterapkan.

Satu hal lagi -- tuned bisa bentrok dengan tool power management lain seperti `tlp` atau `power-profiles-daemon`. Pastikan hanya satu tool yang aktif untuk menghindari konflik. Jika sebelumnya menggunakan TLP:

```
$ sudo systemctl disable --now tlp
```

## Insight dan Pembelajaran

Setelah menggunakan tuned beberapa waktu, beberapa insight yang didapat:

- **Profil `balanced` sudah cukup bagus** untuk penggunaan sehari-hari. Tidak perlu langsung jump ke `throughput-performance` kecuali memang butuh.
- **Custom profil dengan `include`** adalah fitur killer -- bisa extend profil yang sudah ada tanpa harus menulis semua parameter dari awal.
- **Switching profil sangat cepat** -- tidak perlu restart service, langsung apply. Ini berguna saat mau build project besar lalu kembali ke mode hemat daya.
- **`tuned-adm verify`** adalah command yang sering terlupakan tapi penting -- memastikan tidak ada parameter yang gagal diterapkan karena konflik atau dependency yang kurang.
Dibanding melakukan tuning manual, tuned memberikan pendekatan yang lebih terstruktur dan reproducible. Profil bisa di-backup, di-share, dan di-version control.

## Penutup

Tuned adalah tool yang simpel tapi powerful untuk optimasi performa sistem di Archlinux. Dengan profil bawaan yang sudah dioptimasi dan kemampuan membuat profil custom, kita bisa mengatur performa sistem sesuai kebutuhan tanpa harus menjadi kernel tuning expert. Cukup satu perintah `tuned-adm profile` untuk beralih antara mode hemat daya dan full performance. Untuk yang sering berpindah antara kerja di battery dan di charger, atau antara ngoding ringan dan heavy build, tuned bisa jadi game changer.

## Referensi

- [Red Hat - Tuned Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/power_management_guide/tuned) -- Diakses pada 2026-04-05
- [Arch Wiki - Power Management](https://wiki.archlinux.org/title/Power_management) -- Diakses pada 2026-04-05
- [Tuned Project GitHub](https://github.com/redhat-performance/tuned) -- Diakses pada 2026-04-05
