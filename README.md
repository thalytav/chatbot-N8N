# chatbot-N8N
Dokumen ini mencakup **Alur Kerja (Logic)**, **Prasyarat (Credentials)**, **Struktur Database**, dan **Panduan Instalasi**.

---

# 🤖 WhatsApp AI Document Assistant (n8n Workflow)

Workflow ini adalah sistem **RAG (Retrieval-Augmented Generation)** berbasis WhatsApp yang dibangun menggunakan n8n. Chatbot ini berfungsi sebagai asisten dokumentasi internal perusahaan, mampu menjawab pertanyaan karyawan berdasarkan dokumen PDF yang tersimpan di Google Drive.

## ✨ Fitur Utama

1. **WhatsApp Interface**: Terintegrasi via WAHA (WhatsApp HTTP API).
2. **User Whitelist**: Validasi nomor HP pengguna melalui Google Sheets sebelum mengizinkan akses.
3. **Advanced RAG**:
* **OCR & Ingestion**: Pipeline otomatis (harian) untuk download PDF, ekstrak teks menggunakan **Gemini Flash 2.0**, dan chunking cerdas.
* **Vector Search**: Menggunakan Supabase (pgvector) untuk penyimpanan embedding.
* **Reranking**: Menggunakan Cohere untuk meningkatkan relevansi pencarian.


4. **Hybrid Information**: Menggabungkan konten isi dokumen (Vector Store) dengan metadata dokumen (Google Sheets).
5. **Memory**: Menyimpan riwayat percakapan pengguna (Postgres).

---

## 🛠️ Prasyarat & Layanan yang Dibutuhkan

Sebelum mengimpor workflow, pastikan Anda memiliki akses dan kredensial untuk layanan berikut:

| Layanan | Kegunaan | Node Credential Name |
| --- | --- | --- |
| **n8n** | Platform Otomasi | - |
| **WAHA** | WhatsApp API Gateway | `WAHA account` |
| **Google Cloud** | Sheets, Drive, Gemini API | `Google Sheets account`, `Google Drive account`, `Google Gemini(PaLM) Api account` |
| **Supabase** | Vector Database | `Supabase account` |
| **Cohere** | Reranking Model | `CohereApi account` |
| **PostgreSQL** | Chat Memory | `Postgres account` |
| **Header Auth** | Auth untuk API Internal (LID) | `Header Auth account` |

---

## ⚙️ Struktur Data (PENTING)

Agar workflow berjalan lancar, struktur data di Google Sheets dan Database harus sesuai dengan format berikut:

### 1. Google Sheets: User Whitelist

Sheet ini digunakan untuk mengecek apakah nomor WA boleh mengakses bot.

* **Sheet Name**: (Sesuai setting di node `Check User Access`)
* **Kolom Wajib**:
* `WHATSAPP`: Nomor HP (format internasional/lokal ok, bot akan normalisasi).



### 2. Google Sheets: Daftar Dokumen (Metadata)

Sheet ini berisi daftar PDF yang akan diproses oleh Daily Sync.

* **Sheet Name**: (Sesuai setting di node `Read Document List`)
* **Kolom Wajib**:
* `Nama Dokumen`: Judul dokumen.
* `Kode Dokumen`: Nomor/Kode unik dokumen.
* `Penanggung Jawab`: PIC dokumen.
* `Pendukung`: Tim pendukung.
* `Link Drive (Real PDF)`: Link shareable Google Drive ke file PDF.



### 3. Supabase (SQL Setup)

Jalankan query ini di SQL Editor Supabase Anda untuk menyiapkan vector store:

```sql
-- Enable pgvector extension
create extension if not exists vector;

-- Create documents table
create table documents (
  id bigserial primary key,
  content text, -- Chunk text
  metadata jsonb, -- Metadata (Title, PIC, Link, dll)
  embedding vector(768) -- Sesuai dimensi Google Gemini Embedding
);

-- Create search function for LangChain
create function match_documents (
  query_embedding vector(768),
  match_threshold float,
  match_count int
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    documents.id,
    documents.content,
    documents.metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where 1 - (documents.embedding <=> query_embedding) > match_threshold
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;

```

---

## 🚀 Instalasi & Konfigurasi

1. **Import Workflow**:
* Buka n8n.
* Klik menu "Workflows" > "Import".
* Upload file `CHATBOT N8N FILE READER.json`.

  **WORKFLOW 1**
<img width="1419" height="235" alt="image" src="https://github.com/user-attachments/assets/c3eaa1ce-89c2-4a6d-b423-b17c5333993f" />

  **WORKFLOW 2**
<img width="1471" height="658" alt="image" src="https://github.com/user-attachments/assets/ca57030f-4ead-4987-8e24-6e1928e177b9" />


