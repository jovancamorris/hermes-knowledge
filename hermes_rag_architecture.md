# Arsitektur Hermes AI Orchestrator: Manajemen Memori dengan RAG

Menjadikan Hermes sebagai "AI Orchestrator" dengan kemampuan *Memory Management* berbasis RAG (Retrieval-Augmented Generation) akan membuatnya menjadi asisten yang cerdas karena ia bisa belajar dari setiap percakapan.

Berikut adalah panduan arsitektur dan langkah-langkah bagaimana Anda bisa menyimpan obrolan tersebut dan mengubahnya menjadi materi RAG.

---

## Teori Dasar: Mengapa RAG, Bukan Fine-Tuning?

Sebelum masuk ke implementasi, penting untuk memahami **mengapa RAG dipilih** sebagai mekanisme memori, bukan pendekatan *fine-tuning* atau *retraining* model.

| Aspek | Fine-Tuning | RAG |
|---|---|---|
| **Cara kerja** | Memodifikasi bobot (weight) model secara permanen | Menyuntikkan konteks eksternal ke dalam prompt saat inferensi |
| **Biaya** | Sangat mahal (GPU hours, data besar) | Murah — hanya butuh vector DB dan embedding |
| **Update memori** | Harus retrain ulang setiap ada data baru | Real-time — tambah data baru ke Vector DB langsung aktif |
| **Risiko** | *Catastrophic forgetting* — model bisa lupa hal lain | Tidak ada risiko ke model asli |
| **Kontrol** | Sulit menelusuri "dari mana" model tahu sesuatu | Transparan — sumber konteks bisa dilacak langsung |

> [!IMPORTANT]
> RAG adalah pilihan yang jauh lebih **praktis, murah, dan aman** untuk menambah memori ke AI. Fine-tuning hanya relevan jika Anda ingin mengubah *gaya bicara* atau *domain spesialis* model, bukan untuk menyimpan memori percakapan.

### Teori: Arsitektur RAG (Lewis et al., 2020)

RAG diperkenalkan oleh Meta AI dalam paper *"Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"*. Ide intinya adalah menggabungkan dua komponen:

```
Query User
    │
    ▼
[Retriever] ──────────────── Knowledge Base (Vector DB)
    │                              ▲
    │  (context relevan)           │ (dokumen yang di-index)
    ▼                              │
[Generator / LLM] ◄────────────────
    │
    ▼
  Jawaban Final
```

Dengan arsitektur ini, LLM tidak "mengingat" — ia **membaca ulang** informasi yang relevan setiap kali dibutuhkan, sehingga memori bisa diperbarui tanpa menyentuh model.

---

## 1. Format Penyimpanan Chat (Raw Data)

> [!TIP]
> Untuk menyimpan *raw chat* (data mentah obrolan), **JSON atau JSONL (JSON Lines)** adalah format yang paling direkomendasikan, bukan Markdown (MD).

**Mengapa JSON/JSONL?**
- **Struktur Metadata:** Dalam RAG, metadata sangat penting. Anda tidak hanya menyimpan "teks", tetapi Anda perlu tahu *siapa* yang bicara (User vs AI), *kapan* (Timestamp), *Topik*, dan *Session ID*. JSON dapat menyimpan ini dengan rapi.
- **Mudah Di-parsing:** Sangat mudah dibaca oleh skrip Python untuk dipotong-potong (chunking) nantinya.

**Contoh Format Penyimpanan (JSONL):**
```json
{"session_id": "12345", "timestamp": "2026-07-09T10:00:00", "role": "user", "content": "Tolong jelaskan cara setup database PostgreSQL."}
{"session_id": "12345", "timestamp": "2026-07-09T10:00:05", "role": "assistant", "content": "Tentu, langkah pertama adalah menginstal..."}
```

> [!NOTE]
> Markdown (.md) lebih cocok digunakan untuk menyimpan hasil "Kesimpulan" (Summary) dari sebuah sesi obrolan yang panjang yang mudah dibaca manusia, bukan untuk *raw log* setiap pesan.

### Teori: Metadata Filtering dalam RAG

Metadata bukan sekadar label — ia adalah **filter pencarian yang sangat powerful**. Dalam RAG modern, proses retrieval tidak hanya dilakukan berdasarkan kemiripan semantik (vektor), tetapi juga dikombinasikan dengan **metadata pre-filtering**:

```
Query: "Apa yang kita diskusikan tentang PostgreSQL?"
    │
    ├─ [Metadata Filter]  → session_id = "xxx" AND role = "user"
    └─ [Semantic Filter]  → embedding similarity > 0.75
             │
             ▼
        Hasil yang relevan & tepat sasaran
```

Teknik ini disebut **Hybrid Search** atau **Filtered Vector Search**, dan jauh lebih presisi dibanding pure semantic search.

---

## 2. Alur Mengubah Obrolan Menjadi Materi RAG (Pipeline)

Untuk membuat memori dari obrolan ini, Anda membutuhkan sebuah *pipeline* (alur kerja). Anda tidak melatih (training/fine-tuning) modelnya ulang, melainkan menggunakan RAG. Berikut alurnya:

