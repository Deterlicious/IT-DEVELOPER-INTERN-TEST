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


