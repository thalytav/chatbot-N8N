# Dokumentasi Teknis: WhatsApp Chatbot dengan RAG dan Sinkronisasi Dokumen Otomatis

## Ringkasan
Workflow n8n ini dirancang untuk mengoperasikan asisten virtual berbasis WhatsApp yang berfungsi sebagai pusat layanan informasi internal. Sistem ini menggunakan metode *Retrieval-Augmented Generation* (RAG) untuk menjawab pertanyaan pengguna berdasarkan dokumen perusahaan yang tersimpan. Selain itu, sistem ini dilengkapi dengan mekanisme sinkronisasi otomatis harian untuk memperbarui basis pengetahuan (*knowledge base*) dari Google Drive ke Supabase Vector Store.

## Prasyarat Sistem
Sebelum mengimpor workflow ini, pastikan Anda memiliki akses dan API Key untuk layanan berikut:
1.  **WAHA (WhatsApp HTTP API)**: Sebagai antarmuka pengiriman dan penerimaan pesan WhatsApp.
2.  **Google Cloud Platform**: Untuk akses ke Google Sheets, Google Drive, dan Vertex AI/Gemini API.
3.  **Supabase**: Digunakan sebagai basis data vektor (*Vector Store*) untuk menyimpan *embeddings* dokumen.
4.  **Cohere**: Digunakan untuk fitur *Reranker* guna meningkatkan relevansi hasil pencarian dokumen.
5.  **PostgreSQL**: Digunakan untuk menyimpan riwayat percakapan (*chat memory*).

## Konfigurasi Kredensial
Agar workflow dapat berjalan, Anda perlu mendaftarkan kredensial berikut pada menu **Credentials** di n8n. Pastikan nama kredensial sesuai dengan tipe yang tertera pada tabel di bawah ini:

| Nama Kredensial | Tipe Kredensial di n8n | Keterangan |
| :--- | :--- | :--- |
| **WAHA account** | `wahaApi` | Masukkan URL instance WAHA dan Secret Key Anda. |
| **Google Sheets account** | `googleSheetsOAuth2Api` | Menggunakan OAuth2 Client ID dan Secret dari Google Cloud Console. |
| **Google Drive account** | `googleDriveOAuth2Api` | Menggunakan OAuth2 Client ID dan Secret dengan cakupan (scope) akses Google Drive. |
| **Google Gemini(PaLM) Api account** | `googlePalmApi` | API Key dari Google AI Studio untuk model Gemini. |
| **Supabase account** | `supabaseApi` | URL Proyek Supabase dan Service Role Key. |
| **CohereApi account** | `cohereApi` | API Key dari platform Cohere. |
| **Postgres account** | `postgres` | Detail koneksi database PostgreSQL (Host, User, Password, Database). |
| **Header Auth account** | `httpHeaderAuth` | (Opsional) Digunakan jika API konversi LID memerlukan autentikasi header tertentu. |

## Arsitektur Workflow

Workflow ini terdiri dari dua proses utama yang berjalan secara terpisah namun saling terintegrasi:

### 1. Alur Interaksi Pengguna (User Interaction Flow)
<img width="1228" height="432" alt="image" src="https://github.com/user-attachments/assets/994b0992-e0ee-43cb-9825-d2ca0ba2ef28" />

Alur ini menangani pesan masuk dari pengguna, validasi akses, dan pembuatan respons jawaban.

* **Pemicu (Trigger)**: Node `WAHA Trigger` mendeteksi pesan masuk.
* **Pra-pemrosesan Data**:
    * Sistem melakukan normalisasi nomor telepon pengguna (menghapus karakter non-numerik, format 628xxx).
    * Jika ID pengguna berupa LID (Linked ID), sistem akan mengonversinya menjadi nomor telepon standar melalui node `Convert LID`.
