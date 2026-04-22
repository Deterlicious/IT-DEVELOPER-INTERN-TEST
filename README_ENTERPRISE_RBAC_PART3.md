# 🏗️ Enterprise RBAC — Part 3: Frontend, Migration & Testing

## 9. Perubahan Frontend (Flutter)

### 9.1 Model Permission Helper

```dart
// lib/src/core/utils/permission_helper.dart — FILE BARU

class PermissionHelper {
  final List<String> _permissions;
  final bool isOwner;

  PermissionHelper({required List<String> permissions, required this.isOwner})
      : _permissions = permissions;

  /// Cek apakah user bisa akses modul tertentu (Layer 1)
  bool canAccessModule(String module) {
    if (isOwner) return true;
    return _permissions.contains('$module:access');
  }

  /// Cek apakah user punya permission spesifik (Layer 2)
  bool hasPermission(String permission) {
    if (isOwner) return true;
    return _permissions.contains(permission);
  }

  /// Cek apakah user punya salah satu dari beberapa permission
  bool hasAnyPermission(List<String> perms) {
    if (isOwner) return true;
    return perms.any((p) => _permissions.contains(p));
  }

  /// Ambil semua modul yang bisa diakses
  List<String> get accessibleModules {
    if (isOwner) {
      return ['dashboard', 'pos', 'inventory', 'gudang', 'keuangan',
              'pengeluaran', 'pengaturan', 'booking', 'hrm', 'laporan', 'aset'];
    }
    return _permissions
        .where((p) => p.endsWith(':access'))
        .map((p) => p.split(':').first)
        .toList();
  }

  /// Group permissions by module
  Map<String, List<String>> get groupedPermissions {
    final map = <String, List<String>>{};
    for (final p in _permissions) {
      final module = p.split(':').first;
      map.putIfAbsent(module, () => []).add(p);
    }
    return map;
  }
}
```

### 9.2 Integrasi dengan AuthProvider

```dart
// Di authProvider atau state management — tambahkan:

class AuthState {
  final Akun? profile;
  final Pengguna? activeStaff;
  final PermissionHelper? permissions;

  AuthState({this.profile, this.activeStaff, this.permissions});
}

// Saat login staff / switch user:
void loginStaff(Pengguna pengguna) {
  final isOwner = pengguna.role?.namaRole == 'Owner';
  final permList = pengguna.role?.permissions?.map((p) => p.nama).toList() ?? [];

  state = AuthState(
    profile: state.profile,
    activeStaff: pengguna,
    permissions: PermissionHelper(permissions: permList, isOwner: isOwner),
  );
}
```

### 9.3 Sidebar: Filter Menu Berdasarkan Module Access

```dart
// Di MasterSidebar — modifikasi menu items

Widget build(BuildContext context) {
  final perms = ref.watch(authProvider).permissions;

  // Definisi menu → module mapping
  final menuItems = [
    if (perms?.canAccessModule('dashboard') ?? true)
      _SidebarItem(icon: Icons.dashboard, label: 'Dashboard', route: '/dashboard'),

    if (perms?.canAccessModule('pos') ?? true)
      _SidebarItem(icon: Icons.point_of_sale, label: 'POS', route: '/pos'),

    if (perms?.canAccessModule('inventory') ?? true)
      _SidebarItem(icon: Icons.inventory_2, label: 'Inventaris', route: '/inventory'),

    if (perms?.canAccessModule('gudang') ?? true)
      _SidebarItem(icon: Icons.warehouse, label: 'Gudang', route: '/gudang'),

    if (perms?.canAccessModule('keuangan') ?? true)
      _SidebarItem(icon: Icons.account_balance, label: 'Keuangan', route: '/keuangan'),

    if (perms?.canAccessModule('pengeluaran') ?? true)
      _SidebarItem(icon: Icons.money_off, label: 'Pengeluaran', route: '/pengeluaran'),

    if (perms?.canAccessModule('booking') ?? true)
      _SidebarItem(icon: Icons.calendar_month, label: 'Booking', route: '/booking'),

    if (perms?.canAccessModule('pengaturan') ?? true)
      _SidebarItem(icon: Icons.settings, label: 'Pengaturan', route: '/pengaturan'),
  ];

  return ListView(children: menuItems.map((item) => _buildMenuItem(item)).toList());
}
```

### 9.4 Screen: Sembunyikan Tombol Berdasarkan Permission

