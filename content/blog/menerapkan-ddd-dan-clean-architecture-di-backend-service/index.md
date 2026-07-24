+++
draft = false
date = '2026-07-04'
title = 'Menerapkan DDD dan Clean Architecture di Backend Service'
type = 'blog'
description = 'Pengalaman menerapkan Domain-Driven Design dan Clean Architecture di backend service, dari organisasi kode sampai strategi modularitas yang siap dimigrasi ke microservice'
image = ''
tags = ['architecture', 'ddd', 'clean-architecture', 'golang', 'software-design']
+++

## Latar Belakang

Hampir semua backend service yang saya tulis menerapkan **Domain-Driven Design** dan **Clean Architecture**. Dua prinsip ini jadi fondasi default: DDD untuk memodelkan domain bisnis, Clean Architecture untuk menjaga separation of concerns antar layer. Kombinasinya bikin kode terstruktur, testable, dan relatif mudah di-maintain.

Tapi setelah beberapa project jalan cukup lama, saya sadar bahwa DDD dan Clean Architecture itu soal prinsip, bukan soal struktur folder. Keduanya bilang "pisahkan domain dari infrastructure", tapi tidak menjawab pertanyaan praktis: file-file ini taruh di mana? Bagaimana cara mengorganisasi kode supaya modular dan siap dipisah jadi microservice kalau monolith-nya sudah terlalu besar?

Di sinilah pertanyaan soal code organization muncul. Kode dikelompokkan berdasarkan layer teknis seperti `handler/`, `service/`, `repository/`? Atau berdasarkan fitur bisnis seperti `order/`, `payment/`, `inventory/`? Dua-duanya bisa dikombinasikan dengan DDD dan Clean Architecture, tapi implikasinya terhadap modularitas dan kesiapan microservice sangat berbeda. Dua pendekatan ini dikenal sebagai **Horizontal Slice Architecture** dan **Vertical Slice Architecture**.

## Permasalahan

Beberapa masalah yang sering saya hadapi saat menerapkan DDD dan Clean Architecture di monolith:

- **Modularity yang semu**: kode sudah dipisah per layer, tapi antar fitur masih saling import bebas. Boundary antar bounded context tidak ter-enforce, jadi modularitas cuma di permukaan
- **Sulit dimigrasi ke microservice**: ketika monolith sudah terlalu besar dan perlu dipecah, ternyata coupling antar fitur sudah terlalu dalam. Memisahkan satu fitur jadi service sendiri butuh refactoring besar-besaran
- **Navigasi sulit di project besar**: satu fitur tersebar di banyak folder (`handler/order.go`, `service/order.go`, `repository/order.go`, `model/order.go`), mengerjakan satu fitur butuh lompat-lompat antar folder yang jauh
- **Coupling antar fitur tidak terlihat jelas**: semua service ada di satu package, satu service bisa dengan mudah import service lain tanpa ada boundary yang jelas. Dependency antar fitur jadi implicit dan makin kusut seiring waktu
- **Tidak jelas kapan harus beralih strategi**: project kecil mungkin tidak perlu vertical slice karena overhead-nya tidak sebanding. Tapi di titik mana monolith dianggap cukup besar untuk dipecah?

## Apa Itu Horizontal dan Vertical Slice

Dalam konteks DDD dan Clean Architecture, cara mengorganisasi kode menentukan seberapa modular monolith kita dan seberapa mudah nantinya kalau perlu dipecah jadi microservice.

### Horizontal Slice Architecture

**Horizontal slice** mengelompokkan kode berdasarkan peran teknisnya. Semua handler di satu folder, semua service di satu folder, semua repository di satu folder. Ini pattern yang paling sering ditemui di tutorial dan boilerplate.

```
project/
├── handler/
│   ├── order_handler.go
│   ├── payment_handler.go
│   └── user_handler.go
├── service/
│   ├── order_service.go
│   ├── payment_service.go
│   └── user_service.go
├── repository/
│   ├── order_repository.go
│   ├── payment_repository.go
│   └── user_repository.go
├── model/
│   ├── order.go
│   ├── payment.go
│   └── user.go
└── main.go
```

Contoh implementasi di `service/order_service.go`:

```go
package service

import "project/repository"

type OrderService struct {
    orderRepo   repository.OrderRepository
    paymentRepo repository.PaymentRepository
    userRepo    repository.UserRepository
}

func (s *OrderService) CreateOrder(userID int, items []OrderItem) (*Order, error) {
    user, err := s.userRepo.FindByID(userID)
    if err != nil {
        return nil, err
    }

    order := &Order{UserID: user.ID, Items: items}
    if err := s.orderRepo.Save(order); err != nil {
        return nil, err
    }

    return order, nil
}
```

Coba perhatikan: `OrderService` punya akses ke `repository.PaymentRepository` dan `repository.UserRepository`. Tidak ada yang melarang secara teknis. Compiler tidak komplain, linter tidak teriak. Satu-satunya penghalang ya disiplin developer. Di project yang sudah jalan berbulan-bulan dengan banyak kontributor, disiplin itu gampang tererosi.

