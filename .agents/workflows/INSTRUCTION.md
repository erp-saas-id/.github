# INSTRUCTION.md

Tahapan implementasi **Decoupled Multi-Tenant ERP System**. Setiap fase dieksekusi per step, tidak boleh lompat ke step berikutnya tanpa instruksi atau persetujuan eksplisit dari user. Referensi konvensi: `.agents/AGENTS.md`. Referensi requirement: `PRD.md`.

---

## Fase 1 — Backend Foundation & Multi-Tenancy

1. Inisialisasi project NestJS (`backend/`).
2. Buat `docker-compose.yml` di root dengan service `postgres` dan `redis`.
3. Setup Prisma, definisikan schema awal: `tenants`, `users`, `roles`, `permissions`, `role_permissions`, `user_roles`.
4. Terapkan RLS policy pada tabel yang memiliki kolom `tenant_id`.
5. Implementasi middleware resolusi tenant dari subdomain request.
6. Implementasi `AsyncLocalStorage` untuk menyimpan `tenant_id` per request.
7. Implementasi Prisma Client Extension untuk `SET LOCAL app.current_tenant_id` per transaction.
8. Verifikasi isolasi data: uji query dari dua tenant berbeda memastikan tidak ada kebocoran data.

---

## Fase 2 — Auth & RBAC

1. Modul registrasi tenant baru: buat tenant, user pertama dengan role `Owner`, seed default Chart of Accounts.
2. Modul login: generate JWT, simpan di HTTP-Only Cookie, simpan refresh token di Redis.
3. Modul logout: hapus refresh token dari Redis.
4. Implementasi guard `@Roles()` dan `@Permissions()`.
5. Implementasi cache profil tenant di Redis (invalidasi saat data tenant berubah).
6. Standarisasi response sesuai `docs/API_CONTRACT.md` pada seluruh endpoint modul ini.

---

## Fase 3 — Inventaris & Frontend Dasar

1. Modul Produk (SKU): CRUD produk, harga beli, harga jual, stok minimum.
2. Modul Multi-Warehouse: CRUD gudang, stok per produk per gudang.
3. Modul Mutasi Stok: transaction + row lock (`SELECT ... FOR UPDATE`) pada operasi masuk/keluar/pindah gudang.
4. Inisialisasi project Next.js (`frontend/`).
5. Implementasi subdomain middleware untuk resolusi tenant di frontend.
6. UI dasar: halaman login, dashboard, daftar produk, daftar gudang.

---

## Fase 4 — Penjualan, Akuntansi, Integrasi Akhir

1. Modul Invoice Penjualan.
2. Otomatisasi jurnal akuntansi dalam transaction sama dengan pembuatan invoice.
3. Modul Chart of Accounts: tambah/ubah akun manual oleh Owner/Manager.
4. Standarisasi error response dan audit log (`audit_log` table, soft-delete pada entity transaksional) di seluruh endpoint.
5. Verifikasi end-to-end: registrasi tenant → login → transaksi inventaris → invoice → jurnal otomatis.

---

## Aturan Eksekusi

- Setiap step dikonfirmasi selesai sebelum lanjut ke step berikutnya dalam fase yang sama.
- Perpindahan antar fase membutuhkan persetujuan eksplisit user.
- Commit dilakukan otomatis per step (mengikuti `.agents/AGENTS.md` Bagian 6), push membutuhkan persetujuan eksplisit.
- Jika step membutuhkan keputusan arsitektur yang belum tercantum di `PRD.md`, hentikan eksekusi dan tanyakan ke user terlebih dahulu.