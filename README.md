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


