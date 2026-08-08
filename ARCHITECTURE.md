# Architecture — Souvenir Catalog Management System

## 1. Overview

Sistem manajemen katalog produk souvenir dengan tiga kapabilitas utama:
1. Import data produk dari marketplace eksternal (Alibaba, dll)
2. Generate foto katalog menggunakan AI
3. Manajemen harga bertingkat (tiered pricing) dan data produk

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                       │
│              Next.js Frontend (React + Tailwind)              │
└───────────────────────────┬─────────────────────────────────┘
                             │ HTTPS / REST
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     API LAYER (Next.js API Routes             │
│                       atau Express Backend)                    │
│                                                                 │
│   /api/products        /api/products/import                   │
│   /api/categories       /api/products/:id/photos/generate      │
│   /api/photos           /api/products/:id/photos/upload        │
└─────┬───────────────┬──────────────┬──────────────┬──────────┘
      │               │              │              │
      ▼               ▼              ▼              ▼
┌───────────┐  ┌──────────────┐ ┌──────────┐ ┌─────────────┐
│ PostgreSQL │  │  Job Queue    │ │ Storage  │ │  External   │
│ (Supabase) │  │ (BullMQ/      │ │(Supabase │ │  Services   │
│            │  │  Inngest)     │ │ Storage/ │ │             │
│ - products │  │               │ │  S3)     │ │ - Scraper   │
│ - photos   │  │ ┌───────────┐ │ │          │ │ - AI Image  │
│ - tiers    │  │ │  Workers  │ │ │ - fotos  │ │   Gen API   │
│ - categories│ │ │(background│ │ │  produk  │ │ - Alibaba/  │
└───────────┘  │ │ processing)│ │ └──────────┘ │   Marketplace│
                │ └───────────┘ │               └─────────────┘
                └───────────────┘
```

---

## 3. Komponen Utama

### 3.1 Frontend
| Bagian | Teknologi | Fungsi |
|---|---|---|
| UI Framework | Next.js (App Router) | Rendering halaman, routing |
| Styling | Tailwind CSS | Styling komponen |
| State (client) | React Query / SWR | Cache & sinkronisasi data dari API, termasuk polling job status |
| Form | React Hook Form + Zod | Validasi form (harga tier, dimensi dinamis) |

### 3.2 Backend / API Layer
| Bagian | Teknologi | Fungsi |
|---|---|---|
| API Routes | Next.js API Routes / Express | Endpoint CRUD produk, kategori, foto |
| Auth (opsional) | Supabase Auth / NextAuth | Login admin/tim internal |
| Validasi | Zod / Joi | Validasi request sebelum masuk ke DB |

### 3.3 Database
| Tabel | Fungsi |
|---|---|
| `products` | Data inti produk (nama, kategori, HPP, status) |
| `categories` | Daftar kategori produk |
| `product_price_tiers` | Harga per tier (retail/grosir kecil/grosir besar) |
| `product_dimensions` | Dimensi produk (fleksibel, key-value) |
| `product_photos` | Foto produk + metadata (sumber, ukuran varian, status pilih) |
| `import_jobs` | Riwayat & status job scraping |
| `generate_photo_jobs` | Riwayat & status job generate foto AI |

### 3.4 Job Queue / Background Worker
Dipakai untuk proses yang **butuh waktu lama** dan tidak boleh bikin user menunggu (blocking):
- **Scraping produk** dari URL eksternal
- **Generate foto AI**
- **Generate varian ukuran foto** (400/800/1200/1600px + convert ke WebP)

Alur: API menerima request → simpan job dengan status `processing` → worker proses di background → hasil disimpan → frontend polling status via endpoint `/job/:id`.

### 3.5 Storage
Menyimpan file foto (original + varian ukuran). Bisa pakai Supabase Storage, AWS S3, atau Cloudflare R2 — cukup butuh signed URL untuk upload aman dari client.

### 3.6 External Services (Third-Party)
| Layanan | Fungsi |
|---|---|
| Scraper service | Ambil data produk (nama, foto, deskripsi) dari halaman marketplace |
| AI Image Generation API | Generate variasi foto katalog dari foto sumber |

---

## 4. Alur Data (Data Flow) — Contoh: Import Produk

```
User paste URL
     │
     ▼
Frontend → POST /api/products/import
     │
     ▼
API buat record job (status: processing) → masuk ke Job Queue
     │
     ▼
Worker ambil job → panggil Scraper Service → dapat data mentah
     │
     ▼
Worker cek foto (hash/URL) vs data existing → tandai duplikat jika ada
     │
     ▼
Worker simpan hasil ke DB (job status: completed, result: {...})
     │
     ▼
Frontend polling GET /api/products/import/:job_id
     │
     ▼
Job selesai → tampilkan hasil ke user → user review & simpan sebagai draft produk
```

---

## 5. Prinsip Desain

- **Async-first untuk proses berat**: scraping, generate AI photo, dan resize gambar semua lewat job queue — supaya API tetap responsif dan tidak timeout.
- **Idempotent import**: mengimpor URL yang sama dua kali seharusnya tidak membuat data ganda tanpa peringatan (deteksi duplikat).
- **Progressive enhancement pada form produk**: produk bisa disimpan sebagai draft meski data belum lengkap; validasi ketat baru diberlakukan saat status diubah ke `published`.
- **Pemisahan source_type pada foto**: setiap foto punya label asal (`upload`, `imported`, `ai_generated`) untuk keperluan audit dan analisis kualitas AI generate ke depannya.
- **Stateless API**: semua state disimpan di DB/queue, bukan di memory server — supaya API bisa di-scale horizontal tanpa masalah.

---

## 6. Skalabilitas (Pertimbangan Lanjutan)

| Area | Potensi Bottleneck | Mitigasi |
|---|---|---|
| Generate foto AI | Rate limit dari provider AI, biaya per-generate | Queue + rate limiting + caching hasil |
| Scraping | Marketplace bisa block IP / berubah struktur halaman | Retry logic, monitoring, fallback manual input |
| Storage foto | Ukuran file besar, banyak varian | CDN + lifecycle policy (hapus draft lama yang tidak dipublish) |
| Database | Query produk dengan banyak relasi (tier, foto, dimensi) | Index yang tepat, pertimbangkan denormalisasi untuk list view |

---

## 7. Struktur Folder (Contoh — Next.js)

```
/app
  /products
    page.tsx           → list produk
    /new
      page.tsx          → form tambah produk
    /[id]
      page.tsx          → detail/edit produk
  /api
    /products
      route.ts
      /import
        route.ts
      /[id]
        route.ts
        /photos
          /generate/route.ts
          /upload/route.ts
/components
  ProductForm.tsx
  PhotoUploader.tsx
  PriceTierEditor.tsx
  AIPhotoReview.tsx
/lib
  supabase.ts
  queue.ts
  scraper.ts
  aiImageClient.ts
/types
  product.ts
/workers
  scrapeWorker.ts
  generatePhotoWorker.ts
```
