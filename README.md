---

# Dokumentasi Teknis: n8n WhatsApp PDF Reader (RAG System)

Dokumen ini berfungsi sebagai panduan teknis untuk alur kerja (workflow) n8n "CHATBOT N8N PDF READER". Sistem ini mengimplementasikan arsitektur *Retrieval-Augmented Generation* (RAG) untuk memungkinkan interaksi tanya-jawab berbasis dokumen perusahaan melalui antarmuka WhatsApp.

## Ringkasan Sistem

Sistem ini mengintegrasikan layanan WhatsApp (melalui WAHA), Google Workspace (Sheets & Drive), Basis Data Vektor (Supabase), dan Kecerdasan Buatan (Google Gemini) untuk menciptakan asisten virtual yang mampu:

1. Memverifikasi otorisasi pengguna berdasarkan daftar putih (whitelist).
2. Menjawab pertanyaan berdasarkan konteks dokumen PDF yang tersimpan.
3. Melakukan sinkronisasi dan ekstraksi teks (OCR) dari dokumen PDF baru secara otomatis setiap hari.

## Prasyarat dan Kredensial

Sebelum mengimpor dan menjalankan alur kerja ini, pastikan kredensial berikut telah dikonfigurasi pada instance n8n Anda:

### 1. Google Cloud Platform

Diperlukan akun layanan (Service Account) atau OAuth2 dengan cakupan akses (scopes) berikut:

* **Google Sheets API:** Digunakan untuk membaca daftar pengguna yang diizinkan dan metadata dokumen.
* **Google Drive API:** Digunakan untuk mengunduh berkas PDF fisik.
* **Google Gemini (PaLM) API:** Digunakan sebagai *Large Language Model* (LLM) untuk agen percakapan dan pembuatan *embedding*.

### 2. Basis Data (Supabase/PostgreSQL)

* **Supabase API:** Kredensial untuk mengakses basis data Supabase.
* **Postgres Connection:** Digunakan untuk *Chat Memory* (menyimpan riwayat percakapan) dan penyimpanan vektor (pgvector).
* **Tabel Wajib:**
* Tabel penyimpanan dokumen (contoh: `documents`) dengan ekstensi vektor aktif.
* Tabel memori percakapan.



### 3. Layanan Pesan & Pendukung

* **WAHA (WhatsApp HTTP API):** Koneksi ke instance WAHA untuk menerima dan mengirim pesan WhatsApp.
* **Cohere API:** Digunakan untuk fungsi *Reranker* guna meningkatkan relevansi hasil pencarian dokumen.

---

## Arsitektur Alur Kerja

Alur kerja ini terbagi menjadi dua segmen operasional utama:

### A. Jalur Interaksi Pengguna (Real-Time)

Segmen ini menangani pesan masuk dan memberikan respons.

1. **Pemicu (Trigger) & Autentikasi:**
* **WAHA Trigger:** Menerima *webhook* pesan masuk dari WhatsApp.
* **Normalisasi & Validasi:** Sistem menormalisasi format nomor telepon, kemudian memverifikasinya terhadap basis data pengguna di Google Sheets (Node: `Check User Access` dan `User Authorized?`).
* **Penolakan Akses:** Jika nomor tidak terdaftar, sistem mengirimkan pesan penolakan otomatis.


2. **Agen Kecerdasan Buatan (AI Agent):**
* Jika pengguna terautorisasi, permintaan diteruskan ke **AI Agent** berbasis LangChain.
* **Model:** Menggunakan Google Gemini 2.5 Flash.
* **Memori:** Menyimpan konteks percakapan menggunakan `Chat Memory` (Postgres).
* **Alat (Tools):**
* `Supabase Vector Store`: Mencari konten tekstual dalam dokumen menggunakan pencarian vektor dan *Cohere Reranker*.
* `Get Document Metadata`: Mengambil informasi meta dari Google Sheets (seperti tautan PDF, PIC, dan Kode Dokumen).


### B. Jalur Sinkronisasi Data (Ingestion Pipeline)

Segmen ini berjalan secara terjadwal untuk memproses dokumen baru.

1. **Penjadwalan:**
* **Daily Sync:** Berjalan otomatis setiap pukul 02:00 pagi.

2. **Ekstraksi & Analisis Dokumen:**
* **Read Document List:** Membaca daftar dokumen target dari Google Sheets.
* **Download PDF:** Mengunduh fail PDF fisik dari Google Drive.
* **Analyze Document:** Menggunakan node native **Google Gemini** (Model: `models/gemini-2.0-flash`). Node ini menerima input biner PDF secara langsung dan diinstruksikan untuk mengekstrak seluruh isi dokumen tanpa ringkasan (OCR penuh), menjaga struktur tabel dan teks asli.


