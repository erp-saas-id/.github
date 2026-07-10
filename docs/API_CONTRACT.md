# API Contract & Response Standardization

Referensi: `PRD.md` Bagian 5 (Kebutuhan Non-Fungsional).

---

## 1. Base URL & Versioning

```
https://{tenant-slug}.domain.com/api/v1
```

Setiap breaking change pada struktur response atau request menaikkan versi (`/api/v2`). Versi lama tetap aktif selama masa deprecation yang disepakati.

---

## 2. Format Response
### 2.1 Response Sukses

```json
{
  "success": true,
  "data": {},
  "message": "string (optional)"
}
```

Ketentuan:

- `data` bertipe object untuk single resource, array untuk list resource.
- `message` opsional, dipakai untuk aksi yang butuh konfirmasi teks (contoh: `"Product created"`).
### 2.2 Response Sukses — List/Paginated

```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 145,
    "totalPages": 8
  }
}
```

Ketentuan:

- Query parameter: `?page=1&limit=20`.
- Default `page=1`, `limit=20`, `limit` maksimum 100.
### 2.3 Response Error

```json
{
  "success": false,
  "message": "string",
  "errors": null,
  "timestamp": "2026-07-10T10:00:00.000Z",
  "path": "/api/v1/products"
}
```

Response error validasi (`errors` berisi array):

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "email must be a valid email" },
    { "field": "password", "message": "password must be at least 8 characters" }
  ],
  "timestamp": "2026-07-10T10:00:00.000Z",
  "path": "/api/v1/auth/register"
}
```

Ketentuan:

- `errors: null` untuk error tanpa detail per-field (401, 403, 404, 409, 500).
- `errors: array` untuk error validasi input (422).
- Nama exception class tidak dimasukkan ke body, dicatat di server-side log (contoh: Winston/Pino logger dengan `requestId` untuk korelasi).

---

## 3. Konvensi HTTP Status Code

|Kode|Kondisi|
|---|---|
|200|Request berhasil (GET, PATCH, DELETE)|
|201|Resource berhasil dibuat (POST)|
|400|Request malformed (format body salah, parameter tidak valid)|
|401|Token tidak ada / tidak valid / expired|
|403|Token valid, tapi permission tidak mencukupi|
|404|Resource tidak ditemukan|
|409|Konflik state (contoh: duplicate SKU, stok tidak mencukupi)|
|422|Validasi input gagal (per-field)|
|500|Internal server error|

---

## 4. Konvensi Naming Endpoint

- Resource plural, lowercase, kebab-case untuk multi-kata: `/products`, `/stock-movements`, `/journal-entries`.
- Nested resource untuk relasi tenant-scoped: `/warehouses/{warehouseId}/stocks`.
- Action non-CRUD sebagai sub-path verb: `/invoices/{id}/void`, `/auth/refresh-token`.

---

## 5. Header Wajib

|Header|Keterangan|
|---|---|
|`Authorization`|tidak dipakai untuk auth utama (JWT ada di HTTP-Only Cookie), tersedia sebagai fallback untuk service-to-service call|
|`X-Request-Id`|UUID per-request, digunakan untuk korelasi log dan tracing|

Tenant context diresolusi dari subdomain, bukan header eksplisit (lihat `PRD.md` Bagian 5).

---

## 6. Konvensi Request Body

- Field naming: `camelCase`.
- Tanggal/waktu: ISO 8601 (`2026-07-10T10:00:00.000Z`), timezone UTC.
- Angka uang: integer dalam satuan terkecil (contoh: Rupiah tanpa desimal, disimpan sebagai integer utuh), bukan float, untuk menghindari floating-point rounding error.