2. **Setup Credentials**:
* Isi semua credential yang "merah" (hilang) sesuai daftar di bagian Prasyarat.
* Pastikan API Key Google Gemini memiliki akses ke model `gemini-pro` dan `gemini-2.0-flash-exp`.
* **WORKFLOW 1**
  * Read Document List
    <img width="1705" height="933" alt="image" src="https://github.com/user-attachments/assets/2a30516a-17ab-4e97-ba76-de424b93c8f5" />
    Edit Kredensial dengan cara masukkan Client ID dan Client Secret Anda (dapatkan Client ID dan Client Secret di Google Cloud Console)
    <img width="1244" height="803" alt="image" src="https://github.com/user-attachments/assets/57cb1145-99bf-4411-941f-8ac04eba4e58" />


  * Download PDF From Drive
  * Prepare Gemini Payload
  * Gemini Flash OCR
  * Smart Chunking
  * Insert to Vector DB
  * Embedding (Sync)
  * Document Loader
* **WORKFLOW 2**
  * WAHA Trigger
  * is LID?
  * Convert LID
  * Normalize Number
  * Check User Access
  * User Authorized
  * Send Authorized Message
  * AI Agent
  * Gemini 2.5 Flash
  * Chat Memory
  * Get Document Metadata
  * Supabase Vector Store
  * Embeddings Google Gemini
  * Reranker Cohere

3. **Konfigurasi Node Variables**:
* **WAHA Trigger**: Pastikan `webhookId` unik dan sesuai dengan setting di dashboard WAHA Anda.
* **Google Sheets Node**: Pilih ulang File dan Sheet dari akun Google Drive Anda (ID File di JSON mungkin tidak valid di akun Anda).
* **Supabase Vector Store**: Pastikan nama tabel diisi `documents`.


4. **Setup Daily Sync**:
* Bagian bawah workflow ("Daily Sync") berjalan otomatis jam 02:00 AM.
* Untuk tes pertama kali, jalankan manual node-node di jalur ini untuk mengisi database vector.



---

## 🧠 Alur Logika (How it Works)

### A. Chat Flow (Interaksi User)

1. **Pesan Masuk**: Menerima pesan dari WAHA.
2. **Validasi**:
* Normalisasi nomor HP (hapus karakter non-digit, ubah 08xx jadi 62xx).
* Cek apakah nomor ada di Google Sheets Whitelist.
* Jika **TIDAK**: Kirim pesan "Tidak terdaftar".
* Jika **YA**: Lanjut ke AI Agent.


3. **AI Agent (LangChain)**:
* Menganalisis pertanyaan user.
* **Tool 1 - Vector Store**: Mencari potongan teks relevan dari database Supabase (Search + Rerank).
* **Tool 2 - Metadata Sheet**: Mengambil info pelengkap (Link PDF, PIC) dari Google Sheets.
* **Generasi Jawaban**: LLM (Gemini) menyusun jawaban ramah berdasarkan data yang ditemukan.


4. **Respon**: Mengirim jawaban teks ke WhatsApp user.

### B. Ingestion Flow (Update Dokumen Harian)

1. **Trigger**: Cron job harian.
2. **Fetch**: Baca daftar dokumen dari Google Sheets.
3. **Download**: Mengunduh file PDF biner dari Google Drive.
4. **OCR (Gemini Vision)**: Mengirim PDF ke `gemini-2.0-flash-exp` untuk ekstrak teks (termasuk tabel) secara akurat.
5. **Chunking**: Kode JavaScript memecah teks menjadi potongan kecil (chunk) + overlap agar konteks terjaga.
6. **Embedding & Insert**: Mengubah teks jadi vektor dan simpan ke Supabase.

---

## ⚠️ Troubleshooting Umum

* **Error "User Unauthorized" padahal nomor benar**: Cek format nomor di Google Sheet. Pastikan formatnya konsisten (misal: `62812...` tanpa spasi atau `+`).
* **Gemini OCR Gagal**: Pastikan file PDF tidak terlalu besar (limit API Gemini) dan API Key memiliki kuota.
* **Supabase Error**: Pastikan dimensi vector di database (768) sama dengan output model embedding yang dipakai (Google Gemini).

---

## 📝 Catatan Pengembang

* Workflow ini menggunakan **Gemini 2.0 Flash** via HTTP Request untuk OCR karena performanya lebih baik dan murah dibanding library parsing PDF standar.
* Logika chunking ada di node "Smart Chunking", Anda bisa mengubah `chunkSize` dan `overlap` di sana jika potongan teks dirasa terlalu panjang/pendek.

---

### Tips Tambahan untuk Kamu:

1. **Credentials**: Di dalam n8n, kamu harus membuat credential baru untuk masing-masing layanan. Nama credential di JSON (`Google Sheets account`, `WAHA account`, dll) adalah nama *default*. Jika kamu menamai credentialmu berbeda di n8n, kamu harus memilih ulang credential tersebut di setiap node.
2. **LID Logic**: Saya melihat ada logika `Is LID?` dan `Convert LID` yang memanggil endpoint lokal (`192.168.43.198`). Pastikan server ini aktif, atau jika ini tidak diperlukan lagi, node-node ini bisa di-bypass langsung ke `Normalize Number`.
3. **Drive Link**: Pastikan link Google Drive di spreadsheet bersifat "Anyone with the link can view" atau akun Service Account Google yang dipakai di n8n sudah di-invite ke folder/file tersebut.