```dart
// Contoh di POS Screen
Widget build(BuildContext context) {
  final perms = ref.watch(authProvider).permissions;

  return Column(
    children: [
      // Tombol buat transaksi — selalu tampil jika punya pos:create
      if (perms?.hasPermission('pos:create') ?? false)
        ElevatedButton(onPressed: _createTransaction, child: Text('Transaksi Baru')),

      // Tombol void — hanya supervisor+
      if (perms?.hasPermission('pos:void') ?? false)
        OutlinedButton(onPressed: _voidTransaction, child: Text('Void')),

      // Tombol diskon — hanya yang punya izin
      if (perms?.hasPermission('pos:discount') ?? false)
        OutlinedButton(onPressed: _applyDiscount, child: Text('Diskon Manual')),
    ],
  );
}
```

```dart
// Contoh di Gudang — Permintaan Stok
Widget _buildActionButtons(PermintaanStok item) {
  final perms = ref.watch(authProvider).permissions;

  return Row(
    children: [
      // Edit — hanya pembuat & draft
      if (perms?.hasPermission('gudang:permintaan.update') ?? false)
        if (item.status == 'DRAFT')
          IconButton(icon: Icon(Icons.edit), onPressed: () => _edit(item)),

      // Approve — hanya manager/supervisor
      if (perms?.hasPermission('gudang:permintaan.approve') ?? false)
        if (item.status == 'SUBMITTED')
          ElevatedButton(onPressed: () => _approve(item), child: Text('Approve')),

      // Reject — hanya manager/supervisor
      if (perms?.hasPermission('gudang:permintaan.reject') ?? false)
        if (item.status == 'SUBMITTED')
          OutlinedButton(onPressed: () => _reject(item), child: Text('Tolak')),
    ],
  );
}
```

### 9.5 UI Role Editor: Grouped Checkbox

```
┌──────────────────────────────────────────────┐
│  Buat Role Baru                              │
│                                              │
│  Nama Role: [________________]               │
│  Template:  [Pilih Template ▾]               │
│             ┌──────────────────┐             │
│             │ Kasir            │             │
│             │ Kasir Senior     │             │
│             │ Admin Gudang     │             │
│             │ Supervisor       │             │
│             │ Akuntan          │             │
│             │ Admin Toko       │             │
│             │ Custom (kosong)  │             │
│             └──────────────────┘             │
│                                              │
│  ─── Permission ──────────────────────────── │
│                                              │
│  ▼ POS                                       │
│    [✓] Akses Modul POS                       │
│    [✓] Buat Transaksi                        │
│    [✓] Lihat Riwayat                         │
│    [ ] Void Transaksi                        │
│    [ ] Diskon Manual                         │
│    [ ] Refund                                │
│                                              │
│  ▼ Gudang                                    │
│    [✓] Akses Modul Gudang                    │
│    [✓] Buat Permintaan Stok                  │
│    [ ] Approve Permintaan                    │
│    [ ] Reject Permintaan                     │
│    ...                                       │
│                                              │
│  ▶ Keuangan (collapsed)                      │
│  ▶ Pengaturan (collapsed)                    │
│                                              │
│  [Simpan Role]                               │
└──────────────────────────────────────────────┘
```

**Behavior:**
- Pilih template → auto-centang permissions sesuai template
- Bisa customize setelah pilih template (tambah/kurangi)
- Centang `:access` = expand group, uncheck `:access` = uncheck semua di grup

---

## 10. Migration Plan

### Phase 1: Backend Foundation (Week 1-2)

```
[ ] 1.1 Update permissionModel.js — tambah field modul, action, isModuleAccess
[ ] 1.2 Buat permissionSeedV2.js — 79 permissions baru
[ ] 1.3 Buat migration script: mapping permission lama → baru
[ ] 1.4 Update authorizePermission.js — versi baru dengan Owner bypass
[ ] 1.5 Update authPengguna.js — populate namaRole ke req.pengguna
[ ] 1.6 Buat roleTemplates.js di config/
[ ] 1.7 Tambah API: GET /role/templates
[ ] 1.8 Tambah API: POST /role/from-template
[ ] 1.9 Tambah API: GET /pengguna/me/permissions
[ ] 1.10 Test semua endpoint baru
```

### Phase 2: Backend Route Protection (Week 2-3)

```
[ ] 2.1 Terapkan checkModuleAccess + checkPermission di penjualanRoute.js
[ ] 2.2 Migrasi permintaanStokRoute.js ke format baru
[ ] 2.3 Terapkan di inventoryRoute.js, produkRoutes.js, kategoriRoutes.js
[ ] 2.4 Terapkan di transferStokRoute.js, locationRoute.js
[ ] 2.5 Terapkan di pembayaranRoute.js, akunKasRoute.js
[ ] 2.6 Terapkan di pajakRoute.js, diskonRoute.js, tarifRoute.js
[ ] 2.7 Terapkan di penggunaRoute.js, roleRoute.js
[ ] 2.8 Terapkan di sesiBookingRoute.js, membershipRoute.js
[ ] 2.9 Terapkan di bebanOperasionalRoute.js, kategoriBebanRoute.js
[ ] 2.10 Integration test semua protected routes
```