* **Validasi Keamanan (Gatekeeper)**:
    * Node `Check User Access` memeriksa apakah nomor telepon pengguna terdaftar dalam basis data (Google Sheets).
    * Jika pengguna **tidak terdaftar**, sistem mengirimkan pesan penolakan akses.
    * Jika pengguna **terdaftar**, permintaan diteruskan ke agen AI.
* **Pemrosesan AI (AI Agent)**:
    * Menggunakan model **Google Gemini 2.5 Flash**.
    * **Manajemen Memori**: Mengambil konteks percakapan sebelumnya dari PostgreSQL (`Chat Memory`).
    * **Pencarian Informasi (Retrieval)**: Agen menggunakan *tools* untuk mencari jawaban di `Supabase Vector Store` (konten dokumen) dan mengambil metadata tambahan dari Google Sheets (`Get Document Metadata`).
    * **Reranking**: Hasil pencarian vektor diurutkan ulang oleh `Reranker Cohere` untuk memastikan relevansi tertinggi.
* **Output**: Jawaban yang dihasilkan dikirim kembali ke pengguna melalui node `Send Response` (WAHA).

### 2. Alur Sinkronisasi Dokumen (Document Sync Flow)
<img width="1165" height="207" alt="image" src="https://github.com/user-attachments/assets/f26545c9-094e-459b-9b2d-d3b46dcd3c76" />

Alur ini berjalan secara otomatis setiap hari pada pukul 02:00 pagi untuk memperbarui database pengetahuan.

* **Pemicu (Trigger)**: `Daily Sync (2 AM)` menggunakan *cron expression*.
* **Pengambilan Daftar Dokumen**: Node `Read Document List` membaca daftar dokumen target dari Google Sheets.
* **Pengunduhan Dokumen**: Node `Download PDF from Drive` mengunduh file biner PDF berdasarkan ID file dari Google Drive.
* **Analisis Konten (OCR/Ekstraksi)**:
    * Dokumen PDF diproses menggunakan node `Analyze document` (Gemini Vision) untuk mengekstrak teks secara utuh, termasuk membaca tabel atau pindaian gambar.
* **Pemrosesan Teks (Chunking)**:
    * Node `Smart Chunking` memecah teks panjang menjadi segmen-segmen lebih kecil (default: 800 karakter) agar optimal untuk proses *embedding*.
    * Metadata dokumen (Nama, Kode, PIC) disematkan ke dalam setiap segmen teks.
* **Penyimpanan Vektor**:
    * Teks yang telah diproses diubah menjadi vektor menggunakan `Embeddings Google Gemini`.
    * Vektor disimpan ke tabel `documents` di Supabase melalui node `Insert to Vector DB`.

## Penyesuaian Node (Manual Setup)
Setelah mengimpor workflow ini, Anda wajib menyesuaikan beberapa parameter berikut agar sesuai dengan data perusahaan Anda:

1.  **Google Sheets (Node: Check User Access, Read Document List, Get Document Metadata)**:
    * Ganti `Document ID` (ID Spreadsheet) dengan ID Google Sheets milik Anda.
    * Pastikan nama `Sheet Name` sesuai dengan tab yang ada di file Anda.
2.  **Prompt Sistem (Node: AI Agent)**:
    * Sesuaikan instruksi pada bagian `System Message` di dalam node AI Agent untuk mengubah gaya bahasa atau aturan bisnis chatbot sesuai kebijakan perusahaan.
3.  **Database Vector (Node: Supabase Vector Store)**:
    * Pastikan tabel `documents` telah dibuat di Supabase dan memiliki ekstensi `pgvector` yang aktif.

## Catatan Penting
* **Biaya API**: Penggunaan model Gemini, Cohere, dan layanan Google Cloud dapat dikenakan biaya tergantung pada volume penggunaan. Pantau kuota API Anda secara berkala.
* **Keamanan Data**: Pastikan Google Sheets yang berisi daftar pengguna (whitelist) tidak dibagikan ke publik (Public Access) dan hanya dapat diakses oleh *Service Account* yang digunakan di n8n.
