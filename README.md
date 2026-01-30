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
| **WAHA account** | `wahaApi` | Masukkan URL instance WAHA dan Secret Key Anda. <img width="624" height="401" alt="image" src="https://github.com/user-attachments/assets/515d2356-a9d5-4f47-9c41-a031635ce26d" />|
| **Google Sheets account** | `googleSheetsOAuth2Api` | Menggunakan OAuth2 Client ID dan Secret dari Google Cloud Console (jangan lupa sign in with Google).  <img width="623" height="404" alt="image" src="https://github.com/user-attachments/assets/91758c15-da09-4934-90b5-44efe545aa0b" /> |
| **Google Drive account** | `googleDriveOAuth2Api` | Menggunakan OAuth2 Client ID dan Secret dengan cakupan (scope) akses Google Drive (jangan lupa sign in with Google). <img width="621" height="403" alt="image" src="https://github.com/user-attachments/assets/7525eabe-0fe2-47b0-a2a8-e010fc30dfe2" />|
| **Google Gemini(PaLM) Api account** | `googlePalmApi` | API Key dari Google AI Studio untuk model Gemini (jangan lupa sign in with Google). <img width="621" height="403" alt="image" src="https://github.com/user-attachments/assets/fe0f4a6d-de4f-4ca6-9039-703fd3b55189" />|
| **Supabase account** | `supabaseApi` | URL Proyek Supabase dan Service Role Key. Pergi ke Supabase untuk melihat URL Project dan Service Role Secret <img width="623" height="410" alt="image" src="https://github.com/user-attachments/assets/4ddda1ca-b998-45c4-a306-44c4774fc984" /> <img width="1796" height="950" alt="image" src="https://github.com/user-attachments/assets/41a597a7-4f42-4ff9-a78e-b750c6aab2bf" /> <img width="893" height="473" alt="image" src="https://github.com/user-attachments/assets/97db22c2-864f-45ee-88cf-2a575e1d0289" />|
| **CohereApi account** | `cohereApi` | API Key dari platform Cohere. <img width="1247" height="803" alt="image" src="https://github.com/user-attachments/assets/3dbe439b-e4ec-4543-b6c4-ec3586c26354" /> <img width="894" height="350" alt="image" src="https://github.com/user-attachments/assets/3cf75c45-7b6d-4895-b07b-2470593f3aaa" />|
| **Postgres account** | `postgres` | Detail koneksi database PostgreSQL (Host, User, Password, Database). Pilih Connect di Table Editor, ganti Method jadi Transaction Poooler. <img width="625" height="404" alt="image" src="https://github.com/user-attachments/assets/5039034a-7db7-465c-a116-87d820005969" /> <img width="762" height="449" alt="image" src="https://github.com/user-attachments/assets/813806cf-6a2f-43e0-9564-0ba17d55c3c4" />|
| **Header Auth account** | `httpHeaderAuth` | Digunakan jika API konversi LID memerlukan autentikasi header tertentu (untuk convert LID). <img width="1243" height="799" alt="image" src="https://github.com/user-attachments/assets/231824e7-4052-45de-a233-9d2b0bf112e1" />|

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

## Kendala
* **Limit google api key (rpm/rpd)**:
  <img width="1460" height="1009" alt="image" src="https://github.com/user-attachments/assets/278891dc-f4a2-4e01-bed5-b2bad2542a9a" />
   Solusi: Aktifkan billing (berbayar) / jangan gunakan terlalu sering agar tidak mencapai limit (free)
* **Eror WAHA tidak merespon**: 
  <img width="1726" height="947" alt="Screenshot 2026-01-14 224031" src="https://github.com/user-attachments/assets/6eb6ab4d-de44-4596-a323-d8532ac16e4f" />
   Solusi: Tunggu update developer / tanya gemini
* **ngrok tidak berjalan di internet INKA**:
  <img width="1391" height="1113" alt="Screenshot 2026-01-20 080154" src="https://github.com/user-attachments/assets/129bdf9b-2e95-4c6d-84d9-61a903563854" />
   Solusi: Gunakan hotspot pribadi
* **Kalau filenya banyak dan besar, ngrok terkena eror bandwidth**:
  <img width="1090" height="647" alt="Screenshot 2026-01-24 213546" src="https://github.com/user-attachments/assets/ba43183f-92de-404f-b1b6-cd7c06748228" />
   Solusi: Gunakan akun ngrok berbayar / tunggu bulan depan (free)
  
## Catatan Penting
* **Biaya API**: Penggunaan model Gemini, Cohere, dan layanan Google Cloud dapat dikenakan biaya tergantung pada volume penggunaan. Pantau kuota API Anda secara berkala.
* **Keamanan Data**: Pastikan Google Sheets yang berisi daftar pengguna (whitelist) tidak dibagikan ke publik (Public Access) dan hanya dapat diakses oleh *Service Account* yang digunakan di n8n.