### Phase 3: Frontend Integration (Week 3-4)

```
[ ] 3.1 Buat PermissionHelper class di Flutter
[ ] 3.2 Integrasi dengan AuthProvider / state management
[ ] 3.3 Update MasterSidebar — filter menu berdasarkan module access
[ ] 3.4 Update POS Screen — conditional buttons
[ ] 3.5 Update Gudang screens — conditional approve/reject
[ ] 3.6 Update Keuangan screens — conditional actions
[ ] 3.7 Update Pengaturan screens — conditional menu items
[ ] 3.8 Buat UI Role Editor dengan grouped checkbox
[ ] 3.9 Buat UI Role Template selector
[ ] 3.10 E2E test: login kasir → sidebar hanya POS
```

### Phase 4: Polish & QA (Week 4-5)

```
[ ] 4.1 Error handling: 403 response → friendly message di Flutter
[ ] 4.2 Edge case: role dihapus saat user sedang login
[ ] 4.3 Edge case: permission diubah saat user sedang aktif
[ ] 4.4 Performance: cache permission di frontend
[ ] 4.5 Audit log: catat siapa mengubah role/permission
[ ] 4.6 QA test semua role template di device fisik
[ ] 4.7 Documentation update
```

### Migration Script (permission lama → baru)

```javascript
// scripts/migratePermissions.js
const MIGRATION_MAP = {
  // Format lama → Format baru
  'read-akun': 'pengaturan:tenant.update', // atau bisa dihapus
  'update-akun': 'pengaturan:tenant.update',
  'read-tenant': 'pengaturan:tenant.update',
  'update-tenant': 'pengaturan:tenant.update',
  'delete-tenant': 'pengaturan:tenant.update',
  'read-pengguna': 'pengaturan:pengguna.manage',
  'create-pengguna': 'pengaturan:pengguna.manage',
  'update-pengguna': 'pengaturan:pengguna.manage',
  'delete-pengguna': 'pengaturan:pengguna.manage',
  'read-role': 'pengaturan:role.manage',
  'create-role': 'pengaturan:role.manage',
  'update-role': 'pengaturan:role.manage',
  'delete-role': 'pengaturan:role.manage',
  'read-permission': 'pengaturan:role.manage',
  'create-permission': 'pengaturan:role.manage',
  'update-permission': 'pengaturan:role.manage',
  'delete-permission': 'pengaturan:role.manage',
  'read-inventory': 'inventory:read',
  'update-inventory-minimum': 'inventory:stok.minimum',
  'opname-inventory': 'inventory:stok.opname',
  'read-permintaan-stok': 'gudang:permintaan.create',
  'create-permintaan-stok': 'gudang:permintaan.create',
  'approve-permintaan-stok': 'gudang:permintaan.approve',
  'reject-permintaan-stok': 'gudang:permintaan.reject',
  'read-transfer-stok': 'gudang:transfer.create',
  'create-transfer-stok': 'gudang:transfer.create',
  'approve-transfer-stok': 'gudang:transfer.approve',
  'receive-transfer-stok': 'gudang:transfer.receive',
  'cancel-transfer-stok': 'gudang:transfer.cancel',
};

// Script akan:
// 1. Insert permission baru (V2)
// 2. Untuk setiap Role yang ada:
//    a. Baca permission lama
//    b. Map ke permission baru
//    c. Tambahkan :access permission untuk setiap modul yang ter-map
//    d. Update role.permissions dengan ID baru
// 3. Hapus permission lama
```

---

## 11. Testing Strategy

### 11.1 Unit Test: Middleware

```javascript
// __tests__/unit/middleware/authorizePermission.test.js
describe('checkPermission', () => {
  it('should allow Owner to bypass all checks', async () => {
    req.pengguna = { roleID: { namaRole: 'Owner' }, permissions: [] };
    const middleware = checkPermission('pos:void');
    middleware(req, res, next);
    expect(next).toHaveBeenCalledWith(); // no error
  });

  it('should block user without permission', async () => {
    req.pengguna = { roleID: { namaRole: 'Kasir' }, permissions: ['pos:access', 'pos:create'] };
    const middleware = checkPermission('pos:void');
    middleware(req, res, next);
    expect(next).toHaveBeenCalledWith(expect.objectContaining({ status: 403 }));
  });

  it('should allow user with correct permission', async () => {
    req.pengguna = { roleID: { namaRole: 'Kasir' }, permissions: ['pos:access', 'pos:create'] };
    const middleware = checkPermission('pos:create');
    middleware(req, res, next);
    expect(next).toHaveBeenCalledWith(); // no error
  });
});
```