Tapi kelebihan horizontal slice yang sering di-underestimate adalah **konsistensi layer**. Semua handler ada di satu tempat, jadi gampang memastikan pattern yang sama diterapkan di semua handler. Mau tambah middleware baru? Buka satu folder. Mau cek semua repository pakai transaction pattern yang sama? Satu folder juga. Keunggulan ini hilang di vertical slice.

### Vertical Slice Architecture

**Vertical slice** mengelompokkan kode berdasarkan fitur bisnis. Setiap slice berisi semua layer yang dibutuhkan fitur itu. Kalau horizontal slice memotong kode per layer, vertical slice memotong per fitur.

```
project/
├── order/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   ├── model.go
│   └── order.go          # public interface (port)
├── payment/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   └── model.go
├── user/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   └── model.go
├── shared/
│   └── database/
│       └── postgres.go
└── main.go
```

Kalau butuh data dari slice lain, dependency-nya lewat interface yang didefinisikan di consumer, bukan import langsung. File `order/order.go`:

```go
package order

import "context"

type Service interface {
    CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error)
    GetOrder(ctx context.Context, id int) (*Order, error)
}

type Repository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id int) (*Order, error)
}

// UserProvider adalah interface untuk mengakses data dari slice lain.
// Slice order tidak import package user secara langsung.
type UserProvider interface {
    GetUser(ctx context.Context, id int) (*UserInfo, error)
}

// UserInfo adalah struct minimal yang dibutuhkan slice order dari user.
type UserInfo struct {
    ID    int
    Name  string
    Email string
}
```

Implementasi service di `order/service.go`:

```go
package order

import "context"

type orderService struct {
    repo         Repository
    userProvider UserProvider
}

func NewService(repo Repository, userProvider UserProvider) Service {
    return &orderService{repo: repo, userProvider: userProvider}
}

func (s *orderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    user, err := s.userProvider.GetUser(ctx, req.UserID)
    if err != nil {
        return nil, err
    }

    order := &Order{
        UserID:   user.ID,
        UserName: user.Name,
        Items:    req.Items,
    }

    if err := s.repo.Save(ctx, order); err != nil {
        return nil, err
    }

    return order, nil
}
```

Slice `order` sama sekali tidak import package `user`. Ketergantungannya lewat interface `UserProvider` yang didefinisikan di slice `order` sendiri. Prinsipnya **Dependency Inversion**: yang butuh data yang bikin kontraknya, bukan yang nyediakan.

Semua dependency di-resolve di `main.go` sebagai composition root:

```go
package main

import (
    "project/order"
    "project/user"
    "project/shared/database"
)

func main() {
    db := database.NewPostgres("postgresql://localhost:5432/app")

    userRepo := user.NewRepository(db)
    userService := user.NewService(userRepo)

    orderRepo := order.NewRepository(db)
    orderService := order.NewService(orderRepo, userService)

    orderHandler := order.NewHandler(orderService)
    userHandler := user.NewHandler(userService)
    // ...
}
```

`userService` di-inject sebagai `UserProvider` ke `orderService`. Nanti kalau slice `user` perlu dipecah jadi microservice terpisah, tinggal ganti implementasi `UserProvider` dengan HTTP client atau gRPC client. Kode di slice `order` tidak perlu diubah sama sekali.

## Perbandingan

| Kriteria | Horizontal Slice | Vertical Slice |
| --- | --- | --- |
| **Organisasi kode** | Per layer teknis | Per fitur bisnis |
| **Coupling antar fitur** | Implicit, tidak ada boundary eksplisit | Eksplisit, melalui interface di consumer |
| **Merge conflict** | Sering, banyak developer edit folder yang sama | Jarang, setiap developer kerja di folder terpisah |
| **Konsistensi pattern** | Tinggi, semua handler/service di satu tempat | Perlu guideline ketat supaya antar slice konsisten |
| **Refactoring blast radius** | Luas, perubahan satu layer bisa affect semua fitur | Terisolasi per slice |
| **Boilerplate** | Minimal | Lebih banyak, setiap cross-slice dependency perlu interface |
| **Testing** | Mock bisa di-share antar test | Setiap slice punya mock sendiri, lebih terisolasi |
| **Kesiapan microservice** | Rendah, boundary tidak jelas | Tinggi, setiap slice sudah kandidat service |

Tidak ada yang menang di semua kriteria. Horizontal slice unggul di kesederhanaan, konsistensi, dan kecepatan iterasi awal. Vertical slice unggul di modularitas, isolasi, dan skalabilitas tim.

Soal **testing**, perbedaannya cukup kerasa. Di horizontal slice, satu mock `repository.UserRepository` bisa dipakai di test `OrderService`, `PaymentService`, dan service lain. Praktis, tapi semua service bergantung pada kontrak yang sama. Ubah interface repository, semua mock harus ikut berubah.

