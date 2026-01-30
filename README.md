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
4.  **WAHA DASHBOARD**:
    * Setting container WAHA di Docker menjadi seperti di bawah ini:
      <img width="501" height="496" alt="image" src="https://github.com/user-attachments/assets/d4b630e0-5091-4fc2-80ac-48693f2c4633" />
      <img width="653" height="194" alt="image" src="https://github.com/user-attachments/assets/9c47c298-758f-4b15-9c15-7d74c2a9ac3f" />
    * Masukkan API KEY KE WAHA DASHBOARD dan Kredential Node Convert LID, Send Response, dan Send Unauthorized Message
      <img width="671" height="820" alt="image" src="https://github.com/user-attachments/assets/7c334e13-c04d-4127-882e-673e5e8a1837" />
      <img width="1715" height="934" alt="image" src="https://github.com/user-attachments/assets/3a58214a-795a-42d3-b0e8-1e24f3049c0c" />
      <img width="1242" height="804" alt="image" src="https://github.com/user-attachments/assets/8aac0883-d90a-4bed-9897-4b3d0b4bec9d" />
      <img width="1241" height="805" alt="image" src="https://github.com/user-attachments/assets/73694403-4638-406e-988e-dd47ff03b8a5" />

## Catatan Penting
* **Biaya API**: Penggunaan model Gemini, Cohere, dan layanan Google Cloud dapat dikenakan biaya tergantung pada volume penggunaan. Pantau kuota API Anda secara berkala.
* **Keamanan Data**: Pastikan Google Sheets yang berisi daftar pengguna (whitelist) tidak dibagikan ke publik (Public Access) dan hanya dapat diakses oleh *Service Account* yang digunakan di n8n.
