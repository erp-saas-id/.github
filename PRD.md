# Product Requirement Document (PRD)
## Projek: Decoupled Multi-Tenant ERP System

---

## 1. Ringkasan Projek

Sistem ERP berbasis Software-as-a-Service (SaaS) multi-tenant. Satu instance backend dan frontend melayani banyak tenant (perusahaan) dengan isolasi data pada level baris (row-level), dijaga oleh PostgreSQL Row-Level Security (RLS). Sistem mencakup tiga modul: Manajemen Pengguna & Tenant, Inventaris, dan Akuntansi Dasar dengan pencatatan jurnal otomatis.

---

## 2. Spesifikasi Teknologi

|Layer|Teknologi|
|---|---|
|Backend API|NestJS (TypeScript)|
|Database|PostgreSQL dengan Row-Level Security|
|ORM|Prisma|
|Cache & Token Store|Redis|
|Frontend|Next.js (App Router, TypeScript)|
|UI|Tailwind CSS, Shadcn/ui|
|State Management|Zustand|
|Komunikasi|REST API, JWT via HTTP-Only Cookie|

---

## 3. Arsitektur Multi-Tenancy

**Strategi: Shared Database, Shared Table dengan kolom `tenant_id` + PostgreSQL Row-Level Security (RLS).**

Ketentuan:

- Setiap tabel yang menyimpan data milik tenant wajib memiliki kolom `tenant_id` (foreign key ke tabel `tenants`).
- RLS policy diterapkan pada setiap tabel tersebut: `USING (tenant_id = current_setting('app.current_tenant_id')::uuid)`.
- Resolusi tenant dilakukan melalui middleware yang membaca subdomain request (lihat Bagian 5).
- Nilai `tenant_id` disimpan dalam request-scoped context menggunakan NestJS `AsyncLocalStorage`, kemudian di-set ke session Postgres (`SET app.current_tenant_id`) pada awal setiap request melalui Prisma middleware/extension sebelum query dijalankan.
- Tidak ada migration schema per tenant. Tenant baru hanya menghasilkan baris baru di tabel `tenants` beserta data awal (lihat Bagian 4, Modul 1).

**ORM: Prisma.**

- Prisma Client Extension digunakan untuk menyisipkan query interceptor yang menjalankan `SET LOCAL app.current_tenant_id` dalam transaction yang sama dengan query utama, menjamin isolasi tanpa kebocoran context antar request.

---

## 4. Kebutuhan Fungsional
### Modul 1: Manajemen Tenant & Autentikasi (RBAC)

- **Registrasi Tenant Baru**: membuat baris baru pada tabel `tenants`, membuat user pertama dengan role `Owner`, dan menjalankan seed default Chart of Accounts (Kas, Piutang Usaha, Pendapatan, Persediaan).
- **RBAC (Role-Permission Model)**: relasi many-to-many antara `roles` dan `permissions`. Tiga role default: Owner, Manager, Staff, masing-masing dengan set permission berbeda per modul (contoh: `inventory.create`, `invoice.create`, `report.view`).
- Endpoint API dijaga menggunakan custom guard `@Roles()` dan `@Permissions()`, dievaluasi terhadap permission user yang tersimpan dalam JWT payload atau di-fetch dari Redis cache.
- **Autentikasi**: login menghasilkan JWT, disimpan di HTTP-Only Cookie. Refresh token disimpan di Redis dengan TTL, mendukung revocation (logout memutus refresh token dari Redis).
- Login dan resource lain terisolasi per tenant berdasarkan subdomain request.
### Modul 2: Manajemen Inventaris & Gudang

- **Manajemen Produk (SKU)**: input, ubah, hapus produk beserta harga beli, harga jual, dan stok minimum.
- **Multi-Warehouse Support**: pelacakan stok per lokasi gudang, satu produk dapat memiliki catatan stok berbeda di tiap gudang.
- **Mutasi Stok**: setiap barang masuk, keluar, atau pindah gudang tercatat di tabel `stock_movements`.
- **Mekanisme Atomik**: setiap mutasi stok dieksekusi dalam satu database transaction dengan row lock (`SELECT ... FOR UPDATE`) pada baris stok terkait, mencegah race condition saat update stok konkuren.
### Modul 3: Penjualan & Otomatisasi Buku Besar (Double-Entry Ledger)

