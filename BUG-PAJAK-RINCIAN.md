# 🛠️ TIKET BACKEND: Penambahan Permission Array pada PIN-Login

**Prioritas:** Tinggi (High)  
**Masalah:** Saat ini endpoint `POST /api/pengguna/pin-login` hanya mengembalikan `role` (string) tanpa menyertakan array `permissions`. Hal ini membuat *frontend* "buta" karena tidak mengetahui hak akses pengguna. *Frontend* tidak bisa melakukan *fetch* manual ke `/api/role` karena staf tidak memiliki izin `read-role` (ditolak 403 Forbidden).  

**Solusi:** Endpoint `pin-login` wajib mengembalikan array nama *permission* (contoh: `["read-inventory", "create-transfer-stok"]`) langsung di dalam payload respons.

Tolong modifikasi 2 file berikut:

### 1. File: `services/penggunaService.js`
**Fungsi:** `login({ nama, pin, tenantID })`

**[UBAH BAGIAN INI] (Sekitar baris 96)**
```javascript
// SEBELUM:
const pengguna = await Pengguna.findOne({
  nama,
  tenantID,
}).populate("roleID", "namaRole permissions");
```
```javascript
// SESUDAH (Gunakan nested populate untuk mengambil nama permission):
const pengguna = await Pengguna.findOne({
  nama,
  tenantID,
}).populate({
  path: "roleID",
  select: "namaRole permissions",
  populate: { path: "permissions", select: "nama" }
});
```

**[UBAH BAGIAN INI] (Sekitar baris 113)**
```javascript
// SEBELUM:
return {
  token: accessToken,
  refreshToken,
  user: {
    _id: pengguna._id,
    nama: pengguna.nama,
    role: pengguna.roleID.namaRole,
  },
};
```
```javascript
// SESUDAH (Ekstrak array of string dari permission):
return {
  token: accessToken,
  refreshToken,
  user: {
    _id: pengguna._id,
    nama: pengguna.nama,
    role: pengguna.roleID.namaRole,
    permissions: pengguna.roleID.permissions ? pengguna.roleID.permissions.map(p => p.nama || p) : []
  },
};
```

---

### 2. File: `controllers/penggunaController.js`
**Fungsi:** `loginPin(req, res, next)`

**[UBAH BAGIAN INI] (Sekitar baris 186)**
```javascript
// SEBELUM:
res.json({
  message: "Login pengguna berhasil.",
  data: {
    _id: result.user._id,
    nama: result.user.nama,
    role: result.user.roleID || result.user.role,
  },
  accessToken: result.token,
  refreshToken: result.refreshToken,
});
```
```javascript
// SESUDAH (Sisipkan permissions ke dalam data):
res.json({
  message: "Login pengguna berhasil.",
  data: {
    _id: result.user._id,
    nama: result.user.nama,
    role: result.user.roleID || result.user.role,
    permissions: result.user.permissions, // <--- Baris Tambahan
  },
  accessToken: result.token,
  refreshToken: result.refreshToken,
});
```

---
**Catatan Tambahan untuk Backend:**
Perubahan ini penting untuk menjaga **Single Source of Truth** di *backend* dan mencegah kebocoran data (*Principle of Least Privilege*). Dengan perbaikan ini, *frontend* bisa memuat *dashboard* staf dalam 1x pemanggilan API (menghemat 50% *network request*).
