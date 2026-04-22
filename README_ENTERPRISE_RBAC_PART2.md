# 🏗️ Enterprise RBAC — Part 2: Implementasi Backend

## 7. Perubahan Database Schema

### 7.1 Permission Model (UPDATED)

```javascript
// models/permissionModel.js — PERUBAHAN
const permissionSchema = new mongoose.Schema({
  nama: { type: String, required: true, unique: true, trim: true },
  // BARU: field 'modul' menggantikan 'grup'
  modul: { type: String, required: true, trim: true, index: true },
  // BARU: field 'action' untuk identifikasi aksi
  action: { type: String, required: true, trim: true },
  deskripsi: { type: String, default: null },
  // BARU: flag apakah ini permission akses modul
  isModuleAccess: { type: Boolean, default: false },
});

permissionSchema.index({ modul: 1, action: 1 }, { unique: true });
```

**Mapping field lama → baru:**
| Lama | Baru | Contoh |
|------|------|--------|
| `nama: "read-inventory"` | `nama: "inventory:read"` | Format colon |
| `grup: "Inventory"` | `modul: "inventory"` | Lowercase, konsisten |
| _(tidak ada)_ | `action: "read"` | Aksi terpisah |
| _(tidak ada)_ | `isModuleAccess: false` | Flag untuk `:access` |

### 7.2 Role Model (TETAP, tambah field template)

```javascript
// models/roleModel.js — PERUBAHAN MINOR
const roleSchema = new mongoose.Schema({
  tenantID: { type: mongoose.Schema.Types.ObjectId, ref: "Tenant", required: true },
  namaRole: { type: String, required: true, trim: true },
  deskripsi: { type: String, default: null },
  // BARU: template asal (null jika custom)
  template: { type: String, default: null, enum: [null, 'kasir', 'kasir_senior', 'admin_gudang', 'supervisor', 'akuntan', 'admin_toko'] },
  permissions: [{ type: mongoose.Schema.Types.ObjectId, ref: "Permission" }],
});
```

### 7.3 Pengguna Model (TIDAK BERUBAH)

Model Pengguna tetap sama — `roleID` sudah mereferensi Role yang berisi permissions.

---

## 8. Perubahan Backend

### 8.1 Permission Seed Baru

