+++
draft = false
date = '2026-04-27'
title = 'Mengatasi LSP Neovim yang Tidak Mau Mati Setelah Buffer Ditutup'
type = 'blog'
description = 'Cara menghentikan proses LSP secara otomatis di Neovim ketika tidak ada buffer aktif yang masih terhubung ke client.'
image = ''
tags = ['neovim', 'lsp', 'archlinux', 'lua']
+++

## Latar Belakang

Sebagai pengguna **Neovim** di Arch Linux, LSP adalah fitur yang hampir tidak bisa dipisahkan dari workflow coding sehari-hari. Autocompletion, diagnostics, go-to-definition -- semua bergantung pada LSP client yang berjalan di background.

Masalahnya, saya mulai sadar ada sesuatu yang tidak beres. Setiap kali mengecek proses yang berjalan lewat `htop`, saya sering menemukan proses LSP server seperti `gopls`, `lua-language-server`, atau `typescript-language-server` yang masih aktif -- padahal buffer yang bersangkutan sudah lama ditutup. Dalam beberapa kasus, proses ini bahkan tetap hidup setelah Neovim ditutup.

## Permasalahan

- **Proses LSP tidak berhenti setelah buffer ditutup** -- ketika semua buffer yang menggunakan LSP client tertentu sudah ditutup, proses server-nya tetap berjalan di background tanpa alasan yang jelas.
- **Proses bertahan setelah Neovim keluar** -- dalam beberapa kasus, proses LSP masih aktif bahkan setelah Neovim sendiri sudah ditutup. Proses ini menjadi process zombie yang terus mengonsumsi resource.
- **Konsumsi CPU dan memory yang tidak perlu** -- proses LSP yang idle tapi tetap berjalan memakan resource sistem padahal lspnya sudah tidak digunakan lagi sehingga membuat boros CPU dan memory (ya walaupun memory saya 32GB, saya tetep pelit kalau urusan resource).

## Pendekatan Solusi

Ada beberapa cara untuk mengatasi proses LSP yang menggantung:

| Pendekatan                                   | Kelebihan                                 | Kekurangan                                         |
| -------------------------------------------- | ----------------------------------------- | -------------------------------------------------- |
| Kill manual via terminal                     | Langsung efektif, tidak perlu konfigurasi | Harus dilakukan setiap kali, mudah lupa            |
| Script `autocmd VimLeavePre`                 | Otomatis saat keluar Neovim               | Tidak menangani kasus buffer ditutup satu per satu |
| `autocmd LspDetach` dengan pengecekan buffer | Otomatis, granular per client, real-time  | Perlu memahami lifecycle LSP dan event system      |

Saya memilih pendekatan ketiga -- menggunakan event `LspDetach` yang di-trigger setiap kali sebuah buffer terlepas dari LSP client. Dengan pendekatan ini, pengecekan terjadi secara real-time setiap kali buffer ditutup. Jika client sudah tidak memiliki buffer aktif yang terhubung, client langsung dihentikan.

