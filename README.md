# Souvenir Catalog Management System

Web app untuk mengelola katalog produk souvenir (piring, tumbler, mug, dll) — dengan fitur import produk dari marketplace eksternal, generate foto katalog pakai AI, dan manajemen harga bertingkat.

## Fitur Utama

- **Import produk dari URL** (Alibaba/marketplace lain) — ambil nama, foto, deskripsi otomatis
- **Deteksi foto duplikat** — cegah katalog berantakan akibat sumber foto yang sama
- **Generate foto katalog AI** — bikin variasi foto lifestyle dari foto sumber, dengan review manual (Pakai/Buang)
- **Upload foto manual** — auto-generate varian ukuran (400/800/1200/1600px) dalam format WebP
- **Harga bertingkat (tiered pricing)** — retail, grosir kecil, grosir besar
- **Manajemen kategori & dimensi produk** yang fleksibel

## Tech Stack

- **Frontend**: Next.js 14 (App Router) + TypeScript + Tailwind CSS
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL (Supabase)
- **Storage**: Supabase Storage
- **Job Queue**: untuk proses async (scraping, generate AI photo, resize gambar)

## Getting Started

### 1. Clone & install dependencies

```bash
git clone <repo-url>
cd souvenir-catalog
npm install
```

### 2. Setup environment variable

```bash
cp .env.example .env.local
```

Isi `.env.local` dengan kredensial Supabase, API key AI image provider, dan scraper service. Lihat komentar di `.env.example` untuk penjelasan tiap variable.

### 3. Setup database

Jalankan migration (sesuaikan dengan tool migration yang dipakai, misal Prisma atau Supabase CLI):

```bash
npx prisma migrate dev
# atau
supabase db push
```

### 4. Jalankan development server

```bash
npm run dev
```

Buka [http://localhost:3000](http://localhost:3000).

### 5. (Opsional) Jalankan worker untuk job queue

Kalau job queue dipisah dari proses Next.js utama:

```bash
npm run worker
```

## Struktur Dokumen

| File | Isi |
|---|---|
| `ARCHITECTURE.md` | Arsitektur sistem, alur data, state machine |
| `API_CONTRACT.md` | Daftar endpoint API, request/response schema |
| `AGENTS.md` | Panduan untuk AI coding agent (Claude Code, Cursor, dll) |
| `.env.example` | Daftar environment variable yang dibutuhkan |

## Struktur Folder

```
/app            → halaman & API routes (App Router)
/components     → komponen React reusable
/lib            → client Supabase, util, integrasi eksternal
/types          → TypeScript types/interfaces
/workers        → background job processor
```

## Testing

```bash
npm run test
```

## Kontribusi

Sebelum menambah fitur atau mengubah struktur, baca `ARCHITECTURE.md` dan `API_CONTRACT.md` supaya perubahan konsisten dengan desain yang sudah ada.
