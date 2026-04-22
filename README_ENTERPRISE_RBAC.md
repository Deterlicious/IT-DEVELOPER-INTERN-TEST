# 🏗️ Enterprise RBAC & Access Control — Blueprint Lengkap

> **Dokumen Perencanaan Teknis** untuk implementasi sistem akses fitur enterprise pada Ajis ERP.
> Versi: 1.0 | Tanggal: 23 April 2026

---

## 📑 Daftar Isi

1. [Analisis Sistem Saat Ini](#1-analisis-sistem-saat-ini)
2. [Masalah & Tantangan](#2-masalah--tantangan)
3. [Arsitektur Baru: 3-Layer Access Control](#3-arsitektur-baru-3-layer-access-control)
4. [Format Permission Baru: `module:action`](#4-format-permission-baru-moduleaction)
5. [Daftar Permission Lengkap Per Modul](#5-daftar-permission-lengkap-per-modul)
6. [Role Template System](#6-role-template-system)
7. [Perubahan Database Schema](#7-perubahan-database-schema)
8. [Perubahan Backend](#8-perubahan-backend)
9. [Perubahan Frontend (Flutter)](#9-perubahan-frontend-flutter)
10. [Migration Plan](#10-migration-plan)
11. [Testing Strategy](#11-testing-strategy)
12. [FAQ & Edge Cases](#12-faq--edge-cases)

---

## 1. Analisis Sistem Saat Ini

### 1.1 Database Schema Existing

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│    Akun      │     │   Tenant     │     │   Permission     │
│ (email/pass) │────▶│ (toko/cabang)│     │ nama: String     │
└──────────────┘     └──────┬───────┘     │ grup: String     │
                           │              │ deskripsi: String│
                           │              └────────┬─────────┘
                     ┌─────▼───────┐               │
                     │    Role     │               │
                     │ namaRole    │◄──────────────┘
                     │ permissions │  (array of ObjectId)
                     │ tenantID    │
                     └─────┬───────┘
                           │
                     ┌─────▼───────┐
                     │  Pengguna   │
                     │ nama, pin   │
                     │ roleID      │
                     │ tenantID    │
                     └─────────────┘
```

### 1.2 Permission yang Sudah Ada (permissionSeed.js)

| Grup | Permission | Deskripsi |
|------|-----------|-----------|
| **Akun** | `read-akun` | Melihat akun sendiri |
| | `update-akun` | Update akun sendiri |
| **Tenant** | `read-tenant` | Lihat toko |
| | `update-tenant` | Update toko |
| | `delete-tenant` | Hapus toko |
| **Pengguna** | `read-pengguna` | Lihat staf |
| | `create-pengguna` | Tambah staf |
| | `update-pengguna` | Edit staf |
| | `delete-pengguna` | Hapus staf |
| **Role** | `read-role` | Lihat role |
| | `create-role` | Tambah role |
| | `update-role` | Edit role |
| | `delete-role` | Hapus role |
| **Permission** | `read-permission` | Lihat permission |
| | `create-permission` | Tambah permission |
| | `update-permission` | Edit permission |
| | `delete-permission` | Hapus permission |
| **Inventory** | `read-inventory` | Lihat stok |
| | `update-inventory-minimum` | Edit minimum stok |
| | `opname-inventory` | Koreksi stok |
| **Permintaan Stok** | `read-permintaan-stok` | Lihat permintaan |
| | `create-permintaan-stok` | Buat permintaan |
| | `approve-permintaan-stok` | Setujui permintaan |
| | `reject-permintaan-stok` | Tolak permintaan |
| **Transfer Stok** | `read-transfer-stok` | Lihat transfer |
| | `create-transfer-stok` | Buat transfer |
| | `approve-transfer-stok` | Kirim barang |
| | `receive-transfer-stok` | Terima barang |
| | `cancel-transfer-stok` | Batalkan transfer |

**Total saat ini: 27 permissions, 7 grup**

### 1.3 Middleware yang Sudah Ada

```
middleware/
├── authAkun.js            → Verifikasi token akun (email/password level)
├── authPengguna.js        → Verifikasi token pengguna + populate permissions
├── authorizePermission.js → checkPermission('nama-permission') 
├── checkPermission.js     → Versi alternatif (sama fungsinya)
└── errorHandler.js        → Global error handler
```

### 1.4 Kondisi Route Saat Ini

| Route | Proteksi Auth | Proteksi Permission |
|-------|:---:|:---:|
| `/api/permintaanstok` | ✅ authPengguna | ✅ checkPermission per-endpoint |
| `/api/penjualan` | ✅ authPengguna | ❌ Belum ada |
| `/api/inventory` | ✅ authPengguna | ❌ Belum ada |
| `/api/transferstok` | ✅ authPengguna | ❌/Partial |
| `/api/produk` | ✅ authPengguna | ❌ Belum ada |
| `/api/pajak` | ✅ authPengguna | ❌ Belum ada |
| `/api/pembayaran` | ✅ authPengguna | ❌ Belum ada |
| ...dan lainnya | ✅ | ❌ |

> **Kesimpulan**: Hanya `permintaanStokRoute.js` yang sudah benar-benar menerapkan permission checking per-endpoint. Sisanya hanya dicek auth (siapa kamu) tapi tidak dicek izin (boleh ngapain).

---

## 2. Masalah & Tantangan

### 2.1 Kenapa Pendekatan Flat CRUD Bermasalah?

Jika membuat permission 1:1 per CRUD untuk semua 39 route files:

```
Estimasi: 39 routes × rata-rata 4 CRUD = ~156 permissions
+ aksi khusus (approve, reject, void, opname, dll) = ~200+ permissions
```

**Dampak:**

| Aspek | Dampak |
|-------|--------|
| **UX Admin** | Owner harus scroll 200 checkbox saat buat role baru |
| **Maintenance** | Setiap fitur baru = 4-5 migration permission |
| **Debugging** | "Kasir gabisa void" → harus cari di 200 string |
| **Konsistensi** | `read-inventory` vs `read-permintaan-stok` — format tak seragam |
| **Frontend** | Sidebar filtering jadi sangat kompleks |

### 2.2 Apa yang Dibutuhkan Enterprise ERP?

1. **Menu Visibility** — Kasir buka app → cuma lihat menu POS
2. **Action Control** — Kasir di POS → bisa buat transaksi, TIDAK bisa void
3. **Scalable** — Tambah modul Payroll bulan depan → cukup tambah 1 grup permission
4. **Role Template** — Owner buat role baru → pilih template "Kasir" → langsung jadi
5. **Audit Trail** — "Siapa yang approve permintaan stok ini?" → bisa dijawab

---

## 3. Arsitektur Baru: 3-Layer Access Control

```
╔══════════════════════════════════════════════════════════════╗
║                    3-LAYER ACCESS CONTROL                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  LAYER 1: MODULE ACCESS                                      ║
║  ┌─────────────────────────────────────────────────────┐     ║
║  │  "Modul mana yang bisa diakses?"                    │     ║
║  │  → Menentukan menu yang muncul di sidebar           │     ║
║  │  → Format: module:access                            │     ║
║  │  → Contoh: pos:access, gudang:access                │     ║
║  └─────────────────────────────────────────────────────┘     ║
║                          ↓                                    ║
║  LAYER 2: ACTION PERMISSION                                  ║
║  ┌─────────────────────────────────────────────────────┐     ║
║  │  "Di dalam modul, bisa ngapain?"                    │     ║
║  │  → Menentukan tombol/aksi yang aktif                │     ║
║  │  → Format: module:sub.action                        │     ║
║  │  → Contoh: gudang:permintaan.approve                │     ║
║  └─────────────────────────────────────────────────────┘     ║
║                          ↓                                    ║
║  LAYER 3: DATA SCOPE (Future Phase)                          ║
║  ┌─────────────────────────────────────────────────────┐     ║
║  │  "Data mana yang bisa dilihat?"                     │     ║
║  │  → own = hanya data sendiri                         │     ║
║  │  → branch = data 1 cabang                           │     ║
║  │  → all = semua cabang                               │     ║
║  └─────────────────────────────────────────────────────┘     ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### 3.1 Alur Pengecekan

```
User Request masuk
    │
    ▼
[authPengguna.js] ── Siapa kamu? Token valid?
    │
    ▼
[Layer 1: Module Access] ── Kamu boleh akses modul ini?
    │                        Cek: permissions.includes('pos:access')
    │                        Jika tidak → 403 "Modul tidak tersedia"
    ▼
[Layer 2: Action Permission] ── Kamu boleh lakukan aksi ini?
    │                            Cek: permissions.includes('pos:void')
    │                            Jika tidak → 403 "Anda tidak memiliki izin"
    ▼
[Controller] ── Proses request
```

---

## 4. Format Permission Baru: `module:action`

### 4.1 Konvensi Penamaan

```
Format:  {module}:{sub}.{action}
         │         │      │
         │         │      └── Aksi: access, create, read, update, delete,
         │         │              approve, reject, void, manage, export
         │         │
         │         └── Sub-fitur (opsional): permintaan, transfer, produk
         │
         └── Nama modul: pos, gudang, keuangan, inventory, pengaturan,
                         booking, hrm, laporan
```

### 4.2 Aturan Khusus

| Aturan | Penjelasan | Contoh |
|--------|-----------|--------|
| `:access` | Wajib ada per modul, untuk sidebar visibility | `pos:access` |
| `:manage` | Shortcut untuk full CRUD (create+read+update+delete) | `pengaturan:pajak.manage` |
| `:read` tanpa sub | Baca semua data di modul | `inventory:read` |
| `.approve` / `.reject` | Aksi approval workflow | `gudang:permintaan.approve` |
| `:export` | Ekspor data ke Excel/PDF | `laporan:keuangan.export` |

---

## 5. Daftar Permission Lengkap Per Modul

### 5.1 Modul: Dashboard (`dashboard`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `dashboard:access` | Bisa buka halaman dashboard | Module |
| `dashboard:analytics` | Lihat grafik & statistik | Action |

### 5.2 Modul: POS / Penjualan (`pos`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `pos:access` | Bisa buka modul POS | Module |
| `pos:create` | Buat transaksi penjualan baru | Action |
| `pos:read` | Lihat riwayat transaksi | Action |
| `pos:void` | Void/batalkan transaksi | Action |
| `pos:discount` | Berikan diskon manual | Action |
| `pos:refund` | Proses refund/retur | Action |

### 5.3 Modul: Inventory (`inventory`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `inventory:access` | Bisa buka modul Inventaris | Module |
| `inventory:read` | Lihat daftar produk & stok | Action |
| `inventory:produk.create` | Tambah produk baru | Action |
| `inventory:produk.update` | Edit detail produk | Action |
| `inventory:produk.delete` | Hapus produk | Action |
| `inventory:kategori.manage` | Kelola kategori produk | Action |
| `inventory:bahanbaku.manage` | Kelola bahan baku & resep | Action |
| `inventory:stok.minimum` | Edit batas minimum stok | Action |
| `inventory:stok.opname` | Koreksi stok fisik (opname) | Action |

### 5.4 Modul: Gudang / WMS (`gudang`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `gudang:access` | Bisa buka modul Gudang | Module |
| `gudang:read` | Lihat dashboard gudang | Action |
| `gudang:lokasi.manage` | Kelola lokasi penyimpanan | Action |
| `gudang:permintaan.create` | Buat permintaan stok | Action |
| `gudang:permintaan.update` | Edit draft permintaan | Action |
| `gudang:permintaan.approve` | Setujui permintaan stok | Action |
| `gudang:permintaan.reject` | Tolak permintaan stok | Action |
| `gudang:transfer.create` | Buat transfer stok antar lokasi | Action |
| `gudang:transfer.approve` | Kirim/approve transfer | Action |
| `gudang:transfer.receive` | Terima barang transfer | Action |
| `gudang:transfer.cancel` | Batalkan transfer | Action |
| `gudang:pembelian.create` | Buat pembelian stok | Action |
| `gudang:pembelian.approve` | Approve pembelian stok | Action |
| `gudang:jurnal.read` | Lihat jurnal stok | Action |

### 5.5 Modul: Keuangan (`keuangan`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `keuangan:access` | Bisa buka modul Keuangan | Module |
| `keuangan:read` | Lihat dashboard keuangan | Action |
| `keuangan:invoice.create` | Buat invoice | Action |
| `keuangan:invoice.update` | Edit invoice draft | Action |
| `keuangan:invoice.void` | Void invoice | Action |
| `keuangan:pembayaran.create` | Catat pembayaran masuk | Action |
| `keuangan:pembayaran.read` | Lihat riwayat pembayaran | Action |
| `keuangan:akunkas.manage` | Kelola akun kas/bank | Action |
| `keuangan:metode.manage` | Kelola metode pembayaran | Action |

### 5.6 Modul: Pengeluaran (`pengeluaran`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `pengeluaran:access` | Bisa buka modul Pengeluaran | Module |
| `pengeluaran:create` | Catat pengeluaran baru | Action |
| `pengeluaran:read` | Lihat daftar pengeluaran | Action |
| `pengeluaran:update` | Edit pengeluaran | Action |
| `pengeluaran:delete` | Hapus pengeluaran | Action |
| `pengeluaran:approve` | Approve pengeluaran (future) | Action |
| `pengeluaran:kategori.manage` | Kelola kategori beban | Action |

### 5.7 Modul: Pengaturan (`pengaturan`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `pengaturan:access` | Bisa buka modul Pengaturan | Module |
| `pengaturan:pajak.manage` | Kelola pajak & biaya layanan | Action |
| `pengaturan:diskon.manage` | Kelola diskon | Action |
| `pengaturan:tarif.manage` | Kelola tarif layanan | Action |
| `pengaturan:pengguna.manage` | Kelola staf/pengguna | Action |
| `pengaturan:role.manage` | Kelola role & permission | Action |
| `pengaturan:tenant.update` | Edit info toko | Action |

### 5.8 Modul: Booking (`booking`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `booking:access` | Bisa buka modul Booking | Module |
| `booking:create` | Buat sesi booking baru | Action |
| `booking:read` | Lihat jadwal booking | Action |
| `booking:update` | Edit booking | Action |
| `booking:cancel` | Batalkan booking | Action |
| `booking:membership.manage` | Kelola membership & paket | Action |
| `booking:pelanggan.manage` | Kelola data pelanggan | Action |

### 5.9 Modul: HRM — Human Resource (Future) (`hrm`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `hrm:access` | Bisa buka modul HRM | Module |
| `hrm:absensi.read` | Lihat absensi | Action |
| `hrm:absensi.manage` | Kelola absensi | Action |
| `hrm:izincuti.create` | Ajukan izin/cuti | Action |
| `hrm:izincuti.approve` | Approve izin/cuti | Action |
| `hrm:kontrak.manage` | Kelola kontrak & kompensasi | Action |
| `hrm:posisi.manage` | Kelola posisi/jabatan | Action |

### 5.10 Modul: Laporan (Future) (`laporan`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `laporan:access` | Bisa buka modul Laporan | Module |
| `laporan:harian.read` | Lihat laporan harian | Action |
| `laporan:bulanan.read` | Lihat laporan bulanan | Action |
| `laporan:keuangan.export` | Ekspor laporan keuangan | Action |
| `laporan:stok.export` | Ekspor laporan stok | Action |

### 5.11 Modul: Aset (Future) (`aset`)

| Permission | Deskripsi | Layer |
|-----------|-----------|-------|
| `aset:access` | Bisa buka modul Aset | Module |
| `aset:create` | Tambah aset baru | Action |
| `aset:read` | Lihat daftar aset | Action |
| `aset:update` | Edit aset | Action |
| `aset:delete` | Hapus aset | Action |
| `aset:tipe.manage` | Kelola tipe aset | Action |

### Ringkasan Total

| Modul | Jumlah Permission |
|-------|:-:|
| Dashboard | 2 |
| POS | 6 |
| Inventory | 9 |
| Gudang/WMS | 14 |
| Keuangan | 9 |
| Pengeluaran | 7 |
| Pengaturan | 7 |
| Booking | 7 |
| HRM | 7 |
| Laporan | 5 |
| Aset | 6 |
| **TOTAL** | **79** |

> Dibanding flat CRUD (~200+), format `module:action` menghasilkan **79 permission** yang terstruktur rapi per modul.

---

## 6. Role Template System

### 6.1 Template Bawaan

Owner tidak perlu centang 79 permission satu-satu. Sistem menyediakan **template role** yang bisa langsung dipakai atau di-customize.

#### 🟢 Template: Kasir

```json
{
  "namaRole": "Kasir",
  "deskripsi": "Akses POS untuk transaksi penjualan harian",
  "template": "kasir",
  "permissions": [
    "dashboard:access",
    "pos:access",
    "pos:create",
    "pos:read"
  ]
}
```

> **Catatan:** Kasir TIDAK bisa void, diskon manual, atau refund.

#### 🔵 Template: Kasir Senior

```json
{
  "namaRole": "Kasir Senior",
  "deskripsi": "Kasir dengan kemampuan void dan diskon",
  "template": "kasir_senior",
  "permissions": [
    "dashboard:access",
    "pos:access",
    "pos:create",
    "pos:read",
    "pos:void",
    "pos:discount",
    "pos:refund"
  ]
}
```

#### 🟠 Template: Admin Gudang

```json
{
  "namaRole": "Admin Gudang",
  "deskripsi": "Mengelola stok, permintaan, dan transfer barang",
  "template": "admin_gudang",
  "permissions": [
    "dashboard:access",
    "gudang:access",
    "gudang:read",
    "gudang:lokasi.manage",
    "gudang:permintaan.create",
    "gudang:permintaan.update",
    "gudang:permintaan.approve",
    "gudang:permintaan.reject",
    "gudang:transfer.create",
    "gudang:transfer.approve",
    "gudang:transfer.receive",
    "gudang:transfer.cancel",
    "gudang:pembelian.create",
    "gudang:jurnal.read",
    "inventory:access",
    "inventory:read",
    "inventory:stok.minimum",
    "inventory:stok.opname"
  ]
}
```

#### 🟣 Template: Supervisor

```json
{
  "namaRole": "Supervisor",
  "deskripsi": "Monitoring operasional dan approval lintas modul",
  "template": "supervisor",
  "permissions": [
    "dashboard:access",
    "dashboard:analytics",
    "pos:access",
    "pos:create",
    "pos:read",
    "pos:void",
    "pos:discount",
    "gudang:access",
    "gudang:read",
    "gudang:permintaan.approve",
    "gudang:permintaan.reject",
    "gudang:transfer.approve",
    "gudang:jurnal.read",
    "inventory:access",
    "inventory:read",
    "keuangan:access",
    "keuangan:read",
    "pengeluaran:access",
    "pengeluaran:read",
    "pengeluaran:approve"
  ]
}
```

#### 🔴 Template: Akuntan

```json
{
  "namaRole": "Akuntan",
  "deskripsi": "Keuangan, invoice, pembayaran, dan pajak",
  "template": "akuntan",
  "permissions": [
    "dashboard:access",
    "dashboard:analytics",
    "keuangan:access",
    "keuangan:read",
    "keuangan:invoice.create",
    "keuangan:invoice.update",
    "keuangan:invoice.void",
    "keuangan:pembayaran.create",
    "keuangan:pembayaran.read",
    "keuangan:akunkas.manage",
    "keuangan:metode.manage",
    "pengeluaran:access",
    "pengeluaran:create",
    "pengeluaran:read",
    "pengeluaran:update",
    "pengeluaran:kategori.manage",
    "pengaturan:access",
    "pengaturan:pajak.manage",
    "laporan:access",
    "laporan:harian.read",
    "laporan:bulanan.read",
    "laporan:keuangan.export"
  ]
}
```

#### ⚫ Template: Admin Toko (Full Access tanpa Owner)

```json
{
  "namaRole": "Admin Toko",
  "deskripsi": "Akses penuh kecuali manajemen tenant dan role",
  "template": "admin_toko",
  "permissions": [
    "dashboard:access", "dashboard:analytics",
    "pos:access", "pos:create", "pos:read", "pos:void", "pos:discount", "pos:refund",
    "inventory:access", "inventory:read", "inventory:produk.create", "inventory:produk.update",
    "inventory:produk.delete", "inventory:kategori.manage", "inventory:bahanbaku.manage",
    "inventory:stok.minimum", "inventory:stok.opname",
    "gudang:access", "gudang:read", "gudang:lokasi.manage",
    "gudang:permintaan.create", "gudang:permintaan.update",
    "gudang:permintaan.approve", "gudang:permintaan.reject",
    "gudang:transfer.create", "gudang:transfer.approve",
    "gudang:transfer.receive", "gudang:transfer.cancel",
    "gudang:pembelian.create", "gudang:pembelian.approve", "gudang:jurnal.read",
    "keuangan:access", "keuangan:read",
    "keuangan:invoice.create", "keuangan:invoice.update", "keuangan:invoice.void",
    "keuangan:pembayaran.create", "keuangan:pembayaran.read",
    "keuangan:akunkas.manage", "keuangan:metode.manage",
    "pengeluaran:access", "pengeluaran:create", "pengeluaran:read",
    "pengeluaran:update", "pengeluaran:delete", "pengeluaran:approve",
    "pengeluaran:kategori.manage",
    "pengaturan:access", "pengaturan:pajak.manage", "pengaturan:diskon.manage",
    "pengaturan:tarif.manage", "pengaturan:pengguna.manage",
    "booking:access", "booking:create", "booking:read", "booking:update", "booking:cancel",
    "booking:membership.manage", "booking:pelanggan.manage"
  ]
}
```

> **Owner** tidak memerlukan template — Owner otomatis bypass semua permission check.

Lanjutan ada di **Part 2**: [README_ENTERPRISE_RBAC_PART2.md](./README_ENTERPRISE_RBAC_PART2.md)
