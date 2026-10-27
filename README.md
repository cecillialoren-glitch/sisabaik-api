# SisaBaik API

SisaBaik API adalah layanan backend RESTful berbasis Node.js dan Express untuk platform penyelamatan makanan *SisaBaik*. Proyek ini menghubungkan penyedia makanan (seperti restoran, toko roti, atau supermarket) dengan pembeli untuk mengurangi sampah makanan melalui penjualan bahan atau porsi makanan dengan harga terjangkau.

---

## 1. Persyaratan Sistem & Dependensi

- **Node.js**: v18.x atau lebih baru
- **PostgreSQL**: v14.x atau lebih baru
- **Package Manager**: `npm`

---

## 2. Langkah Migrasi Database

Sebelum menjalankan server, pastikan database PostgreSQL lokal telah siap. Jalankan skrip migrasi secara berurutan:

1. `database/01_schema.sql` (Skema awal)
2. `database/02_seed.sql` (Data awal/dummy)
3. ...
4. `database/07_auth.sql` (Migrasi autentikasi & otorisasi)

> **Catatan Migrasi `07_auth.sql`**: Skrip ini menambahkan kolom `kata_sandi_hash` dan `versi_token` ke dalam tabel `pengguna` tanpa menghapus data yang sudah ada, serta menetapkan nilai awal versi token untuk seluruh pengguna yang ada.

Perintah eksekusi via `psql`:
```bash
psql -U postgres -d sisabaik_dev -f database/07_auth.sql
```

---

## 3. Konfigurasi Variable Environment (`.env`)

Buat file `.env` di direktori utama proyek berdasarkan templat `.env.example`. Jangan pernah menyertakan nilai kredensial/rahasia sebenarnya di dalam repositori kontrol versi.

Daftar nama variabel environment yang diperlukan:

```env
# Server Configuration
PORT=3000
NODE_ENV=development

# Database Configuration
PGHOST=localhost
PGPORT=5432
PGDATABASE=sisabaik_dev
PGUSER=postgres
PGPASSWORD=your_postgres_password

# Authentication & JWT Configuration
JWT_SECRET=your_jwt_secret_64_hex_characters
JWT_EXPIRES_IN=1h
```

> **Tips**: Anda dapat menghasilkan `JWT_SECRET` acak (64 digit hex) menggunakan Node.js:
> ```bash
> node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
> ```

---

## 4. Cara Jalankan Server & Halaman Latihan Auth

1. **Instalasi Dependensi**:
   ```bash
   npm install
   ```

2. **Jalankan Server dalam Mode Pengembangan**:
   ```bash
   npm run dev
   ```
   Server akan berjalan di `http://127.0.0.1:3000` atau `http://localhost:3000`.

3. **Akses Halaman Latihan Auth**:
   Buka peramban (browser) dan akses:
   `http://127.0.0.1:3000/auth.html`

---

## 5. Alur Alur Pendaftaran & Otorisasi Pengguna

* **Pendaftaran Mandiri**:
  * **Pembeli**: Mendaftar via `/api/auth/daftar/pembeli`. Akun langsung aktif dan dapat langsung digunakan untuk masuk (*login*) dan memesan penawaran.
  * **Penyedia**: Mendaftar via `/api/auth/daftar/penyedia` dengan melengkapi profil usaha (nama usaha, deskripsi, alamat, jenis usaha). Status awal akun penyedia adalah `MENUNGGU_PERSETUJUAN`.
* **Peninjauan Penyedia**:
  * Penyedia baru **tidak dapat masuk** sampai administrator menyetujui akunnya melalui database (mengubah status menjadi `AKTIF`).
* **Efek Logout Semua Sesi (Revocation)**:
  * Fitur keluar dari seluruh sesi (`/api/auth/keluar-semua`) akan menaikkan `versi_token` di tabel `pengguna`. Hal ini secara otomatis me-revok/membatalkan validitas **seluruh JWT Access Token** yang telah diterbitkan sebelumnya untuk akun tersebut.

---

## 6. Pengujian Otomatis (`auth:test`)

Proyek ini menyediakan skrip pengujian otomatis khusus untuk menguji alur autentikasi, otorisasi peran, validasi input, serta pembatasan akses.

Untuk menjalankan suite pengujian autentikasi:
```bash
npm run auth:test
```