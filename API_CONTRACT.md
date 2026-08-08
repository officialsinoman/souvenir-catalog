# API Contract — Souvenir Catalog Management System

Base URL: `/api`
Format: JSON (`Content-Type: application/json`), kecuali endpoint upload yang pakai `multipart/form-data`.

---

## 1. Products

### `GET /api/products`
List produk dengan filter & pagination.

**Query params:** `category_id`, `status`, `search`, `page` (default 1), `limit` (default 20)

**Response 200**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Classical Lace-up Design Plain White Platter",
      "category": { "id": "uuid", "name": "Piring" },
      "status": "published",
      "hpp": 15000,
      "thumbnail_url": "https://.../400.webp",
      "price_range": { "min": 20000, "max": 45000 },
      "created_at": "2026-08-01T10:00:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 87, "total_pages": 5 }
}
```

---

### `POST /api/products`
Buat produk baru (status awal: `draft`).

**Request**
```json
{
  "name": "Classical Lace-up Design Plain White Platter",
  "category_id": "uuid",
  "description": "Piring keramik putih polos, cocok untuk souvenir",
  "hpp": 15000,
  "source_url": "https://www.alibaba.com/product-detail/..."
}
```

**Response 201**
```json
{ "id": "uuid", "status": "draft", "created_at": "2026-08-08T07:53:00Z" }
```

---

### `GET /api/products/:id`
Detail produk lengkap (foto, tier harga, dimensi).

**Response 200**
```json
{
  "id": "uuid",
  "name": "Classical Lace-up Design Plain White Platter",
  "category": { "id": "uuid", "name": "Piring" },
  "description": "...",
  "hpp": 15000,
  "status": "published",
  "source_url": "https://...",
  "price_tiers": [
    { "id": "uuid", "tier_name": "retail", "min_qty": 1, "price": 45000 },
    { "id": "uuid", "tier_name": "grosir_kecil", "min_qty": 12, "price": 35000 },
    { "id": "uuid", "tier_name": "grosir_besar", "min_qty": 100, "price": 20000 }
  ],
  "dimensions": [
    { "label": "diameter", "value": 26, "unit": "cm" },
    { "label": "tinggi", "value": 3, "unit": "cm" }
  ],
  "photos": [
    {
      "id": "uuid",
      "url": "https://.../1600.webp",
      "variants": { "400": "...", "800": "...", "1200": "...", "1600": "..." },
      "source_type": "ai_generated",
      "is_selected": true
    }
  ]
}
```

**Error 404**
```json
{ "error": "NOT_FOUND", "message": "Produk tidak ditemukan" }
```

---

### `PATCH /api/products/:id`
Update sebagian data produk. Kirim hanya field yang berubah.

**Request**
```json
{ "hpp": 16000, "status": "published" }
```

**Response 200:** objek produk lengkap (struktur sama seperti `GET /api/products/:id`)

**Error 400** (validasi gagal saat publish)
```json
{
  "error": "VALIDATION_FAILED",
  "message": "Produk belum bisa dipublish",
  "details": ["Minimal 1 foto harus dipilih", "Minimal 1 tier harga harus diisi"]
}
```

---

### `DELETE /api/products/:id`
Soft delete (ubah status jadi `deleted`).

**Response 204:** No content

---

## 2. Categories

### `GET /api/categories`
**Response 200**
```json
{ "data": [{ "id": "uuid", "name": "Tumbler" }, { "id": "uuid", "name": "Piring" }] }
```

### `POST /api/categories`
**Request**
```json
{ "name": "Mug Custom" }
```
**Response 201**
```json
{ "id": "uuid", "name": "Mug Custom" }
```

---

## 3. Price Tiers

### `PUT /api/products/:id/price-tiers`
Replace seluruh tier harga produk (bulk update).

**Request**
```json
{
  "tiers": [
    { "tier_name": "retail", "min_qty": 1, "price": 45000 },
    { "tier_name": "grosir_kecil", "min_qty": 12, "price": 35000 }
  ]
}
```
**Response 200:** array tier tersimpan (masing-masing dengan `id`)

---

## 4. Dimensions

### `PUT /api/products/:id/dimensions`
**Request**
```json
{
  "dimensions": [
    { "label": "diameter", "value": 26, "unit": "cm" },
    { "label": "tinggi", "value": 3, "unit": "cm" }
  ]
}
```
**Response 200:** array dimensi tersimpan

---

## 5. Import dari URL (Scraping)

### `POST /api/products/import`
Ambil data produk dari link eksternal. Proses async (job).

**Request**
```json
{ "source_url": "https://www.alibaba.com/product-detail/Classical-Lace-up..." }
```
**Response 202**
```json
{ "job_id": "uuid", "status": "processing" }
```

---

### `GET /api/products/import/:job_id`
Polling status/hasil scraping.

**Response 200 — processing**
```json
{ "job_id": "uuid", "status": "processing" }
```

**Response 200 — completed**
```json
{
  "job_id": "uuid",
  "status": "completed",
  "result": {
    "name": "Classical Lace-up Design Wholesale Plain White Platter Plates",
    "description": "...",
    "photos": [
      { "url": "https://.../source1.jpg", "is_duplicate": true, "duplicate_of_product_id": "uuid" },
      { "url": "https://.../source2.jpg", "is_duplicate": false }
    ],
    "suggested_category": "Piring"
  }
}
```

**Response 200 — failed**
```json
{ "job_id": "uuid", "status": "failed", "error": "URL tidak valid atau tidak bisa diakses" }
```

---

## 6. Foto Katalog AI

### `POST /api/products/:id/photos/generate`
Generate foto AI dari foto sumber. Proses async (job).

**Request**
```json
{ "source_photo_id": "uuid", "count": 2, "style": "lifestyle" }
```
**Response 202**
```json
{ "job_id": "uuid", "status": "processing" }
```

---

### `GET /api/products/:id/photos/generate/:job_id`
**Response 200 — completed**
```json
{
  "job_id": "uuid",
  "status": "completed",
  "result": [
    { "photo_id": "uuid", "url": "https://.../gen1.webp", "review_state": "pending_review" },
    { "photo_id": "uuid", "url": "https://.../gen2.webp", "review_state": "pending_review" }
  ]
}
```

---

### `PATCH /api/photos/:photo_id/review`
User memilih Pakai/Buang untuk foto hasil AI.

**Request**
```json
{ "action": "select" }
```
atau
```json
{ "action": "discard" }
```

**Response 200**
```json
{ "photo_id": "uuid", "is_selected": true, "review_state": "selected" }
```

---

## 7. Upload Foto Manual

### `POST /api/products/:id/photos/upload`
`multipart/form-data`, field `files` (bisa banyak file sekaligus).

**Response 202**
```json
{
  "uploaded": [
    { "temp_id": "uuid", "original_filename": "plate1.jpg", "status": "processing_variants" }
  ]
}
```

**Error 422**
```json
{
  "error": "VALIDATION_FAILED",
  "details": [{ "filename": "plate2.jpg", "reason": "Resolusi minimal 1600px lebar" }]
}
```

---

### `GET /api/photos/upload-status/:temp_id`
**Response 200**
```json
{
  "temp_id": "uuid",
  "status": "ready",
  "photo_id": "uuid",
  "variants": {
    "400": "https://.../400.webp",
    "800": "https://.../800.webp",
    "1200": "https://.../1200.webp",
    "1600": "https://.../1600.webp"
  }
}
```

---

### `DELETE /api/photos/:photo_id`
**Response 204:** No content

---

## Kode Error Umum

| Code | Kapan Dipakai |
|---|---|
| `400 VALIDATION_FAILED` | Data request tidak valid |
| `404 NOT_FOUND` | Resource (produk/foto/job) tidak ditemukan |
| `409 DUPLICATE_PHOTO` | Foto sudah dipakai produk lain (strict mode) |
| `422 UNPROCESSABLE_ENTITY` | File upload gagal validasi (resolusi, format) |
| `429 TOO_MANY_REQUESTS` | Rate limit generate AI photo |
| `502 EXTERNAL_SERVICE_ERROR` | Scraping/AI API gagal di pihak ketiga |

Semua response error mengikuti format:
```json
{ "error": "ERROR_CODE", "message": "Pesan yang bisa dibaca manusia", "details": [] }
```
`details` bersifat opsional, dipakai untuk validasi multi-field.