- **Pembuatan Invoice Penjualan**: mencatat transaksi penjualan ke pelanggan.
- **Otomatisasi Akuntansi (Synchronous)**: entri jurnal (Debit Piutang/Kas, Kredit Pendapatan) dibuat dalam transaction database yang sama dengan pembuatan invoice. Invoice dan jurnal tercatat bersamaan atau tidak sama sekali (atomic, bukan eventual consistency).
- Chart of Accounts per tenant tersedia sejak registrasi (auto-seed), dapat ditambah manual oleh Owner/Manager melalui modul konfigurasi akun.

---

## 5. Kebutuhan Non-Fungsional

- **Resolusi Tenant di Frontend**: subdomain middleware. Format `{tenant-slug}.domain.com`. Next.js middleware membaca hostname, mengekstrak slug, dan menyertakan `X-Tenant-ID` (atau slug) pada setiap request ke backend.

- **Keamanan Lintas Domain**: CORS pada NestJS dikonfigurasi menerima origin dengan pola wildcard subdomain terdaftar (`*.domain.com`) beserta credential (cookie).

- **Performa & Caching**: Redis menyimpan cache profil tenant (menghindari query berulang ke tabel `tenants`) dan menyimpan refresh token untuk mendukung revocation. Redis tidak digunakan sebagai session store karena autentikasi bersifat stateless (JWT).

- **Audit Trail**: entity transaksional (invoice, stock_movements, journal_entries) menggunakan soft-delete (`deletedAt`). Tabel `audit_log` mencatat aktor, aksi, dan timestamp untuk mutasi stok dan invoice.

- **Standar Response Format**: format konsisten pada seluruh endpoint, detail lengkap pada `docs/API_CONTRACT.md`.

    Response sukses:

```
{
  "success": true,
  "data": {},
  "message": string (optional)
}
```

Response error:

```
{
  "success": false,
  "message": string,
  "errors": array | null,
  "timestamp": string,
  "path": string
}
```

Ketentuan:

    - HTTP status code menjadi source of truth di response header, tidak diduplikasi dalam body.
    - `errors` bertipe `array` untuk error validasi (per-field), atau `null` untuk error umum (unauthorized, forbidden, not found, internal error).
    - Nama exception class tidak dimasukkan ke body response, cukup dicatat pada server-side logging.

---

## 6. Environment Development

- **Database**: PostgreSQL dijalankan dalam Docker container (`docker-compose.yml` pada root project). Redis turut dijalankan sebagai container terpisah.
- **Aplikasi (backend & frontend)**: dijalankan native di host Windows (bukan di dalam Docker), untuk menghindari overhead build/start container di Windows. Environment variable (`DATABASE_URL`, `REDIS_URL`) pada backend mengarah ke port container yang di-expose ke host (contoh: `localhost:5432`, `localhost:6379`).
- Service yang aktif dalam `docker-compose.yml` terbatas pada: `postgres`, `redis`. Tidak ada service `app`/`api`/`web` dalam compose file.
## 7. Struktur Folder

```
root-project/
├── frontend/
├── backend/
├── docs/
├── .agents/
│   ├── AGENTS.md
│   └── workflows/
│       └── INSTRUCTION.md
├── PRD.md
└── docker-compose.yml
```

Ketentuan:

- `docker-compose.yml` berada di root, hanya mendefinisikan service `postgres` dan `redis`.
- `frontend/` dan `backend/` adalah project independen (masing-masing punya `package.json`, `.env` sendiri).
- `docs/` menyimpan dokumen referensi teknis yang di-load on-demand via `@path` syntax dari `AGENTS.md`.
- `.agents/AGENTS.md` menyimpan konvensi global project (git workflow, commit format, branch confirmation).
- `.agents/workflows/INSTRUCTION.md` menyimpan tahapan implementasi per fase.
## 8. Rencana Langkah Implementasi

1. **Fase 1 (Backend)**: inisialisasi NestJS, setup `docker-compose.yml` (postgres, redis), Prisma + PostgreSQL, implementasi RLS policy dan AsyncLocalStorage tenant context.
2. **Fase 2 (Backend)**: modul Auth, RBAC (Role-Permission), refresh token via Redis.
3. **Fase 3 (Backend & Frontend)**: modul Inventaris (SKU, multi-warehouse, mutasi stok atomik), setup Next.js subdomain middleware, UI dasar.
4. **Fase 4 (Backend & Frontend)**: modul Penjualan & Akuntansi (invoice + jurnal synchronous), integrasi akhir, standarisasi error response dan audit log.