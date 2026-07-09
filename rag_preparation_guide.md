# Panduan Step-by-Step: Menyiapkan Chat History untuk RAG

Jika Anda sudah memiliki riwayat percakapan ("profil" atau sesi chat lama) dan ingin mengubahnya menjadi memori RAG secara cepat, cara terbaik adalah **menggunakan AI (LLM) untuk memformat ulang data tersebut** agar Anda tidak perlu mengetik JSON secara manual satu per satu.

Berikut adalah langkah lengkap dari awal sampai data siap masuk ke RAG.

---

## Tahap 1: Mendapatkan Data Mentah (Manual)

1. **Siapkan Riwayat Chat:**
   Buka riwayat obrolan Anda (bisa dari ekspor data ChatGPT/Claude, atau sekadar *copy-paste* langsung dari layar percakapan).
2. **Simpan ke File Teks:**
   Tempelkan (*paste*) teks tersebut ke dalam sebuah file sementara, misalnya `raw_chat.txt`. Bentuknya mungkin akan berantakan, tidak masalah.
   *Contoh teks mentah:*
   ```text
   Saya: Bro, gimana cara install postgresql di windows?
   AI: Gampang. Download dulu dari website resminya, lalu jalankan installernya.
   Saya: Oke udah, terus setting passwordnya di mana?
   AI: Pas instalasi nanti akan ada prompt untuk memasukkan password superuser (postgres).
   ```

---

## Tahap 2: Konversi Otomatis Menggunakan AI (Prompting)

Daripada membuat format JSONL manual, lemparkan teks mentah tadi ke AI (seperti ChatGPT, Claude, atau Gemini) menggunakan *Prompt* khusus berikut.

> [!TIP]
> **Copy dan Paste Prompt di bawah ini ke AI Anda:**

```text
Anda adalah data engineer ahli yang bertugas memproses data untuk sistem RAG (Retrieval-Augmented Generation).
Tugas Anda adalah membaca transkrip obrolan mentah di bawah ini dan mengonversinya menjadi format JSONL (JSON Lines).

Setiap baris dalam JSONL harus mewakili satu interaksi pesan dan memiliki struktur JSON yang valid:
{"session_id": "topik_singkat", "timestamp": "YYYY-MM-DDTHH:MM:SS", "role": "user_atau_assistant", "content": "teks pesan"}

Aturan Ketat:
1. "session_id": Buatlah satu ID atau kata kunci yang sama untuk semua baris berdasarkan topik utama percakapan ini (misal: "setup_postgres_windows").
2. "timestamp": Jika waktu asli tidak ada di teks mentah, buatlah timestamp urut fiktif untuk hari ini (misal mulai dari jam 10:00:00 dan bertambah 1 menit setiap pesan).
3. "role": Gunakan "user" untuk pertanyaan/pernyataan dari manusia, dan "assistant" untuk balasan AI.
4. "content": Teks pesan. Bersihkan dari karakter aneh, tetapi jangan ubah makna pesannya. Pastikan karakter kutipan (") di dalam teks di-escape (\") agar tidak merusak format JSON.
5. HANYA OUTPUTKAN TEKS JSONL. Jangan gunakan blockquote markdown (```json), jangan berikan sapaan, jangan berikan penjelasan. Murni baris demi baris JSON.

Berikut adalah teks obrolan mentahnya:

[PASTE_TEKS_MENTAH_OBROLAN_ANDA_DI_SINI]
```

---

## Tahap 3: Menyimpan Hasil (Format JSONL)

1. AI akan menghasilkan output berupa teks berurutan seperti ini:
   ```json
   {"session_id": "setup_postgres", "timestamp": "2026-07-09T10:00:00", "role": "user", "content": "Bro, gimana cara install postgresql di windows?"}
   {"session_id": "setup_postgres", "timestamp": "2026-07-09T10:01:00", "role": "assistant", "content": "Gampang. Download dulu dari website resminya, lalu jalankan installernya."}
   {"session_id": "setup_postgres", "timestamp": "2026-07-09T10:02:00", "role": "user", "content": "Oke udah, terus setting passwordnya di mana?"}
   {"session_id": "setup_postgres", "timestamp": "2026-07-09T10:03:00", "role": "assistant", "content": "Pas instalasi nanti akan ada prompt untuk memasukkan password superuser (postgres)."}
   ```
2. Salin seluruh *output* tersebut.
3. Buka *Workspace* Anda (folder `hermes-knowledge`).
4. Buat file baru bernama `chat_history.jsonl` dan tempelkan isinya ke file tersebut. **Data Anda sekarang sudah terstruktur dan siap!**

---

## Tahap 4: "Melempar" Data ke RAG (Python Script)

Setelah Anda memiliki file `chat_history.jsonl`, langkah terakhir adalah memasukkannya ke Vector Database (misalnya ChromaDB) agar bisa dicari oleh Hermes AI. 

Ini dilakukan melalui skrip Python. Secara garis besar, skrip ini akan melakukan hal berikut (jika Anda ingin saya menulis kodenya, beri tahu saya):

1. **Membaca JSONL**: Skrip membuka `chat_history.jsonl` baris per baris.
2. **Chunking**: Menggabungkan metadata (`session_id`, `role`, `timestamp`) dengan `content` menjadi satu dokumen teks yang utuh.
3. **Embedding & Load**: Menggunakan model embedding (misal: *OpenAI API* atau *Nomic* lokal) untuk mengubah dokumen teks tersebut menjadi vektor, lalu menyimpannya ke koleksi ChromaDB.

Setelah tersimpan di database, setiap kali Anda ngobrol dengan Hermes, ia akan melakukan *similarity search* ke database ini dan menarik baris JSON obrolan lama Anda sebagai referensi!
