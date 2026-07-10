# AGENTS.md

Konvensi global untuk pengembangan **Decoupled Multi-Tenant ERP System**. Referensi requirement lengkap: `PRD.md`. Referensi API contract: `docs/API_CONTRACT.md`.

---

## 1. Tech Stack

|Layer|Teknologi|
|---|---|
|Backend|NestJS (TypeScript), Prisma, PostgreSQL, Redis|
|Frontend|Next.js (App Router, TypeScript), Tailwind CSS, Shadcn/ui, Zustand|
|Auth|JWT via HTTP-Only Cookie, refresh token di Redis|
|Multi-Tenancy|Shared Table + `tenant_id` + PostgreSQL RLS, resolusi via subdomain|

---

## 2. Environment Development

- Service Docker aktif: `postgres`, `redis` saja (`docker-compose.yml` di root).
- Backend dan frontend dijalankan native di host, bukan di container.
- Environment variable backend mengarah ke port container yang di-expose ke host (`localhost:5432`, `localhost:6379`).

---

## 3. Struktur Folder

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

- `docs/` berisi dokumen referensi teknis (`API_CONTRACT.md`, dokumen lain sesuai kebutuhan), di-load on-demand melalui `@path` syntax, bukan dibaca penuh di awal task.
- `frontend/` dan `backend/` adalah project independen, masing-masing dengan `package.json` dan `.env` sendiri.

---

## 4. Konvensi Backend (NestJS)

- Pattern layer: Controller → Service → Repository. Query database tidak boleh langsung di Controller atau Service.
- Tenant context: `AsyncLocalStorage` menyimpan `tenant_id` per request, di-set melalui middleware yang membaca subdomain.
- Setiap query melalui Prisma Client Extension yang menjalankan `SET LOCAL app.current_tenant_id` dalam transaction sama dengan query utama.
- RBAC: guard `@Roles()` dan `@Permissions()` pada level Controller method, tidak ada authorization check manual di dalam Service.
- Mutasi stok: wajib dalam database transaction dengan `SELECT ... FOR UPDATE` pada baris terkait.
- Invoice dan jurnal akuntansi: dibuat dalam satu database transaction, tidak menggunakan Event Emitter untuk proses ini.
- Response format: mengikuti `docs/API_CONTRACT.md`, tidak boleh menyimpang tanpa update dokumen tersebut terlebih dahulu.

---

## 5. Konvensi Frontend (Next.js)

- Resolusi tenant: subdomain middleware, tidak menggunakan dynamic route `[tenantId]`.
- State management global: Zustand. State lokal komponen tetap pakai `useState`/`useReducer`.
- Komponen UI: Shadcn/ui sebagai basis, kustomisasi lewat Tailwind, tidak membuat komponen UI dasar dari nol jika varian Shadcn/ui tersedia.

---

## 6. Git Workflow

- Commit dilakukan otomatis oleh agent tanpa perlu persetujuan eksplisit per commit.
- Push ke remote wajib persetujuan eksplisit dari user.
- Sebelum eksekusi task yang mengubah branch, agent konfirmasi nama branch ke user terlebih dahulu.
- Format commit: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`).
- Prettier dijalankan sebelum setiap commit.

---

## 7. Token Efficiency

- Baca hanya file yang relevan dengan task berjalan, bukan seluruh isi `docs/`.
- Dokumen referensi (`API_CONTRACT.md`, `PRD.md`) di-load via `@path` sesuai kebutuhan step, bukan di-load penuh di awal sesi.

---

## 8. Batasan Eksekusi

- Implementasi mengikuti tahapan pada `INSTRUCTION.md`, tidak boleh melompati fase tanpa instruksi atau persetujuan eksplisit dari user.
- Perubahan pada keputusan arsitektur (multi-tenancy strategy, ORM, format response, dsb.) wajib update `PRD.md` terlebih dahulu sebelum implementasi menyesuaikan.

## Referensi Dokumen

- Requirement lengkap: @PRD.md
- API contract & response format: @docs/API_CONTRACT.md
- Tahapan implementasi: @.agents/workflows/INSTRUCTION.md