Di vertical slice, slice `order` punya `UserProvider` sendiri yang cuma berisi method yang dia butuhkan. Mock-nya spesifik dan kecil. Kalau slice `user` nambah method baru, mock di slice `order` tidak terpengaruh. Test jadi lebih stabil karena interface-nya tidak ikut membengkak. Trade-off-nya: setiap cross-slice dependency butuh interface dan mock tersendiri.

### Kapan Pakai yang Mana

Dari pengalaman saya, dua faktor yang paling menentukan: **ukuran tim** dan **kematangan domain**.

**Pakai horizontal slice kalau:**

- Project masih di fase eksplorasi, requirement belum stabil dan bounded context belum jelas
- Tim kecil (1-3 orang) yang mengerjakan semua fitur secara bergantian
- Project punya sedikit fitur (kurang dari 5 bounded context)
- Kecepatan iterasi lebih penting daripada modularitas jangka panjang

**Pakai vertical slice kalau:**

- Bounded context sudah jelas dan stabil
- Tim mulai besar (5+ developer) dan mengerjakan fitur berbeda secara paralel
- Merge conflict di folder `service/` atau `handler/` sudah jadi masalah rutin
- Ada rencana atau kemungkinan migrasi ke microservice
- Project sudah melewati fase MVP dan mulai butuh maintainability jangka panjang

**Pakai pendekatan hybrid kalau:**

- Project sedang dalam transisi, beberapa fitur sudah cukup kompleks untuk di-slice, tapi sebagian lain masih terlalu kecil
- Tim butuh shared utility (database, middleware, pagination) tanpa mengorbankan boundary antar fitur
- Migrasi dari horizontal ke vertical dilakukan secara bertahap, tidak big bang

Soal **granularity**, satu slice idealnya satu **bounded context** di DDD. Bukan satu entity, bukan satu use case. Slice `order` mencakup order creation, order status update, dan order history. Terlalu granular bikin overhead boilerplate tidak sebanding. Terlalu kasar bikin slice jadi monolith mini.

Buat yang project-nya sudah jalan pakai horizontal slice dan mau migrasi, lakukan per bounded context. Pilih satu yang paling independen, pindahkan ke vertical slice, validasi. Kalau sudah stabil, baru lanjut ke berikutnya. Jangan coba migrasi semuanya sekaligus.

## Insight dan Pembelajaran

- **Horizontal slice bukan pattern yang buruk**: untuk project kecil atau fase eksplorasi, horizontal slice justru lebih tepat. Jangan buru-buru migrasi ke vertical slice cuma karena terlihat lebih "modern"
- **Circular dependency itu fitur, bukan bug**: di Go, circular import error langsung mendeteksi boundary yang salah. Solusinya: pecah slice, ekstrak shared concern, atau pakai domain event supaya komunikasi jadi satu arah
- **Compiler itu penjaga boundary gratis**: manfaatkan compiler constraint di vertical slice, bukan dihindari. Kalau compiler menolak import, itu sinyal domain boundary perlu dievaluasi
- **Konsistensi antar slice butuh tooling**: tanpa guideline ketat, setiap slice bisa berkembang dengan pattern berbeda. Sediakan generator atau template untuk bikin slice baru supaya struktur file dan naming convention seragam sejak awal
- **Interface di consumer itu investasi, bukan overhead**: memang verbose, tapi ini yang bikin setiap slice bisa di-test dan di-refactor secara independen. Setelah 5-6 bounded context, overhead ini mulai terbayar
- **Shared package harus kecil dan jarang berubah**: kalau shared package sering di-commit, itu tanda ada domain logic yang seharusnya ada di slice
- **Migrasi bertahap lebih aman daripada big bang**: pindahkan satu bounded context, validasi, baru lanjut. Ini ngurangin risiko dan kasih waktu tim untuk terbiasa
- **Enforce boundary lewat tooling, bukan disiplin**: linter seperti `depguard` bisa nangkap import antar slice yang tidak seharusnya. Aturan arsitektur yang cuma andalkan code review pasti tererosi seiring waktu

## Penutup

DDD dan Clean Architecture menyediakan prinsip, tapi cara mengorganisasi kode yang menentukan apakah monolith kita benar-benar modular dan siap dipecah. Horizontal slice cocok untuk project yang masih kecil dan butuh kecepatan iterasi. Vertical slice cocok ketika bounded context sudah jelas dan tim butuh isolasi antar fitur. Yang penting, pahami trade-off-nya dan tahu kapan waktunya beralih.

## Referensi

- [Vertical Slice Architecture - Jimmy Bogard](https://www.jimmybogard.com/vertical-slice-architecture/), diakses pada 2026-07-04
- [Domain-Driven Design Reference - Eric Evans](https://www.domainlanguage.com/ddd/reference/), diakses pada 2026-07-04
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), diakses pada 2026-07-04
- [Restructuring to Vertical Slice Architecture - Derek Comartin](https://codeopinion.com/restructuring-to-a-vertical-slice-architecture/), diakses pada 2026-07-04
- [GopherCon 2018: Kat Zien - How Do You Structure Your Go Apps](https://www.youtube.com/watch?v=oL6JBUk6tj0), diakses pada 2026-07-04
