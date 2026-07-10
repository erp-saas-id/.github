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

## Repository

| Repo | Deskripsi |
|---|---|
| [`erp-saas-id`](https://github.com/erp-saas-id/erp-saas-id) | Dokumentasi, PRD, API contract, konvensi development (repo ini) |
| [`frontend`](https://github.com/erp-saas-id/frontend) | Next.js client |
| [`backend`](https://github.com/erp-saas-id/backend) | NestJS API |

## Dokumen

- [`PRD.md`](./PRD.md) — requirement dan keputusan arsitektur
- [`docs/API_CONTRACT.md`](./docs/API_CONTRACT.md) — standar request/response API
- [`.agents/AGENTS.md`](./.agents/AGENTS.md) — konvensi global development
- [`.agents/workflows/INSTRUCTION.md`](./.agents/workflows/INSTRUCTION.md) — tahapan implementasi per fase