```javascript
// seeds/permissionSeedV2.js
const permissionsList = [
  // === DASHBOARD ===
  { nama: "dashboard:access", modul: "dashboard", action: "access", deskripsi: "Buka halaman dashboard", isModuleAccess: true },
  { nama: "dashboard:analytics", modul: "dashboard", action: "analytics", deskripsi: "Lihat grafik & statistik" },

  // === POS ===
  { nama: "pos:access", modul: "pos", action: "access", deskripsi: "Buka modul POS", isModuleAccess: true },
  { nama: "pos:create", modul: "pos", action: "create", deskripsi: "Buat transaksi penjualan" },
  { nama: "pos:read", modul: "pos", action: "read", deskripsi: "Lihat riwayat transaksi" },
  { nama: "pos:void", modul: "pos", action: "void", deskripsi: "Void/batalkan transaksi" },
  { nama: "pos:discount", modul: "pos", action: "discount", deskripsi: "Berikan diskon manual" },
  { nama: "pos:refund", modul: "pos", action: "refund", deskripsi: "Proses refund/retur" },

  // === INVENTORY ===
  { nama: "inventory:access", modul: "inventory", action: "access", deskripsi: "Buka modul Inventaris", isModuleAccess: true },
  { nama: "inventory:read", modul: "inventory", action: "read", deskripsi: "Lihat daftar produk & stok" },
  { nama: "inventory:produk.create", modul: "inventory", action: "produk.create", deskripsi: "Tambah produk baru" },
  { nama: "inventory:produk.update", modul: "inventory", action: "produk.update", deskripsi: "Edit detail produk" },
  { nama: "inventory:produk.delete", modul: "inventory", action: "produk.delete", deskripsi: "Hapus produk" },
  { nama: "inventory:kategori.manage", modul: "inventory", action: "kategori.manage", deskripsi: "Kelola kategori" },
  { nama: "inventory:bahanbaku.manage", modul: "inventory", action: "bahanbaku.manage", deskripsi: "Kelola bahan baku" },
  { nama: "inventory:stok.minimum", modul: "inventory", action: "stok.minimum", deskripsi: "Edit minimum stok" },
  { nama: "inventory:stok.opname", modul: "inventory", action: "stok.opname", deskripsi: "Koreksi stok fisik" },

  // === GUDANG ===
  { nama: "gudang:access", modul: "gudang", action: "access", deskripsi: "Buka modul Gudang", isModuleAccess: true },
  { nama: "gudang:read", modul: "gudang", action: "read", deskripsi: "Lihat dashboard gudang" },
  { nama: "gudang:lokasi.manage", modul: "gudang", action: "lokasi.manage", deskripsi: "Kelola lokasi" },
  { nama: "gudang:permintaan.create", modul: "gudang", action: "permintaan.create", deskripsi: "Buat permintaan stok" },
  { nama: "gudang:permintaan.update", modul: "gudang", action: "permintaan.update", deskripsi: "Edit draft permintaan" },
  { nama: "gudang:permintaan.approve", modul: "gudang", action: "permintaan.approve", deskripsi: "Setujui permintaan" },
  { nama: "gudang:permintaan.reject", modul: "gudang", action: "permintaan.reject", deskripsi: "Tolak permintaan" },
  { nama: "gudang:transfer.create", modul: "gudang", action: "transfer.create", deskripsi: "Buat transfer stok" },
  { nama: "gudang:transfer.approve", modul: "gudang", action: "transfer.approve", deskripsi: "Approve transfer" },
  { nama: "gudang:transfer.receive", modul: "gudang", action: "transfer.receive", deskripsi: "Terima barang" },
  { nama: "gudang:transfer.cancel", modul: "gudang", action: "transfer.cancel", deskripsi: "Batalkan transfer" },
  { nama: "gudang:pembelian.create", modul: "gudang", action: "pembelian.create", deskripsi: "Buat pembelian" },
  { nama: "gudang:pembelian.approve", modul: "gudang", action: "pembelian.approve", deskripsi: "Approve pembelian" },
  { nama: "gudang:jurnal.read", modul: "gudang", action: "jurnal.read", deskripsi: "Lihat jurnal stok" },

  // === KEUANGAN ===
  { nama: "keuangan:access", modul: "keuangan", action: "access", deskripsi: "Buka modul Keuangan", isModuleAccess: true },
  { nama: "keuangan:read", modul: "keuangan", action: "read", deskripsi: "Lihat dashboard keuangan" },
  { nama: "keuangan:invoice.create", modul: "keuangan", action: "invoice.create", deskripsi: "Buat invoice" },
  { nama: "keuangan:invoice.update", modul: "keuangan", action: "invoice.update", deskripsi: "Edit invoice" },
  { nama: "keuangan:invoice.void", modul: "keuangan", action: "invoice.void", deskripsi: "Void invoice" },
  { nama: "keuangan:pembayaran.create", modul: "keuangan", action: "pembayaran.create", deskripsi: "Catat pembayaran" },
  { nama: "keuangan:pembayaran.read", modul: "keuangan", action: "pembayaran.read", deskripsi: "Lihat pembayaran" },
  { nama: "keuangan:akunkas.manage", modul: "keuangan", action: "akunkas.manage", deskripsi: "Kelola akun kas" },
  { nama: "keuangan:metode.manage", modul: "keuangan", action: "metode.manage", deskripsi: "Kelola metode bayar" },

  // === PENGELUARAN ===
  { nama: "pengeluaran:access", modul: "pengeluaran", action: "access", deskripsi: "Buka modul Pengeluaran", isModuleAccess: true },
  { nama: "pengeluaran:create", modul: "pengeluaran", action: "create", deskripsi: "Catat pengeluaran" },
  { nama: "pengeluaran:read", modul: "pengeluaran", action: "read", deskripsi: "Lihat pengeluaran" },
  { nama: "pengeluaran:update", modul: "pengeluaran", action: "update", deskripsi: "Edit pengeluaran" },
  { nama: "pengeluaran:delete", modul: "pengeluaran", action: "delete", deskripsi: "Hapus pengeluaran" },
  { nama: "pengeluaran:approve", modul: "pengeluaran", action: "approve", deskripsi: "Approve pengeluaran" },
  { nama: "pengeluaran:kategori.manage", modul: "pengeluaran", action: "kategori.manage", deskripsi: "Kelola kategori beban" },

  // === PENGATURAN ===
  { nama: "pengaturan:access", modul: "pengaturan", action: "access", deskripsi: "Buka Pengaturan", isModuleAccess: true },
  { nama: "pengaturan:pajak.manage", modul: "pengaturan", action: "pajak.manage", deskripsi: "Kelola pajak" },
  { nama: "pengaturan:diskon.manage", modul: "pengaturan", action: "diskon.manage", deskripsi: "Kelola diskon" },
  { nama: "pengaturan:tarif.manage", modul: "pengaturan", action: "tarif.manage", deskripsi: "Kelola tarif" },
  { nama: "pengaturan:pengguna.manage", modul: "pengaturan", action: "pengguna.manage", deskripsi: "Kelola staf" },
  { nama: "pengaturan:role.manage", modul: "pengaturan", action: "role.manage", deskripsi: "Kelola role" },
  { nama: "pengaturan:tenant.update", modul: "pengaturan", action: "tenant.update", deskripsi: "Edit info toko" },

  // === BOOKING ===
  { nama: "booking:access", modul: "booking", action: "access", deskripsi: "Buka modul Booking", isModuleAccess: true },
  { nama: "booking:create", modul: "booking", action: "create", deskripsi: "Buat booking" },
  { nama: "booking:read", modul: "booking", action: "read", deskripsi: "Lihat jadwal" },
  { nama: "booking:update", modul: "booking", action: "update", deskripsi: "Edit booking" },
  { nama: "booking:cancel", modul: "booking", action: "cancel", deskripsi: "Batalkan booking" },
  { nama: "booking:membership.manage", modul: "booking", action: "membership.manage", deskripsi: "Kelola membership" },
  { nama: "booking:pelanggan.manage", modul: "booking", action: "pelanggan.manage", deskripsi: "Kelola pelanggan" },
];
```