```
[Raw JSONL] → [Chunking] → [Embedding] → [Vector DB] → [Retrieval → System Prompt]
```

---

### Tahap 1: Ekstraksi & Chunking (Pemotongan)

Obrolan yang panjang tidak bisa langsung dimasukkan ke RAG. Anda perlu mengelompokkan pesan yang saling berkaitan atau merangkumnya.
- **Teknik:** Gunakan skrip (biasanya Python dengan LangChain atau LlamaIndex) untuk mengambil file JSON tadi, lalu memotong teksnya (*chunking*) menjadi blok-blok teks yang bermakna.

#### Teori: Strategi Chunking

Chunking adalah seni memotong teks agar setiap potongan mengandung **satu ide yang utuh**. Ada beberapa strategi utama:

| Strategi | Cara Kerja | Cocok untuk |
|---|---|---|
| **Fixed-size Chunking** | Potong setiap N karakter/token, dengan overlap M token | Teks panjang yang homogen |
| **Sentence-based Chunking** | Potong per kalimat atau per paragraf | Artikel, dokumen terstruktur |
| **Semantic Chunking** | Gunakan AI untuk mendeteksi batas topik secara semantik | Chat log, obrolan multi-topik |
| **Session-based Chunking** | Gabungkan seluruh pesan dalam satu sesi menjadi satu chunk | Chat history dengan session_id |

Untuk chat history Hermes, **kombinasi Session-based + Semantic** adalah yang paling ideal:

```python
# Contoh: gabungkan pesan dalam satu sesi menjadi chunk
def build_chunk_from_session(messages: list[dict]) -> str:
    return "\n".join(
        f"[{m['role'].upper()}]: {m['content']}"
        for m in messages
    )
```

> [!TIP]
> **Chunk Overlap** — saat menggunakan fixed-size chunking, selalu tambahkan overlap sekitar 10–20% dari ukuran chunk (misal: chunk 512 token, overlap 64 token). Ini mencegah informasi penting terpotong di batas chunk.

---

### Tahap 2: Embedding

Setelah dipotong, teks tersebut harus diubah menjadi angka (vektor) agar AI bisa mencari kesamaan maknanya.
- **Model Embedding:** Anda bisa menggunakan model embedding *open-source* lokal (seperti `nomic-embed-text` via Ollama) atau API (seperti OpenAI `text-embedding-3`).

#### Teori: Vector Space & Representasi Semantik

Model embedding mengonversi teks menjadi titik di dalam **ruang vektor berdimensi tinggi** (biasanya 768–3072 dimensi). Teks yang **bermakna mirip** akan menghasilkan vektor yang **berdekatan** secara geometris.

```
Ruang Vektor (disederhanakan menjadi 2D):

        "cara install PostgreSQL"  ●
                                    \
        "setup database postgres"   ● ← sangat dekat (mirip makna)
                                    
        
        "resep nasi goreng"         ●  ← jauh (tidak berkaitan)
```

**Kemiripan diukur dengan Cosine Similarity:**

```
cosine_similarity(A, B) = (A · B) / (|A| × |B|)
```

- Nilai `1.0` → identik secara semantik
- Nilai `0.0` → tidak berkaitan sama sekali
- Nilai `-1.0` → berlawanan makna (jarang terjadi pada kalimat normal)

**Perbandingan Model Embedding:**

| Model | Dimensi | Tipe | Kekuatan |
|---|---|---|---|
| `nomic-embed-text` | 768 | Lokal (Ollama) | Gratis, privasi terjaga |
| `text-embedding-3-small` | 1536 | OpenAI API | Cepat, murah, akurat |
| `text-embedding-3-large` | 3072 | OpenAI API | Paling akurat, mahal |
| `mxbai-embed-large` | 1024 | Lokal (Ollama) | Alternatif lokal berkualitas tinggi |

> [!NOTE]
> **Jangan campur model embedding yang berbeda** dalam satu Vector DB. Vektor dari model berbeda tidak bisa dibandingkan satu sama lain, karena mereka hidup di "ruang matematika" yang berbeda.

---

### Tahap 3: Vector Database (Penyimpanan RAG)

Teks yang sudah di-embed harus disimpan ke dalam **Vector Database**. Ini adalah tempat penyimpanan akhir untuk materi RAG Anda.
- **Rekomendasi Vector DB Lokal:** 
  - **ChromaDB** (Sangat mudah untuk Python, berbasis file lokal).
  - **Qdrant** atau **Milvus** (Jika skalanya mulai besar).

#### Teori: Algoritma HNSW (Cara Vector DB Mencari dengan Cepat)

Mencari vektor yang paling mirip di antara jutaan vektor secara *brute-force* (hitung semua jarak) sangat lambat. Vector DB menggunakan algoritma **ANN (Approximate Nearest Neighbor)**, yang paling populer adalah **HNSW (Hierarchical Navigable Small World)**.

HNSW membangun struktur **graf berlapis** seperti ini:

```
Layer 2 (sparse):  A ─────────────── E
                   │                 │
Layer 1 (medium):  A ──── B ──── D ──E
                   │      │     │    │
Layer 0 (dense):   A──B──C──D──E──F──G  ← semua vektor ada di sini
```