3. **Penyimpanan Vektor:**
* **Smart Chunking:** Memecah teks hasil ekstraksi menjadi segmen-segmen lebih kecil (chunks) dan memperkaya setiap segmen dengan metadata.
* **Embedding & Insert:** Mengonversi teks menjadi representasi vektor (menggunakan `Embeddings Google Gemini`) dan menyimpannya ke tabel `documents` di Supabase.

---

## Panduan Konfigurasi Node

Setelah mengimpor fail JSON, Anda wajib melakukan penyesuaian pada node-node berikut:

1. **Node: Check User Access (Google Sheets)**
* Ganti `Document ID` (Spreadsheet ID) dengan ID Google Sheet milik Anda yang berisi daftar pengguna (whitelist).
* Pastikan nama kolom pencarian (`lookupColumn`) sesuai dengan header di Google Sheet Anda (misalnya: "WHATSAPP").


2. **Node: Get Document Metadata & Read Document List**
* Sesuaikan `Document ID` dan `Sheet Name` agar mengarah ke inventaris dokumen utama perusahaan Anda.


3. **Node: Supabase Vector Store & Insert to Vector DB**
* Pastikan parameter `Table Name` sesuai dengan nama tabel yang telah Anda buat di Supabase (default: `documents`).


4. **Node: Analyze document (Google Gemini)**
* Node ini menggunakan kredensial **Google Gemini (PaLM) Api**.
* Pastikan akun Google Cloud Anda telah mengaktifkan akses ke model `gemini-2.0-flash`.
* Jika model tersebut tidak muncul dalam daftar *dropdown*, Anda mungkin perlu mengetikkan `models/gemini-2.0-flash` secara manual pada kolom **Model Name** atau memperbarui kredensial n8n Anda.


## Catatan Penting

* **Anti-Halusinasi:** Agen AI telah dikonfigurasi dengan instruksi sistem (System Prompt) yang ketat untuk hanya menjawab berdasarkan data yang ditemukan di Vector Store atau Metadata. Agen diinstruksikan untuk menyertakan sitasi dan tautan dokumen asli.
* **Penanganan Galat (Error Handling):** Pada proses sinkronisasi data, jika terjadi kegagalan pada satu berkas, skrip `Prepare Gemini Payload` dirancang untuk mencatat kesalahan dan melanjutkan ke berkas berikutnya (continue on error).

---
Berikut adalah kueri SQL yang diperlukan untuk menginisialisasi basis data di Supabase agar kompatibel dengan alur kerja n8n dan model Google Gemini (Embedding-001).
Buka Dashboard Supabase --> Pilih Your Organization --> Pilih Project --> Pilih SQL Editor di bar sebelah kiri

```
-- Enable the pgvector extension (idempotent)
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table to store your documents
CREATE TABLE IF NOT EXISTS documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT, -- corresponds to Document.pageContent
  metadata JSONB, -- corresponds to Document.metadata
  embedding vector(3072) -- adjust dimension if needed
);

-- Create a function to search for documents by embedding similarity
CREATE OR REPLACE FUNCTION match_documents (
  query_embedding vector(3072),
  match_count int DEFAULT NULL,
  filter jsonb DEFAULT '{}'
) RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  -- If match_count IS NULL, we don't apply a LIMIT
  IF match_count IS NULL THEN
    RETURN QUERY
    SELECT
      d.id,
      d.content,
      d.metadata,
      1 - (d.embedding <=> query_embedding) AS similarity
    FROM documents d
    WHERE (filter = '{}'::jsonb) OR (d.metadata @> filter)
    ORDER BY d.embedding <=> query_embedding;
  ELSE
    RETURN QUERY
    SELECT
      d.id,
      d.content,
      d.metadata,
      1 - (d.embedding <=> query_embedding) AS similarity
    FROM documents d
    WHERE (filter = '{}'::jsonb) OR (d.metadata @> filter)
    ORDER BY d.embedding <=> query_embedding
    LIMIT match_count;
  END IF;
END;
$$;
```

Catatan Tambahan untuk README: Anda dapat menghapus node "Gemini Flash OCR" (HTTP Request) yang tidak tersambung tersebut dari kanvas n8n agar alur kerja terlihat lebih rapi dan tidak membingungkan saat pemeliharaan sistem di masa depan.
