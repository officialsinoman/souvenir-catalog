# AGENTS.md — Souvenir Catalog Management System

Panduan ini dibaca oleh AI coding agent (Claude Code, Cursor, dll) sebelum bekerja di repo ini. Tujuannya biar agent paham konteks project, konvensi kode, dan batasan yang harus dipatuhi.

---

## 1. Ringkasan Project

Web app untuk mengelola katalog produk souvenir (piring, tumbler, mug, dll). Fitur utama:
- Import data produk dari marketplace eksternal (Alibaba, dll) via scraping
- Generate foto katalog pakai AI, dengan review manual (Pakai/Buang)
- Manajemen harga bertingkat (tier retail / grosir kecil / grosir besar)
- Upload & optimasi foto (auto-generate varian WebP)

Lihat `ARCHITECTURE.md` untuk detail arsitektur sistem dan `API_CONTRACT.md` untuk daftar endpoint.

---

## 2. Tech Stack

- **Frontend**: Next.js 14 (App Router) + TypeScript + Tailwind CSS
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL via Supabase
- **Storage**: Supabase Storage (foto produk)
- **Job Queue**: [isi sesuai pilihan — misal BullMQ/Inngest] untuk proses async (scraping, generate AI photo, resize gambar)
- **Validasi**: Zod
- **Form**: React Hook Form

---

## 3. Struktur Folder

```
/app            → halaman & API routes (App Router)
/components     → komponen React reusable
/lib            → client Supabase, util, integrasi eksternal
/types          → TypeScript types/interfaces
/workers        → background job processor
```

Ikuti struktur ini. Jangan taruh logic bisnis di dalam komponen UI — pisahkan ke `/lib`.

---

## 4. Konvensi Kode

- **TypeScript strict mode** — semua file harus fully typed, hindari `any`.
- **Naming**: `camelCase` untuk variabel/fungsi, `PascalCase` untuk komponen React, `snake_case` untuk kolom database (sesuai konvensi Postgres/Supabase).
- **API response**: selalu ikuti format di `API_CONTRACT.md`. Error response wajib punya `error` (kode) dan `message` (human-readable).
- **Async job**: proses yang > 2 detik (scraping, generate foto AI, resize gambar) WAJIB lewat job queue, tidak boleh blocking di API route langsung.
- **Validasi input**: semua request body divalidasi pakai Zod schema sebelum diproses — schema disimpan di `/lib/validation`.
- **Commit message**: format `type: deskripsi singkat` (misal `feat: tambah endpoint generate foto AI`, `fix: validasi resolusi upload`).

---

## 5. Aturan Khusus Domain

- **Status produk** (`draft | incomplete | ready_to_publish | published | archived | deleted`) — jangan izinkan transisi ke `published` kalau validasi belum lolos (minimal 1 foto terpilih + 1 tier harga). Cek `ARCHITECTURE.md` bagian State Machine.
- **Deteksi foto duplikat**: setiap kali ada foto baru masuk (dari import atau upload), cek dulu terhadap `product_photos` yang sudah ada (pakai hash gambar, bukan cuma URL) sebelum disimpan.
- **Foto AI generate**: hasil generate WAJIB masuk state `pending_review` dulu — tidak boleh otomatis `is_selected: true`. User yang menentukan lewat aksi Pakai/Buang.
- **Varian foto**: setiap upload foto harus menghasilkan 4 varian ukuran (400/800/1200/1600px) dalam format WebP. Jangan skip proses ini meskipun untuk kebutuhan development/testing — pakai data dummy variant kalau perlu mock.

---

## 6. Yang TIDAK Boleh Dilakukan Agent

- Jangan hardcode API key atau kredensial Supabase/AI provider di kode — selalu pakai environment variable (`.env.local`, jangan pernah commit file ini).
- Jangan membuat migration database yang mengubah/menghapus kolom existing tanpa membuat migration terpisah yang reversible.
- Jangan mengubah `API_CONTRACT.md` tanpa memberi tahu di ringkasan perubahan — kontrak ini jadi acuan frontend & backend.
- Jangan menambah dependency baru tanpa alasan jelas — cek dulu apakah kebutuhan sudah bisa dipenuhi library yang sudah ada di `package.json`.

---

## 7. Testing

- Setiap API route baru minimal punya 1 test happy-path + 1 test error-case (misal validasi gagal).
- Untuk fitur job queue (scraping, generate AI photo), buat mock/stub agar test tidak benar-benar memanggil service eksternal.
- Jalankan `npm run test` sebelum menganggap task selesai.

---

## 8. Cara Menjalankan Project

```bash
npm install
cp .env.example .env.local   # isi kredensial Supabase & AI provider
npm run dev
```

---

## 9. Referensi Dokumen Lain

- `ARCHITECTURE.md` — arsitektur sistem, alur data, state machine
- `API_CONTRACT.md` — daftar endpoint, request/response schema
- `.env.example` — daftar environment variable yang dibutuhkan
