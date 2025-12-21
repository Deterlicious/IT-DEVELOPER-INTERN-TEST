# Job Architecture Management API

![NodeJS](https://img.shields.io/badge/Node.js-v14+-green.svg) ![Express](https://img.shields.io/badge/Express-v4.17-lightgrey.svg) ![Sequelize](https://img.shields.io/badge/ORM-Sequelize-blue.svg) ![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-blue.svg)

## 📋 Daftar Isi
1. [Introduction](#-introduction-)
2. [Getting Started](#-getting-started)
3. [Architecture & Folder Structure](#-architecture--folder-structure)
4. [API Documentation](#-api-documentation)
5. [Database Schema](#-database-schema)

---

## 👋 Introduction 

Selamat datang di dokumentasi teknis **Job Architecture Management API**.

### 📖 Tentang Aplikasi
**Job Architecture Management API** adalah layanan backend RESTful yang kuat dan terukur, dirancang khusus sebagai modul inti dalam ekosistem *Human Resource Information System* (HRIS). Tujuan utama aplikasi ini adalah menangani kompleksitas struktur organisasi perusahaan, mulai dari definisi jabatan, hierarki pelaporan, hingga spesifikasi kualifikasi teknis karyawan.

Baik Anda seorang *Backend Developer* yang ingin melakukan integrasi sistem, atau *System Assessor* yang sedang meninjau arsitektur kode, dokumentasi ini adalah panduan komprehensif untuk memahami logika bisnis dan struktur data yang dibangun di dalamnya.

### 🌟 Key Features
Aplikasi ini dilengkapi dengan fitur-fitur esensial untuk manajemen organisasi modern. Berikut adalah fitur unggulan berdasarkan *codebase* yang tersedia:

* **Advanced Job Hierarchy Management**
    Sistem ini tidak berdiri sendiri, melainkan terintegrasi penuh dengan entitas organisasi lainnya. Aplikasi memetakan relasi yang presisi antara Posisi (`JobPosition`), Divisi (`Division`), Level Jabatan (`JobLevel`), serta garis komando ke Atasan (`Superior`) untuk merefleksikan struktur perusahaan yang nyata.

* **Dynamic Qualification Specification**
    Berbeda dengan sistem statis, aplikasi ini menangani persyaratan jabatan secara fleksibel. Menggunakan kombinasi tabel relasional untuk data terstruktur (Pendidikan, Masa Kerja) dan tipe data `JSONB` untuk deskripsi yang dinamis, memungkinkan penyimpanan data yang adaptif tanpa mengorbankan performa query.

* **Transactional Data Integrity**
    Keamanan dan konsistensi data adalah prioritas. Operasi pembaruan data yang kompleks, seperti penyuntingan persyaratan jabatan (`JobPosition_Requirement`), dibungkus dalam *Database Transactions* (`sequelize.transaction`). Hal ini menjamin bahwa data tetap konsisten dan valid bahkan jika terjadi gangguan di tengah proses penyimpanan.

### 🎯 Who Should Use This Documentation
Dokumentasi ini ditujukan untuk audiens teknis berikut:

* **Backend Developers:** Panduan ini membantu Anda memahami *endpoints*, *payload*, dan logika bisnis untuk pengembangan lebih lanjut atau integrasi dengan frontend.
* **System Architects / Assessors:** Jika Anda sedang mengevaluasi kualitas kode dan desain sistem, bagian *Architecture* dan *Database Schema* akan memberikan wawasan mendalam mengenai pola desain (MVC) dan skema ERD yang diterapkan.
* **Database Administrators:** Untuk memahami migrasi skema dan relasi antar tabel yang digunakan dalam sistem ini.

### 🧭 How to Use This Documentation
Untuk memaksimalkan penggunaan dokumentasi ini, Anda dapat mengikuti panduan berikut:

* **Getting Started:** Mulailah dari sini untuk instruksi instalasi, konfigurasi *environment*, dan menjalankan server lokal.
* **API Documentation:** Lihat bagian ini untuk daftar lengkap *endpoints* HTTP, parameter request, dan format respons JSON.
* **Architecture & Schema:** Pelajari bagian ini untuk memahami bagaimana data mengalir dari *Route* ke *Controller* hingga ke *Database* (PostgreSQL).


---
*Dokumentasi ini dibuat sebagai bagian dari Laporan Teknis Pengembangan Backend.*

## 🎯 Getting Started

Bagian ini memandu Anda untuk menyiapkan lingkungan pengembangan lokal (*local environment*) agar **Job Architecture Management API** dapat berjalan dengan lancar.

### 🛠️ Prerequisites

Pastikan sistem Anda telah terinstal *tools* berikut sebelum melanjutkan:

* **Node.js** (v14.x atau lebih baru) - [Download Node.js](https://nodejs.org/)
* **npm** atau **yarn** (Package Manager)
* **PostgreSQL** (v12.x atau lebih baru) - [Download PostgreSQL](https://www.postgresql.org/)
* **Postman/Insomnia** (Untuk pengujian API endpoint)

### ⚙️ Installation

Ikuti langkah-langkah berikut untuk menginstal dependensi proyek:

1.  **Clone Repository**
    Salin kode sumber ke direktori lokal Anda:
    ```bash
    git clone [https://github.com/blpbeauty/itdevintern_test]
    cd itdevintern_tes
    ```

2.  **Install Dependencies**
    Instal seluruh paket yang dibutuhkan oleh Express dan Sequelize (termasuk driver `pg` dan `sequelize-cli`):
    ```bash
    npm install
    ```

### 🔐 Configuration

Aplikasi ini menggunakan **Environment Variables** untuk konfigurasi sensitif. 

1.  Duplikasi file `.env.example` menjadi `.env` di *root directory*.
2.  Sesuaikan kredensial di bawah ini dengan konfigurasi PostgreSQL lokal Anda:

```env
# Server Configuration
NODE_ENV=development
PORT=3000

# Database Configuration
DB_USERNAME=postgres
DB_PASSWORD=password_anda
DB_NAME=hris_job_architecture
DB_HOST=127.0.0.1
DB_DIALECT=postgres
```

### 🗄️ Database Setup & Migration
Karena modul ini memiliki struktur relasional yang kompleks (JobPosition, JobLevel, Requirements) dan menggunakan tipe data JSONB, Anda wajib menjalankan migrasi database.

1. Create Database Buat database baru sesuai nama di .env:
   ```bash
    npx sequelize-cli db:create
    ```
2. Run Migrations Eksekusi file migrasi untuk membuat tabel dan mengubah tipe kolom requirements menjadi JSONB:
    ```bash
    npx sequelize-cli db:migrate
    ```
    Note: Langkah ini krusial untuk memastikan skema tabel sesuai dengan definisi model di jobposition.js dan jobpositionrequirement.js.

### 🚀 Running the App
Setelah konfigurasi selesai, jalankan server dengan perintah berikut:
Development Mode (dengan Hot-Reload):
1. Development Mode (dengan Hot-Reload):
   ```bash
    npm run dev
    ```
2. Production Mode:
    ```bash
    npm start
    ```
3. Jika berhasil, terminal akan menampilkan output:
    ```Plaintext
    Server is running on port 3000
    Database connected successfully.
    ```

## 📐 Architecture

Bagian ini menjelaskan desain teknis sistem. Aplikasi ini dibangun menggunakan pola arsitektur **Layered MVC (Model-View-Controller)** yang diperkuat dengan **Repository Pattern** untuk memisahkan logika bisnis dari akses data.

Pendekatan ini dipilih untuk memastikan kode yang *modular*, mudah diuji (*testable*), dan mudah dipelihara (*maintainable*).

### High-Level Overview

Sistem backend ini beroperasi sebagai **RESTful API Service**. Alur data dirancang untuk menangani integritas relasional yang kompleks antara struktur organisasi (Divisi, Jabatan) dan spesifikasi kualifikasi.

* **Runtime:** Node.js environment.
* **Framework:** Express.js untuk manajemen routing dan middleware.
* **Database Abstraction:** Sequelize ORM untuk pemetaan objek ke database PostgreSQL.
* **Architecture Style:** Monolithic Modular (diorganisir berdasarkan domain fitur).

### 📂 Project Structure

Berdasarkan implementasi kode, berikut adalah struktur direktori utama proyek ini:

```text
src/
├── controllers/          # Business Logic Layer
│   └── jobposition.controllers.js  # Mengelola request & response
├── models/               # Data Definition Layer (Sequelize Models)
│   ├── jobposition.js    # Schema tabel JobPositions
│   └── jobpositionrequirement.js
├── repositories/         # Data Access Layer
│   └── jobpositionrequirements.repository.js # Abstraksi query database
├── routes/               # API Routing Layer
│   └── jobposition.routes.js       # Definisi endpoint HTTP
├── migrations/           # Database Schema Versioning
│   └── change-jobposition-array.js
│   └── create-job-position.js
│   └── create-jobposition-requirement.js
```

### 🏗️ Design Patterns
Sistem ini menerapkan beberapa Design Pattern industri untuk menjaga kualitas kode:

1.  MVC (Model-View-Controller) Pemisahan tanggung jawab yang jelas:

    * Routes menangani entry point HTTP.
    
    * Controllers menangani validasi input dan orkestrasi logika bisnis.
    
    * Models mendefinisikan struktur data dan relasi tabel.

2.  Repository Pattern Alih-alih mengakses database langsung dari Controller untuk entitas yang kompleks, sistem menggunakan Repository (lihat jobpositionrequirements.repository.js). Ini berfungsi sebagai abstraction layer untuk operasi database spesifik, membuat kode lebih rapi dan dapat digunakan kembali (reusable).

3.  Transactional Operation (ACID) Untuk operasi yang melibatkan penulisan ke beberapa tabel sekaligus (misalnya: update Jabatan sekaligus Kualifikasinya), sistem menggunakan sequelize.transaction. Ini menjamin integritas data; jika satu proses gagal, seluruh perubahan akan dibatalkan (rollback).

### 🔄 Data Flow

Berikut adalah alur perjalanan data (*Request Lifecycle*) dalam sistem ini:

1. **Client Request:** Pengguna mengirim request HTTP (misal: `PUT /job-position/:id`).
2. **Route Layer:** `jobposition.routes.js` menerima request dan meneruskannya ke middleware yang relevan (jika ada).
3. **Controller Layer:** `jobposition.controllers.js` menerima data, memvalidasi input (`req.body`), dan memanggil logika bisnis.
4. **Repository/Model Layer:**
* Controller memanggil `JobPosition.update()` untuk data utama.
* Controller memanggil `jobpositionrequirementsRepository.updateByJobPositionId()` untuk data kualifikasi.


5. **Database:** Sequelize menerjemahkan perintah ke SQL query dan mengeksekusinya di PostgreSQL secara transaksional.
6. **Response:** Server mengembalikan respon JSON standar (`res.sendJson`) ke klien.

---


## 💻 API Documentation

Dokumentasi ini menjelaskan antarmuka pemrograman aplikasi (API) untuk modul **Job Position**. Seluruh endpoint menggunakan standar RESTful dan mengembalikan respons dalam format JSON.

### Base URL
```text
http://localhost:3000/api/v1/job-positions

```

*(Catatan: Base URL dapat bervariasi tergantung pada konfigurasi prefix di `app.js` utama Anda)*

### Authentication

Akses ke endpoint ini dilindungi oleh middleware otentikasi. Anda harus menyertakan token JWT pada header setiap permintaan.

**Header Example:**

```http
Authorization: Bearer <YOUR_ACCESS_TOKEN>
Content-Type: application/json

```

### 📋 Endpoints Summary

| Method | Endpoint | Deskripsi |
| --- | --- | --- |
| `GET` | `/` | Mengambil seluruh data jabatan beserta relasinya. |
| `GET` | `/:id` | Mengambil detail jabatan spesifik berdasarkan ID. |
| `POST` | `/` | Membuat data jabatan baru. |
| `PUT` | `/:id` | Memperbarui jabatan dan persyaratan kualifikasi (Transactional). |
| `DELETE` | `/` | Menghapus data jabatan (Soft delete atau Hard delete tergantung konfigurasi). |

---

### 📝 Request & Response Examples

Berikut adalah detail *payload* dan struktur respons berdasarkan logika pada Controller.

#### 1. Create Job Position

Membuat posisi baru dengan detail hierarki dan deskripsi.

* **URL:** `/`
* **Method:** `POST`
* **Body Parameters:**

```json
{
    "title": "Senior Backend Engineer",
    "joblevel_id": 3,
    "division_id": 2,
    "superior_id": 10,
    "purpose": "Bertanggung jawab atas arsitektur server.",
    "requirements": ["Node.js", "PostgreSQL", "Docker"],  // Disimpan sebagai JSONB
    "descriptions": ["Mengembangkan API", "Code Review"]    // Disimpan sebagai JSONB
}

```

* **Success Response (201 Created):**

```json
{
    "code": 201,
    "status": true,
    "message": "Success create a new Job Position and its Description",
    "data": {
        "id": 15,
        "title": "Senior Backend Engineer",
        "descriptions": ["Mengembangkan API", "Code Review"],
        "createdAt": "2023-10-27T08:00:00.000Z",
        "updatedAt": "2023-10-27T08:00:00.000Z"
    }
}

```

#### 2. Get Job Detail

Mengambil data jabatan lengkap dengan relasi `Division`, `JobLevel`, `Superior`, dan `JobPosition_Requirement`.

* **URL:** `/:id` (Contoh: `/15`)
* **Method:** `GET`
* **Success Response (200 OK):**

```json
{
    "code": 200,
    "status": true,
    "message": "success find data",
    "data": {
        "id": 15,
        "title": "Senior Backend Engineer",
        "joblevel_name": "Manager",
        "division_name": "IT Development",
        "superior_name": "John Doe",
        "purpose": "Bertanggung jawab atas arsitektur server.",
        "requirements": ["Node.js", "PostgreSQL"],
        "descriptions": ["Mengembangkan API"],
        "careerForm": {
            "education": "S1 Teknik Informatika",
            "length_service": 2,
            "performance": "A"
        }
    }
}

```

#### 3. Update Job Position

Endpoint ini menangani **Complex Update** menggunakan database transaction. Payload mencakup data jabatan utama dan data `careerForm` (persyaratan teknis).

* **URL:** `/:id`
* **Method:** `PUT`
* **Body Parameters:**

```json
{
    "title": "Lead Backend Engineer",
    "joblevel_id": 4,
    "division_id": 2,
    "superior_id": 10,
    "purpose": "Memimpin tim backend.",
    "requirements": ["Node.js", "Microservices", "System Design"],
    "descriptions": ["Mentoring", "Architecture Decision"],
    "careerForm": {
        "education": "S2 Teknik Informatika",
        "length_service": 5,
        "performance": "A"
    }
}

```

> **Note:** Object `careerForm` akan diproses oleh `jobpositionrequirementsRepository` untuk membuat atau memperbarui data di tabel `JobPosition_Requirements` secara otomatis.

#### 4. Delete Job Position

Menghapus data jabatan. Berdasarkan implementasi controller saat ini, ID dikirimkan melalui **Body**, bukan URL parameter.

* **URL:** `/`
* **Method:** `DELETE`
* **Body Parameters:**

```json
{
    "id": 15
}

```

*(Catatan Teknis: Endpoint ini menggunakan `req.body.id` sesuai implementasi pada `jobposition.controllers.js`, meskipun praktik umum REST biasanya menggunakan URL Parameter)*

## 💽 Database Schema

Bagian ini memberikan dokumentasi mendalam mengenai struktur basis data yang digunakan dalam **Job Architecture Management API**. Sistem ini dibangun di atas **PostgreSQL** dan dikelola menggunakan **Sequelize ORM**, memanfaatkan fitur relasional standar serta kapabilitas penyimpanan dokumen (NoSQL) untuk fleksibilitas data.

### 📊 Entity Relationship Diagram (ERD)

Diagram berikut memvisualisasikan hubungan antar entitas, termasuk kardinalitas dan arah relasi:

![ERD Job Architecture](ERD%20IT%20DevIntern.drawio.png)

---

### 📋 Detailed Table Structures

Berikut adalah spesifikasi teknis untuk setiap tabel utama dalam modul ini.

#### 1. Table: `JobPositions`
Tabel entitas utama yang merepresentasikan definisi jabatan dalam struktur organisasi.

| Column Name | Data Type | Constraints / Properties | Description |
| :--- | :--- | :--- | :--- |
| **id** | `INTEGER` | `PK`, `AutoIncrement` | Identifikasi unik untuk setiap jabatan. |
| **title** | `STRING` | `UNIQUE`, `AllowNull: false` | Nama resmi jabatan. Constraint `Unique` mencegah duplikasi nama jabatan dalam perusahaan. |
| **joblevel_id** | `INTEGER` | `FK` (ref: `JobLevel`) | Menunjukkan tingkatan/grade dari jabatan ini (misal: Staff, Manager). |
| **division_id** | `INTEGER` | `FK` (ref: `Division`) | Menunjukkan departemen atau divisi tempat jabatan ini bernaung. |
| **superior_id** | `INTEGER` | `FK` (ref: `User`), `Default: null` | ID User yang menjadi atasan langsung (*reporting line*). Bernilai `null` jika posisi tertinggi. |
| **purpose** | `STRING` | `AllowNull: true` | Penjelasan singkat mengenai tujuan utama keberadaan posisi ini. |
| **requirements** | `JSONB` | `Default: []` | **Hybrid Column**. Menyimpan daftar kualifikasi umum dalam format JSON Array. Menggantikan tipe `ARRAY` lama untuk performa query yang lebih baik di PostgreSQL. |
| **descriptions** | `JSONB` | `Default: []` | **Hybrid Column**. Menyimpan daftar tanggung jawab (*Job Desc*) dalam format JSON Array. |
| **createdAt** | `DATE` | `Not Null` | Timestamp otomatis saat data dibuat. |
| **updatedAt** | `DATE` | `Not Null` | Timestamp otomatis saat data diperbarui. |

#### 2. Table: `JobPosition_Requirements`
Tabel ini menyimpan parameter kualifikasi yang terukur (kuantitatif) untuk keperluan validasi kandidat atau promosi.

| Column Name | Data Type | Constraints / Properties | Description |
| :--- | :--- | :--- | :--- |
| **id** | `INTEGER` | `PK`, `AutoIncrement` | Identifikasi unik record requirement. |
| **jobposition_id** | `INTEGER` | `FK` (ref: `JobPositions`), `Not Null` | Relasi ke jabatan induk. Memiliki sifat `ON DELETE CASCADE`. |
| **education** | `STRING` | `AllowNull: true` | Persyaratan tingkat pendidikan minimal (contoh: "S1 Teknik Informatika"). |
| **length_service**| `INTEGER` | `AllowNull: true` | Persyaratan masa kerja minimal dalam satuan tahun (contoh: 2). |
| **performance** | `STRING` | `AllowNull: true` | Persyaratan nilai performa minimal (contoh: "A" atau "Exceeds Expectations"). |

---

### 🔗 Key Relationships & Associations

Aplikasi ini menggunakan asosiasi Sequelize untuk menegakkan integritas referensial antar data. Berikut adalah pemetaan detailnya:

#### Hierarchical Relationships (Relasi Hierarki)
* **JobPosition belongsTo Division**
    * **Foreign Key:** `division_id`
    * **Deskripsi:** Setiap jabatan wajib terhubung ke satu divisi.
* **JobPosition belongsTo JobLevel**
    * **Foreign Key:** `joblevel_id`
    * **Deskripsi:** Setiap jabatan memiliki satu level tingkatan.
* **JobPosition belongsTo User (as 'superior')**
    * **Foreign Key:** `superior_id`
    * **Alias:** `superior`
    * **Deskripsi:** Relasi *Self-Reference* (ke tabel User) untuk menentukan struktur pelaporan kepada atasan.

#### Specification Relationships (Relasi Spesifikasi)
* **JobPosition hasMany JobPosition_Requirement**
    * **Foreign Key:** `jobposition_id`
    * **Alias:** `jobposition_req`
    * **Behavior:** `ON UPDATE CASCADE`, `ON DELETE CASCADE`
    * **Deskripsi:** Relasi 1-to-N. Jika sebuah `JobPosition` dihapus, maka seluruh data `Requirement` spesifik yang melekat padanya akan otomatis terhapus untuk mencegah data sampah (*orphan data*).

#### Operational Relationships (Relasi Operasional)
* **JobPosition hasMany User**
    * **Foreign Key:** `jobposition_id`
    * **Deskripsi:** Satu jabatan dapat diduduki oleh banyak karyawan (User).
* **JobPosition hasMany NumberRequest**
    * **Foreign Key:** `signer_id`
    * **Deskripsi:** Jabatan tertentu memiliki otoritas untuk menandatangani permintaan penomoran dokumen.

---

### 💡 Design Decisions Highlights

#### 1. Penggunaan JSONB untuk Fleksibilitas
Pada migrasi `change-jobposition-array.js`, kolom `requirements` dan `descriptions` diubah dari tipe `ARRAY` menjadi `JSONB`.
* **Alasan:** Format `JSONB` di PostgreSQL memungkinkan penyimpanan data semi-terstruktur yang lebih efisien, mendukung *indexing* pada elemen array, dan memudahkan perubahan struktur data kualifikasi di masa depan tanpa perlu mengubah skema tabel utama secara drastis.

#### 2. Pemisahan Tabel Requirement
Data kualifikasi dipisah menjadi dua:
1.  **Deskriptif:** Disimpan di kolom JSONB `JobPositions` (untuk poin-poin umum).
2.  **Terukur:** Disimpan di tabel `JobPosition_Requirements` (untuk Pendidikan, Masa Kerja, Performa).
* **Alasan:** Pemisahan ini memungkinkan sistem melakukan *query filtering* yang presisi (misal: "Cari jabatan yang butuh pengalaman > 2 tahun") pada tabel relasional, sementara tetap menyimpan deskripsi kualitatif yang panjang di JSONB.