Solusi ini saya dapatkan dari komunitas **Telegram Vim Indonesia** -- dibagikan oleh [@baddmenn](https://t.me/VimID/1/53356) pada bulan Maret 2025.

## Implementasi Teknis

### Membuat Augroup

Langkah pertama, buat augroup untuk mengelompokkan autocmd yang akan dibuat. Ini penting agar autocmd bisa di-clear dan tidak duplikat saat config di-reload.

```lua
local group = vim.api.nvim_create_augroup("LspAutoDetach", { clear = true })
```

Kode di atas membuat augroup bernama `LspAutoDetach` dengan opsi `clear = true` -- artinya setiap kali config di-source ulang, autocmd lama di group ini akan dihapus terlebih dahulu.

### Menambahkan Autocmd LspDetach

Selanjutnya, tambahkan autocmd yang akan dijalankan setiap kali event `LspDetach` terjadi.

```lua
vim.api.nvim_create_autocmd("LspDetach", {
  group = group,
  callback = function(ev)
    local client = vim.lsp.get_client_by_id(ev.data.client_id)
    if not client or not client.attached_buffers then
      return
    end

    for buf_id in pairs(client.attached_buffers) do
      if buf_id ~= ev.buf then
        return
      end
    end

    client:stop()
  end,
  desc = "Auto Detach LSP",
})
```

Kode ini melakukan beberapa hal:

1. **Mendapatkan client** -- `vim.lsp.get_client_by_id()` mengambil referensi LSP client berdasarkan ID yang ada di event data.
2. **Validasi** -- jika client sudah tidak ada atau tidak punya `attached_buffers`, fungsi langsung return tanpa melakukan apa-apa.
3. **Pengecekan buffer aktif** -- iterasi semua buffer yang masih terhubung ke client. Jika ditemukan buffer lain selain buffer yang sedang di-detach (`ev.buf`), berarti client masih dibutuhkan -- fungsi return.
4. **Stop client** -- jika tidak ada buffer lain yang menggunakan client tersebut, `client:stop()` dipanggil untuk menghentikan proses LSP server.

### Kode Lengkap

Berikut konfigurasi lengkap yang bisa langsung ditambahkan ke file konfigurasi Neovim.

```lua
local group = vim.api.nvim_create_augroup("LspAutoDetach", { clear = true })

vim.api.nvim_create_autocmd("LspDetach", {
  group = group,
  callback = function(ev)
    local client = vim.lsp.get_client_by_id(ev.data.client_id)
    if not client or not client.attached_buffers then
      return
    end

    for buf_id in pairs(client.attached_buffers) do
      if buf_id ~= ev.buf then
        return
      end
    end

    client:stop()
  end,
  desc = "Auto Detach LSP",
})
```

Kode ini bisa ditempatkan di file konfigurasi LSP, misalnya di `~/.config/nvim/lua/config/lsp.lua` atau langsung di `init.lua` -- tergantung bagaimana struktur konfigurasi Neovim yang digunakan.

### Verifikasi

Untuk memastikan autocmd sudah terdaftar, jalankan perintah berikut di Neovim.

```vim
:autocmd LspAutoDetach
```

Output yang diharapkan akan menampilkan autocmd `LspDetach` yang sudah didaftarkan di group `LspAutoDetach`.

Untuk memverifikasi bahwa proses LSP benar-benar berhenti setelah buffer ditutup, buka file dengan LSP aktif, lalu tutup buffer-nya. Cek proses yang berjalan dari terminal.

```
$ ps aux | grep language-server
```

Jika tidak ada proses LSP yang tersisa setelah semua buffer ditutup, berarti konfigurasi sudah bekerja dengan benar.

## Tantangan yang Dihadapi

Tantangan utama dalam menyelesaikan masalah ini bukan pada kode solusinya -- yang ternyata cukup ringkas -- tapi pada proses menemukan solusi yang tepat. Dokumentasi Neovim sangat luas dan detail, mencakup ribuan fungsi API, event, dan opsi konfigurasi. Mencari tahu event mana yang tepat untuk kasus ini membutuhkan pemahaman tentang lifecycle LSP client di Neovim, mulai dari `LspAttach`, `LspDetach`, hingga mekanisme internal `attached_buffers`.

Yang membuat proses ini semakin membingungkan adalah perbedaan perilaku antar versi Neovim. Beberapa event dan API baru ditambahkan di versi-versi terbaru, sementara contoh-contoh yang ditemukan di internet belum tentu relevan dengan versi yang sedang digunakan. Ditambah lagi, informasi tentang kasus spesifik ini -- LSP process yang menggantung -- tidak banyak dibahas secara eksplisit di dokumentasi resmi.

Pada akhirnya, solusi justru datang dari komunitas. Diskusi di grup Telegram Vim Indonesia mempersingkat waktu pencarian yang mungkin bisa memakan waktu berjam-jam menjadi hanya beberapa menit. Ini mengingatkan bahwa terkadang komunitas adalah dokumentasi terbaik.

## Insight dan Pembelajaran

- **Event `LspDetach` adalah kunci** -- event ini di-trigger setiap kali buffer terlepas dari LSP client, menjadikannya titik yang tepat untuk melakukan pengecekan dan cleanup servicenya.
- **`attached_buffers` menyimpan state real-time** -- property ini pada LSP client object menyimpan daftar buffer yang masih terhubung, sehingga bisa digunakan untuk menentukan apakah lsp masih dibutuhkan atau tidak.
- **Augroup dengan `clear = true` mencegah duplikasi** -- tanpa opsi ini, setiap kali config di-reload akan menambah autocmd baru yang redundan.
- **Komunitas developer lokal sangat berharga** -- solusi teknis tidak selalu datang dari dokumentasi resmi. Grup seperti Vim Indonesia di Telegram bisa menjadi sumber insight yang sangat praktis dan relevan.
- **Kode yang ringkas belum tentu mudah ditemukan** -- solusi akhirnya hanya belasan baris Lua, tapi proses menemukan kombinasi event, API, dan logika yang tepat membutuhkan pemahaman yang cukup dalam tentang internal Neovim.

## Penutup

Proses LSP yang menggantung setelah buffer ditutup adalah masalah kecil yang bisa berdampak signifikan pada konsumsi resource sistem. Dengan memanfaatkan event `LspDetach` dan pengecekan `attached_buffers`, masalah ini bisa diatasi secara otomatis tanpa perlu intervensi manual. Solusi ini menjadi bagian dari konfigurasi Neovim yang saya gunakan sehari-hari di Arch Linux.

Special thanks untuk [@baddmenn](https://t.me/VimID/1/53356) dari komunitas Telegram Vim Indonesia yang sudah membagikan solusi ini.

## Referensi

- [Neovim Autocmd Documentation](https://neovim.io/doc/user/autocmd/#autocmd) -- Diakses pada 2026-04-27
- [Neovim LSP Documentation](https://neovim.io/doc/user/lsp/#lsp) -- Diakses pada 2026-04-27
- [@baddmenn - Telegram Vim Indonesia](https://t.me/VimID/1/53356) -- Diakses pada 2026-04-27
