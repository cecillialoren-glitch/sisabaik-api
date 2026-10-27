# Dokumentasi API SisaBaik (Chapter 8 - Autentikasi & Otorisasi)

Dokumen ini menjelaskan spesifikasi endpoint, struktur payload, batas laju (*rate limiting*), serta mekanisme otorisasi pada SisaBaik API.

---

## 1. Kebijakan Katalog Publik & Akses Anonim

- **Akses Publik (Tanpa Token)**:
  - Katalog penawaran yang aktif (`GET /api/penawaran`) dapat diakses secara terbuka tanpa perlu menyertakan header `Authorization`.
  - Halaman latihan dan aset statis (`/auth.html`, `/auth.css`, `/auth.js`) dapat diakses secara anonim.
- **Akses Terproteksi (Wajib Token JWT)**:
  - Seluruh endpoint manajemen profil, transaksi pesanan, manajemen penawaran milik penyedia, serta sesi akun memerlukan header HTTP:
    `Authorization: Bearer <access_token>`

---

## 2. Batas Laju Request (*Rate Limiting*)

Sistem menerapkan batas laju request terpisah berdasarkan kategori endpoint untuk melindungi API dari penyalahgunaan dan serangan brute-force:

| Kategori Endpoint | Batas Laju | Jendela Waktu (*Window*) | Respon Saat Melebihi Batas |
| :--- | :--- | :--- | :--- |
| **Autentikasi sensitif** (`/login`, `/daftar/*`) | Maks. 20 request | 15 menit | `429 Too Many Requests` + Header `Retry-After` |
| **API Umum / Operasional** (`/penawaran`, `/pesanan`, dll) | Maks. 100 request | 15 menit | `429 Too Many Requests` |

---

## 3. Endpoint Autentikasi & Registrasi (`/api/auth`)

### 3.1 Registrasi Pembeli
- **URL**: `POST /api/auth/daftar/pembeli`
- **Akses**: Publik
- **Request Body**:
  ```json
  {
    "namaLengkap": "Budi Santoso",
    "email": "budi@example.com",
    "nomorTelepon": "081234567890",
    "kataSandi": "SandiKuat123!"
  }
  ```
- **Respon Sukses (`201 Created`)**:
  ```json
  {
    "status": "success",
    "message": "Pendaftaran pembeli berhasil.",
    "data": {
      "id": "usr-1234-abcd",
      "namaLengkap": "Budi Santoso",
      "email": "budi@example.com",
      "peran": "PEMBELI"
    }
  }
  ```

---

### 3.2 Registrasi Penyedia (Beserta Profil Usaha)
- **URL**: `POST /api/auth/daftar/penyedia`
- **Akses**: Publik
- **Request Body**:
  ```json
  {
    "namaLengkap": "Siti Rahma",
    "email": "siti@bakerymantap.com",
    "nomorTelepon": "089876543210",
    "kataSandi": "SandiPenyedia123!",
    "bisnis": {
      "namaUsaha": "Bakery Mantap",
      "deskripsi": "Toko roti dan kue segar harian",
      "alamat": "Jl. Merdeka No. 45, Padang",
      "jenisUsaha": "Bakery"
    }
  }
  ```
- **Respon Sukses (`201 Created`)**:
  ```json
  {
    "status": "success",
    "message": "Pendaftaran penyedia berhasil diproses. Menunggu persetujuan admin.",
    "data": {
      "id": "usr-5678-efgh",
      "namaLengkap": "Siti Rahma",
      "email": "siti@bakerymantap.com",
      "peran": "PENYEDIA",
      "statusAkun": "MENUNGGU_PERSETUJUAN",
      "bisnis": {
        "id": "biz-9012-ijkl",
        "namaUsaha": "Bakery Mantap",
        "jenisUsaha": "Bakery"
      }
    }
  }
  ```

---

### 3.3 Login / Masuk
- **URL**: `POST /api/auth/login`
- **Request Body**:
  ```json
  {
    "email": "budi@example.com",
    "kataSandi": "SandiKuat123!"
  }
  ```
- **Respon Sukses (`200 OK`)**:
  ```json
  {
    "status": "success",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "pengguna": {
        "id": "usr-1234-abcd",
        "namaLengkap": "Budi Santoso",
        "peran": "PEMBELI"
      }
    }
  }
  ```

---

### 3.4 Sesi Pengguna
- `GET /api/auth/saya` — Mendapatkan profil pengguna aktif.
- `POST /api/auth/logout` — Keluar dari sesi saat ini.
- `POST /api/auth/keluar-semua` — Keluar dari seluruh sesi (revok semua token aktif).

---

## 4. Cakupan Pesanan & Otorisasi Berbasis Peran/Kepemilikan

- **Peran `PEMBELI`**:
  - Hanya dapat membuat pesanan (`POST /api/pesanan`).
  - Hanya dapat melihat daftar pesanan yang ia buat sendiri (`GET /api/pesanan`).
  - Jika mencoba mengakses pesanan milik pembeli lain, API akan mengembalikan status `404 Not Found` (mencegah enumerasi ID).
- **Peran `PENYEDIA`**:
  - Hanya dapat mengelola (tambah, ubah, hapus) penawaran toko miliknya sendiri.
  - Hanya dapat melihat pesanan yang masuk untuk penawaran miliknya.
  - Jika mencoba membuat pesanan atau mengakses data toko lain, API mengembalikan `403 Forbidden` atau `404 Not Found`.

---

## 5. Standar Format Error & Status HTTP

Seluruh respon error menggunakan format konsisten:

```json
{
  "status": "fail",
  "error": {
    "code": "KODE_ERROR",
    "message": "Pesan deskripsi kesalahan.",
    "details": []
  }
}
```

### Ringkasan Status Code HTTP:

| Status Code | Kode Error (*Code*) | Keterangan / Penyebab |
| :--- | :--- | :--- |
| `400 Bad Request` | `VALIDATION_ERROR` | Format body JSON tidak sesuai skema validasi. |
| `401 Unauthorized` | `UNAUTHENTICATED` / `SESSION_REVOKED` | Token JWT tidak menyertai header, kadaluwarsa, atau telah direvok (*logout*). |
| `403 Forbidden` | `FORBIDDEN` / `PROVIDER_NOT_APPROVED` | Peran pengguna tidak diizinkan, atau akun penyedia belum disetujui. |
| `404 Not Found` | `NOT_FOUND` | Resource tidak ditemukan atau pengguna mencoba mengakses data milik pengguna lain. |
| `429 Too Many Requests` | `RATE_LIMIT_EXCEEDED` | Batas jumlah request telah terlampaui. |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | Kesalahan server internal. |