# ⚠️ PROPOSAL: Integrasi Pembayaran → Akun Kas (Backend)

> **Dokumen ini menjelaskan mengapa backend (`pembayaranService.js`) HARUS dimodifikasi
> untuk menambahkan update saldo Akun Kas saat pembayaran invoice dilakukan.**

---

## 📌 Masalah Saat Ini

Saat ini, ketika pembayaran invoice berhasil dicatat (`POST /api/pembayaran`), backend:

- ✅ Membuat record `Pembayaran`
- ✅ Meng-update `totalDibayar` dan `statusBayar` di `Penjualan`
- ❌ **TIDAK** meng-update `saldo` di `AkunKas`

Artinya: **uang masuk tercatat, tapi saldo kas tidak ikut bertambah.**

---

## 🔴 Kasus 1: Data Keuangan Tidak Konsisten (KRITIS)

### Skenario
> Pelanggan A membayar invoice Rp 500.000 via **Transfer BCA**.
> Metode "Transfer BCA" terhubung ke Akun Kas **"Rekening BCA"** (saldo: Rp 1.000.000).

### Hasil Sekarang (SALAH)
| Data | Nilai |
|---|---|
| `Pembayaran.jumlahBayar` | Rp 500.000 ✅ recorded |
| `Penjualan.totalDibayar` | Rp 500.000 ✅ updated |
| `AkunKas "Rekening BCA".saldo` | **Rp 1.000.000** ❌ tidak berubah! |

### Dampak
- Dashboard keuangan menunjukkan saldo yang **salah**
- Laporan arus kas **tidak akurat**
- Rekonsiliasi bank manual → **buang waktu**
- Semakin banyak transaksi, semakin besar **selisih** antara saldo nyata vs saldo sistem

---

## 🔴 Kasus 2: Race Condition Jika Dikerjakan di Frontend

### Skenario
> 2 kasir bayar invoice BERSAMAAN via metode yang sama (misal: "Cash" → Akun Kas "Kas Fisik").

### Alur Jika Dikerjakan di Frontend

| Waktu | Kasir A | Kasir B |
|---|---|---|
| `T+0ms` | GET saldo → **Rp 1.000.000** | GET saldo → **Rp 1.000.000** |
| `T+50ms` | Hitung: 1.000.000 + 300.000 = 1.300.000 | Hitung: 1.000.000 + 200.000 = 1.200.000 |
| `T+100ms` | PUT saldo = **1.300.000** ✅ | — |
| `T+150ms` | — | PUT saldo = **1.200.000** ❌ Overwrite! |

### Hasil
| Seharusnya | Aktual |
|---|---|
| Rp 1.500.000 (1M + 300K + 200K) | **Rp 1.200.000** ❌ |
| Selisih: **Rp 300.000 hilang** dari sistem | |

### Mengapa Ini Terjadi?
Backend `akunKasService.update()` menggunakan `$set` (menimpa nilai), **bukan `$inc`** (menambah secara atomic).
Hanya backend yang bisa menggunakan `$inc` — frontend tidak punya akses ke operasi atomic MongoDB.

### Solusi Backend (1 baris)
```javascript
// ATOMIC — tidak mungkin race condition
await AkunKas.findByIdAndUpdate(akunKasID, { $inc: { saldo: jumlahBayar } });
```

---

## 🔴 Kasus 3: Inconsistent State Saat Network Error

### Skenario (Frontend Approach)
```
1. createPembayaran() → ✅ Sukses, pembayaran tersimpan
2. GET /api/akunkas/:id → ✅ Dapat saldo
3. PUT /api/akunkas/:id → ❌ TIMEOUT / Network Error
```

### Hasil
- Pembayaran **sudah tercatat** di database
- Saldo Akun Kas **belum terupdate**
- **Tidak ada mekanisme rollback** — data permanen tidak konsisten
- User tidak tahu bahwa update saldo gagal

### Jika Dikerjakan di Backend
```
1. Pembayaran.create() → sukses
2. AkunKas.$inc(saldo) → sukses/gagal di satu tempat
3. Jika gagal → bisa rollback pembayaran dalam satu transaction
```
Semua terjadi di **server yang sama, di satu request cycle** — tidak ada network hop yang bisa gagal.

---

## 🔴 Kasus 4: Operasi Delete/Update Pembayaran Tidak Ter-handle

### Skenario
> Admin menghapus pembayaran yang salah input.

### Jika Frontend-Only
- Harus buat logic di Flutter: "setelah delete pembayaran → GET akun kas → kurangi saldo → PUT ulang"
- Setiap screen yang punya fitur delete/edit pembayaran harus **copy-paste** logic yang sama
- Kalau ada screen baru yang bisa delete pembayaran di masa depan, developer harus **ingat** untuk tambahkan logic ini — rawan lupa

### Jika Backend
- Logic saldo ada di **1 tempat**: `pembayaranService.js`
- Semua operasi (create/update/delete) otomatis handle saldo
- Developer Flutter **tidak perlu tahu** tentang logic akun kas — sudah ter-enkapsulasi

---

## 🔴 Kasus 5: Multi-Platform & API Consumer Lain