### 8.2 Middleware `checkPermission` (UPDATED)

```javascript
// middleware/authorizePermission.js — VERSI BARU
const createError = require("http-errors");

// Single permission check
exports.checkPermission = (permissionName) => {
  return (req, res, next) => {
    if (!req.pengguna) {
      return next(createError(401, "Token pengguna diperlukan."));
    }

    // Owner bypass semua permission
    const roleName = req.pengguna.roleID?.namaRole;
    if (roleName === "Owner") return next();

    const permissions = req.pengguna.permissions || [];
    if (!permissions.includes(permissionName)) {
      return next(createError(403, `Izin ditolak: '${permissionName}'`));
    }
    next();
  };
};

// Multiple permissions check (ANY — salah satu cukup)
exports.checkAnyPermission = (...permissionNames) => {
  return (req, res, next) => {
    if (!req.pengguna) {
      return next(createError(401, "Token pengguna diperlukan."));
    }
    const roleName = req.pengguna.roleID?.namaRole;
    if (roleName === "Owner") return next();

    const permissions = req.pengguna.permissions || [];
    const hasAny = permissionNames.some(p => permissions.includes(p));
    if (!hasAny) {
      return next(createError(403, `Izin ditolak. Butuh salah satu: ${permissionNames.join(', ')}`));
    }
    next();
  };
};

// Module access check (untuk sidebar/menu)
exports.checkModuleAccess = (moduleName) => {
  return (req, res, next) => {
    if (!req.pengguna) {
      return next(createError(401, "Token pengguna diperlukan."));
    }
    const roleName = req.pengguna.roleID?.namaRole;
    if (roleName === "Owner") return next();

    const permissions = req.pengguna.permissions || [];
    if (!permissions.includes(`${moduleName}:access`)) {
      return next(createError(403, `Modul '${moduleName}' tidak tersedia untuk role Anda.`));
    }
    next();
  };
};
```

### 8.3 Contoh Route dengan Permission Baru

```javascript
// routes/penjualanRoute.js — CONTOH PENERAPAN
const { checkPermission, checkModuleAccess } = require("../middleware/authorizePermission");

router.use(authPengguna);
router.use(checkModuleAccess("pos")); // Layer 1: cek akses modul

router.post("/", checkPermission("pos:create"), wrap(ctrl.create));
router.get("/", checkPermission("pos:read"), wrap(ctrl.getAll));
router.get("/:id", checkPermission("pos:read"), wrap(ctrl.getById));
router.delete("/:id", checkPermission("pos:void"), wrap(ctrl.delete));
```

```javascript
// routes/permintaanStokRoute.js — MIGRASI
router.use(authPengguna);
router.use(checkModuleAccess("gudang"));

router.get("/", checkPermission("gudang:permintaan.create"), wrap(ctrl.getAll));
router.post("/", checkPermission("gudang:permintaan.create"), wrap(ctrl.create));
router.put("/:id", checkPermission("gudang:permintaan.update"), wrap(ctrl.update));
router.patch("/:id/submit", checkPermission("gudang:permintaan.update"), wrap(ctrl.submit));
router.patch("/:id/approve", checkPermission("gudang:permintaan.approve"), wrap(ctrl.approve));
router.patch("/:id/reject", checkPermission("gudang:permintaan.reject"), wrap(ctrl.reject));
```