### 11.2 Integration Test: Role + Permission Flow

```javascript
describe('Role Template Flow', () => {
  it('should create role from kasir template', async () => {
    const res = await request(app)
      .post('/api/role/from-template')
      .set('Authorization', `Bearer ${ownerToken}`)
      .send({ template: 'kasir', namaRole: 'Kasir Pagi' });
    expect(res.status).toBe(201);
    expect(res.body.data.permissions).toHaveLength(4); // dashboard:access, pos:access, pos:create, pos:read
  });

  it('kasir should only access POS endpoints', async () => {
    // Login sebagai kasir
    const kasirToken = await loginAsKasir();
    // Bisa akses POS
    const posRes = await request(app).get('/api/penjualan').set('Authorization', `Bearer ${kasirToken}`);
    expect(posRes.status).toBe(200);
    // TIDAK bisa akses Gudang
    const gudangRes = await request(app).get('/api/permintaanstok').set('Authorization', `Bearer ${kasirToken}`);
    expect(gudangRes.status).toBe(403);
  });
});
```

### 11.3 Test Matrix: Role × Feature

| Feature | Owner | Kasir | Kasir Senior | Admin Gudang | Supervisor | Akuntan |
|---------|:-----:|:-----:|:------------:|:------------:|:----------:|:-------:|
| Dashboard | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| POS: Buat Transaksi | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| POS: Void | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ |
| POS: Diskon Manual | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ |
| Gudang: Buka | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Gudang: Approve | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Keuangan: Buka | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Keuangan: Void Invoice | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Pengaturan: Kelola Staf | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Pengaturan: Kelola Role | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## 12. FAQ & Edge Cases

### Q: Owner perlu dicek permission tidak?
**A:** TIDAK. Owner selalu bypass. Ini di-handle di middleware level (`if (namaRole === "Owner") return next()`).

### Q: Bagaimana jika role dihapus saat user sedang login?
**A:** `authPengguna.js` sudah handle ini — jika `roleID` null setelah populate, return 403. Frontend harus catch error ini dan force re-login.

### Q: Bagaimana jika permission diubah saat user aktif?
**A:** Permission dibaca fresh setiap request (via `authPengguna.js` populate). Jadi perubahan permission langsung efektif di request berikutnya. Tidak perlu re-login.

### Q: Bisa tidak satu user punya lebih dari 1 role?
**A:** Saat ini TIDAK (1 user = 1 roleID). Jika dibutuhkan di masa depan, bisa ubah `roleID` menjadi array `roleIDs`, lalu merge semua permissions dari semua role.

### Q: Bagaimana handle route yang belum di-protect?
**A:** Gunakan pendekatan **deny-by-default**. Semua route yang pakai `authPengguna` secara default sudah memerlukan login. Tambahkan `checkModuleAccess` dan `checkPermission` secara bertahap. Route tanpa `checkPermission` tetap accessible oleh semua user yang login.

### Q: Bagaimana jika user coba akses halaman via URL langsung di Flutter?
**A:** Di Flutter route guard, cek permission sebelum navigasi:
```dart
// Di onGenerateRoute atau GoRouter redirect
if (routeName == '/gudang' && !(perms?.canAccessModule('gudang') ?? false)) {
  return MaterialPageRoute(builder: (_) => AccessDeniedScreen());
}
```

### Q: Bagaimana performance-nya? Setiap request query Permission collection?
**A:** `authPengguna.js` sudah melakukan `populate` pada `roleID.permissions` — ini 1 query JOIN. Permission list di-cache di `req.pengguna.permissions` sebagai array string. Overhead minimal (~2-5ms per request).

---

## 📋 Checklist Sebelum Mulai

- [ ] Tim Backend & Frontend sepakat dengan format `module:action`
- [ ] Daftar 79 permission di-review dan disetujui
- [ ] 6 role template di-review dan disetujui
- [ ] Prioritas module untuk Phase 2 ditentukan (mulai dari mana?)
- [ ] Backup database production sebelum migration

---

> **Dokumen ini adalah blueprint. Implementasi dilakukan bertahap sesuai Phase 1-4.**
>
> File terkait:
> - [Part 1 — Analisis & Arsitektur](./README_ENTERPRISE_RBAC.md)
> - [Part 2 — Implementasi Backend](./README_ENTERPRISE_RBAC_PART2.md)
> - **Part 3 — Frontend, Migration & Testing** ← Anda di sini
