# ERP SaaS ID

Multi-tenant ERP SaaS untuk pasar Indonesia — manajemen inventaris, otomasi akuntansi, dan RBAC dalam satu platform.

## Arsitektur

- **Multi-Tenancy**: Shared Database, Shared Table dengan `tenant_id` + PostgreSQL Row-Level Security.
- **Backend**: NestJS (TypeScript), Prisma, PostgreSQL, Redis.
- **Frontend**: Next.js (App Router, TypeScript), Tailwind CSS, Shadcn/ui, Zustand.
- **Auth**: JWT via HTTP-Only Cookie, refresh token di Redis.
- Resolusi tenant melalui subdomain (`{tenant-slug}.domain.com`).

## Modul

- Manajemen Tenant & Autentikasi (RBAC berbasis Role-Permission).
- Inventaris & Multi-Warehouse dengan mutasi stok atomik.
- Penjualan & otomasi jurnal akuntansi (double-entry ledger, synchronous).

## Repositori

| Repo | Deskripsi |
|---|---|
| [`.github`](https://github.com/erp-saas-id/.github) | Profil organisasi dan konfigurasi global (repo ini) |
| [`backend`](https://github.com/erp-saas-id/backend) | NestJS API Service |
| [`frontend`](https://github.com/erp-saas-id/frontend) | Next.js Client App |

> [!NOTE]
> Seluruh dokumen utama proyek (PRD, API Contract, dan aturan agent development) disimpan secara lokal pada direktori root workspace proyek dan tidak dilacak di dalam repositori publik ini.