### Skenario Masa Depan
Jika nanti ada:
- **Web dashboard** yang juga bisa create pembayaran
- **Integrasi Xendit webhook** yang auto-confirm pembayaran
- **Mobile app kedua** (misal app khusus kasir)

### Jika Frontend-Only
Setiap platform/consumer harus **implementasi ulang** logic update saldo akun kas.
Satu platform lupa? → saldo tidak konsisten.

### Jika Backend
Semua consumer cukup `POST /api/pembayaran` — saldo akun kas **otomatis terupdate** di server.
**Zero duplication, zero risk.**

---

## 🔴 Kasus 6: Backend SUDAH Melakukan Ini untuk Pengeluaran — Tapi TIDAK untuk Pemasukan

### Bukti dari `bebanOperasionalService.js`

Backend **sudah punya pattern yang persis sama** untuk beban operasional (uang keluar):

```javascript
// bebanOperasionalService.js — CREATE (line 78)
const updateKas = await AkunKas.findOneAndUpdate({
  _id: payload.akunKasID,
  tenantID: payload.tenantID
}, {
  $inc: { saldo: -payload.jumlah }  // ← KURANGI saldo saat ada pengeluaran
}, { new: true });

// bebanOperasionalService.js — DELETE (line 168)
await AkunKas.updateOne({
  _id: target.akunKasID,
  tenantID: requesterTenantID
}, {
  $inc: { saldo: target.jumlah }  // ← KEMBALIKAN saldo saat beban dihapus
});
```

### Kondisi Saat Ini (Tidak Simetris)

| Jenis Transaksi | Update Saldo Akun Kas? | Dampak |
|---|:---:|---|
| **Beban Operasional** (uang keluar) | ✅ `$inc: -jumlah` | Saldo berkurang otomatis |
| **Pembayaran Invoice** (uang masuk) | ❌ Tidak ada | Saldo TIDAK bertambah |

### Artinya
> Saldo akun kas **hanya turun** (karena beban operasional), tapi **tidak pernah naik** (dari pembayaran invoice).
>
> Semakin banyak invoice dibayar, semakin **tidak akurat** saldo akun kas — karena hanya sisi pengeluaran yang ter-track.
>
> Ini bukan fitur baru — ini **melengkapi pattern yang sudah ada** di codebase.

---

## 📊 Perbandingan Lengkap

| Aspek | Frontend-Only | Backend (Recommended) |
|---|---|---|
| **Atomicity** | ❌ GET → Calculate → SET (3 step, bisa race) | ✅ `$inc` atomic (1 step) |
| **Consistency** | ❌ Bisa inconsistent jika network gagal | ✅ Satu transaction cycle |
| **Maintenance** | ❌ Logic tersebar di tiap screen | ✅ 1 tempat (`pembayaranService.js`) |
| **Multi-platform** | ❌ Harus re-implement per platform | ✅ Otomatis untuk semua consumer |
| **Code changes** | ~20 baris di Flutter | **~3 baris** di backend |
| **Testing** | Sulit test race condition | Mudah test, atomic by design |
| **Rollback** | ❌ Tidak mungkin | ✅ Bisa dalam satu try/catch |

---

## ✅ Perubahan yang Dibutuhkan (Minimal)

### File: `services/pembayaranService.js`

**Hanya 3 titik perubahan di 1 file:**

#### 1. `create()` — setelah `Pembayaran.create()` (baris ~327)
```javascript
// Setelah: await this._syncPenjualan(...)
// Tambahkan:
if (payload.status === "PAID" && metodeValid.akunKasID) {
  await AkunKas.findByIdAndUpdate(metodeValid.akunKasID, {
    $inc: { saldo: jumlahBayarNum }
  });
  await redis.del(`akunkas:list:${payload.tenantID}`);
  await redis.del(`akunkas:detail:${metodeValid.akunKasID}`);
}
```

#### 2. `delete()` — sebelum `Pembayaran.deleteOne()` (baris ~502)
```javascript
// Jika pembayaran yang dihapus status PAID, kembalikan saldo
if (target.status === "PAID") {
  const metode = await MetodePembayaran.findById(target.metodePembayaranID);
  if (metode?.akunKasID) {
    await AkunKas.findByIdAndUpdate(metode.akunKasID, {
      $inc: { saldo: -(target.jumlahBayar || 0) }
    });
    await redis.del(`akunkas:list:${requesterTenantID}`);
    await redis.del(`akunkas:detail:${metode.akunKasID}`);
  }
}
```

#### 3. `update()` — saat jumlahBayar atau status berubah
```javascript
// Rollback saldo lama jika status sebelumnya PAID
// Apply saldo baru jika status baru PAID 
```

**Total: ~15 baris tambahan di 1 file. Zero breaking changes. Zero migration.**

---

## 🎯 Kesimpulan

> Backend modification **bukan opsi**, tapi **keharusan** untuk integritas data keuangan.
> 
> Tanpa ini, setiap pembayaran invoice akan membuat selisih saldo yang **terakumulasi**
> dan **tidak mungkin** diperbaiki secara otomatis.
> 
> Perubahan yang dibutuhkan: **~15 baris di 1 file**, tanpa breaking changes.