- **Pencarian dimulai dari layer atas** (yang "jarang") untuk navigasi cepat ke zona yang benar.
- Lalu **turun ke layer bawah** untuk pencarian presisi tinggi di zona tersebut.
- **Trade-off:** Konsumsi RAM lebih tinggi, namun kecepatan kueri sangat cepat (sub-millisecond untuk jutaan vektor).

---

### Tahap 4: Retrieval (Pemanggilan Memori)

Ketika user bertanya sesuatu yang baru ke Hermes Orchestrator:
1. Pertanyaan user di-embed.
2. Sistem mencari ke Vector Database obrolan lama yang maknanya paling mirip.
3. Teks obrolan lama tersebut ditarik dan diselipkan ke *System Prompt* Hermes sebagai konteks (Memory).

#### Teori: Context Window & Retrieval Trade-off

LLM hanya bisa membaca sejumlah token tertentu sekaligus — ini disebut **Context Window**. Karena itu, retrieval bukan hanya tentang "ambil yang paling mirip", tetapi tentang **memilih konteks yang paling berharga** dalam ruang yang terbatas.

```
Context Window LLM (misal: 128k token)
┌──────────────────────────────────────────────┐
│ [System Prompt] ~500 token                   │
│ [Retrieved Memory Chunks] ~2000 token ← RAG  │
│ [Chat History Saat Ini] ~5000 token          │
│ [Query User Sekarang] ~100 token             │
└──────────────────────────────────────────────┘
```

**Strategi Retrieval yang umum digunakan:**

| Strategi | Deskripsi |
|---|---|
| **Top-K Retrieval** | Ambil K chunk dengan similarity tertinggi (misal: top-5) |
| **Threshold Retrieval** | Ambil semua chunk dengan similarity > nilai ambang (misal: > 0.75) |
| **MMR (Maximal Marginal Relevance)** | Ambil chunk yang relevan *sekaligus* beragam (hindari redundansi) |
| **Hybrid Search** | Gabungkan hasil vector search + keyword search (BM25) untuk akurasi lebih tinggi |

> [!TIP]
> **MMR** sangat berguna untuk chat memory karena obrolan sering memiliki banyak pesan yang berulang atau mirip. MMR memastikan konteks yang dipilih tetap beragam dan informatif.

---

## 3. Implementasi di Hermes Agent

Karena Hermes sangat pintar dalam **Function Calling / Tool Usage**, cara terbaik untuk mengimplementasikan ini adalah dengan membuat **Custom Tool** (Alat Kustom) di dalam Hermes.

Anda bisa membuat dua alat bantu (Tools):

1. `save_to_memory(topic, content)`: Alat yang bisa dipanggil oleh Hermes secara otonom ketika ia merasa ada informasi penting dari percakapan saat ini yang harus diingat untuk masa depan. Alat ini akan menyimpannya ke Vector DB.
2. `search_memory(query)`: Alat yang bisa dipanggil Hermes jika user bertanya tentang hal di masa lalu, dan Hermes butuh mengambil konteks dari Vector DB.

### Teori: Agentic Memory — Jenis-Jenis Memori AI

Dalam AI Agent modern, memori dikategorikan menjadi beberapa jenis berdasarkan sifat dan durasinya:

| Jenis Memori | Analogi Manusia | Implementasi di Hermes | Durasi |
|---|---|---|---|
| **Sensory Memory** | Ingatan sesaat (apa yang baru saja dilihat) | Token input dalam satu API call | Satu request |
| **Working Memory** | Memori kerja jangka pendek | Chat history dalam context window aktif | Durasi sesi |
| **Episodic Memory** | Ingatan kejadian spesifik ("kemarin saya makan soto") | `search_memory()` → Vector DB chat log | Permanen |
| **Semantic Memory** | Pengetahuan umum ("soto adalah makanan") | RAG dari dokumen pengetahuan (hermes-knowledge repo ini) | Permanen |
| **Procedural Memory** | Keahlian otomatis ("cara mengetik") | System Prompt & instruksi tetap di config | Permanen |

Dengan dua tools `save_to_memory` dan `search_memory`, Hermes mengimplementasikan **Episodic Memory** secara dinamis — persis seperti cara manusia mengingat dan memanggil kembali pengalaman spesifik dari masa lalu.

```
Percakapan Baru
      │
      ▼
Hermes menilai: "Apakah ini informasi penting?"
      │
      ├─ Ya → save_to_memory(topic, content) → Vector DB
      │
      └─ User bertanya tentang masa lalu?
              │
              └─ Ya → search_memory(query) → ambil konteks → jawab
```

> [!IMPORTANT]
> **Langkah Selanjutnya yang Disarankan:**
> Jika Anda ingin mengimplementasikan ini di dalam repositori proyek Anda, mulailah dengan membuat *script* Python sederhana untuk:
> 1. Membuat fungsi *logger* percakapan yang menulis ke file `history.jsonl`.
> 2. Membuat skrip yang membaca `history.jsonl`, melakukan *embedding*, dan memasukkannya ke Vector Database.