### 8.4 API Endpoint: Role Templates

```javascript
// controllers/roleController.js — TAMBAH endpoint template
exports.getRoleTemplates = async (req, res) => {
  const templates = require("../config/roleTemplates");
  res.json({ message: "Daftar template role", data: templates });
};

exports.createFromTemplate = async (req, res) => {
  const { template, namaRole, deskripsi } = req.body;
  const templates = require("../config/roleTemplates");
  const tpl = templates[template];
  if (!tpl) throw createError(400, "Template tidak ditemukan");

  // Cari permission IDs berdasarkan nama
  const permissions = await Permission.find({ nama: { $in: tpl.permissions } });
  const role = await Role.create({
    tenantID: req.pengguna.tenantID,
    namaRole: namaRole || tpl.label,
    deskripsi: deskripsi || tpl.description,
    template,
    permissions: permissions.map(p => p._id),
  });
  res.status(201).json({ message: "Role berhasil dibuat dari template", data: role });
};
```

### 8.5 API: Get User Permissions (Grouped)

```javascript
// GET /api/pengguna/me/permissions — endpoint baru
exports.getMyPermissions = async (req, res) => {
  const permissions = req.pengguna.permissions || [];
  // Group by module
  const grouped = {};
  permissions.forEach(p => {
    const [modul] = p.split(":");
    if (!grouped[modul]) grouped[modul] = [];
    grouped[modul].push(p);
  });
  res.json({
    role: req.pengguna.roleID?.namaRole,
    isOwner: req.pengguna.roleID?.namaRole === "Owner",
    modules: Object.keys(grouped),
    permissions,
    grouped,
  });
};
```

### 8.6 Mapping Route → Permission (Semua Files)

| Route File | Module | Permissions yang Diterapkan |
|-----------|--------|---------------------------|
| `penjualanRoute.js` | `pos` | `pos:access`, `pos:create`, `pos:read`, `pos:void` |
| `inventoryRoute.js` | `inventory` | `inventory:access`, `inventory:read`, `inventory:stok.minimum`, `inventory:stok.opname` |
| `produkRoutes.js` | `inventory` | `inventory:produk.create/update/delete` |
| `kategoriRoutes.js` | `inventory` | `inventory:kategori.manage` |
| `bahanbakuRoutes.js` | `inventory` | `inventory:bahanbaku.manage` |
| `permintaanStokRoute.js` | `gudang` | `gudang:permintaan.*` |
| `transferStokRoute.js` | `gudang` | `gudang:transfer.*` |
| `pembelianStokRoute.js` | `gudang` | `gudang:pembelian.*` |
| `locationRoute.js` | `gudang` | `gudang:lokasi.manage` |
| `jurnalStokRoute.js` | `gudang` | `gudang:jurnal.read` |
| `pembayaranRoute.js` | `keuangan` | `keuangan:pembayaran.*` |
| `akunKasRoute.js` | `keuangan` | `keuangan:akunkas.manage` |
| `metodePembayaranRoute.js` | `keuangan` | `keuangan:metode.manage` |
| `bebanOperasionalRoute.js` | `pengeluaran` | `pengeluaran:*` |
| `kategoriBebanRoute.js` | `pengeluaran` | `pengeluaran:kategori.manage` |
| `pajakRoute.js` | `pengaturan` | `pengaturan:pajak.manage` |
| `diskonRoute.js` | `pengaturan` | `pengaturan:diskon.manage` |
| `tarifRoute.js` | `pengaturan` | `pengaturan:tarif.manage` |
| `penggunaRoute.js` | `pengaturan` | `pengaturan:pengguna.manage` |
| `roleRoute.js` | `pengaturan` | `pengaturan:role.manage` |
| `tenantRoute.js` | `pengaturan` | `pengaturan:tenant.update` |
| `sesiBookingRoute.js` | `booking` | `booking:*` |
| `membershipRoute.js` | `booking` | `booking:membership.manage` |
| `pelangganRoute.js` | `booking` | `booking:pelanggan.manage` |

---

Lanjutan: [Part 3 — Frontend & Migration](./README_ENTERPRISE_RBAC_PART3.md)
