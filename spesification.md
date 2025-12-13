System Implementation Blueprint: Double-Hop RAG for Legal Documents

Tujuan Dokumen: Instruksi lengkap bagi AI Assistant untuk membangun sistem RAG Hukum Indonesia dengan fitur Query Reformulation.
Target Model: Google Gemini-2.5-Flash
Metode: Pseudo-Relevance Feedback (Initial Retrieval -> Reformulation -> Final Retrieval).

1. Project Context & Requirements

Anda bertindak sebagai Senior Python AI Engineer. Tugas Anda adalah membangun sistem Retrieval-Augmented Generation (RAG) untuk menjawab pertanyaan hukum menggunakan dataset artikel hukum Indonesia.

Core Logic (The Innovation)

Sistem ini TIDAK menggunakan RAG standar (Retrieve -> Generate). Anda harus mengimplementasikan Double-Hop Retrieval Logic:

Hop 1 (Context Gathering): Cari dokumen kasar menggunakan raw query user.

Reasoning Step: Gunakan LLM untuk membaca dokumen kasar tersebut dan mereformulasi pertanyaan user menjadi Standardized Legal Query.

Hop 2 (Precision Search): Cari ulang dokumen menggunakan reformulated query.

Generation: Jawab pertanyaan menggunakan dokumen dari Hop 2, dengan Enriched Context (memperhatikan Tanggal Terbit & Tags).

Tech Stack Constraints

Language: Python 3.10+

LLM & Embedding: google-generativeai (Gemini-2.5-Flash & text-embedding-004).

Orchestration: LangChain (terbaru).

Vector DB: FAISS (CPU version).

Reranker: CrossEncoder (sentence-transformers).

2. Directory Structure

Buatlah struktur proyek seperti berikut:

legal-rag-system/
├── data/
│   └── hukumonline_sample.json 
├── src/
│   ├── config.py                # API Keys & Constants
│   ├── ingestion.py             # Chunking, Metadata Extraction & Indexing
│   ├── rag_engine.py            # Core Logic (Double-Hop)
│   └── utils.py                 # Helper functions
├── main.py                      # CLI / Streamlit Entry point
└── requirements.txt


3. Step-by-Step Implementation Guide

Step 1: Configuration (src/config.py)

Set up GOOGLE_API_KEY dari environment variable.

Definisikan konstanta:

CHUNK_SIZE = 1000

CHUNK_OVERLAP = 200

EMBEDDING_MODEL = "models/text-embedding-004"

LLM_MODEL = "gemini-2.5-flash"

Step 2: Data Ingestion (src/ingestion.py)

Crucial Update: Fungsi build_index harus memaksimalkan metadata untuk Context Enrichment.

Load Data: Baca file JSON.

Metadata Extraction Strategy:

Jangan hanya simpan teks. Simpan field berikut ke metadata chunk:

source_title: Judul Artikel

source_url: Link

publish_date: Tanggal (Penting untuk validitas hukum)

tags: List tag (joined string)

theme: Kategori hukum

Text Splitting: Gunakan RecursiveCharacterTextSplitter.

Indexing: Simpan ke FAISS.

Step 3: The RAG Engine (src/rag_engine.py)

Helper Method: format_docs_with_metadata(docs)

Buat fungsi helper untuk merapikan tampilan dokumen ke LLM agar metadata terbaca:

def format_docs_with_metadata(docs):
    formatted = []
    for d in docs:
        meta = d.metadata
        # Format teks dengan Header Metadata agar LLM sadar konteks
        text = (
            f"[JUDUL]: {meta.get('source_title', 'Unknown')}\n"
            f"[TANGGAL TERBIT]: {meta.get('publish_date', 'Unknown')}\n"
            f"[KATEGORI]: {meta.get('theme', 'General')} | [TAGS]: {meta.get('tags', '')}\n"
            f"[ISI KONTEN]:\n{d.page_content}\n"
            f"--------------------------------------------------"
        )
        formatted.append(text)
    return "\n\n".join(formatted)


A. initial_retrieval(query, top_k=3)

Search FAISS. Return raw docs.

B. reformulate_query(original_query, context_docs)

Gunakan Gemini-2.5-Flash.

Masukkan formatted_docs (dengan metadata) ke prompt.

System Prompt:

Role: Ahli Hukum Senior.
Tugas: Reformulasi pertanyaan awam menjadi QUERY PENCARIAN HUKUM baku.

Konteks Awal (Perhatikan istilah hukum dalam Judul & Tags):
{context_docs_text}

Pertanyaan User: {original_query}

Output: Query baku saja.


C. final_retrieval_and_rerank(reformulated_query)

Search FAISS (Top 10) -> Rerank (Top 5).

D. generate_answer(query, final_docs)

System Prompt (Updated for Metadata Awareness):

Role: Asisten Hukum AI.
Instruksi: Jawab pertanyaan user berdasarkan referensi berikut.

ATURAN PENTING:
1. Perhatikan [TANGGAL TERBIT]. Jika ada dua aturan yang bertentangan, prioritaskan yang lebih baru.
2. Sebutkan Dasar Hukum (Pasal/UU) yang tercantum dalam konten.
3. Gunakan [TAGS] dan [KATEGORI] untuk memahami konteks spesifik (Pidana/Perdata/Acara).

Dokumen Referensi:
{formatted_docs}

Pertanyaan: {query}


E. process_query(user_input)

Jalankan pipeline: Initial -> Reformulate -> Final -> Generate.

4. Dummy Dataset Structure (Rich Metadata)

Update file data/hukumonline_sample.json dengan struktur lengkap ini untuk testing:

[
  {
    "theme": "ilmu-hukum",
    "title": "Apa Itu Pro Justitia?",
    "link": "[https://www.hukumonline.com/klinik/a/pro-justitia-lt5e006f96d6231/](https://www.hukumonline.com/klinik/a/pro-justitia-lt5e006f96d6231/)",
    "publish_date": "2025-10-24T10:10:00+00:00",
    "tags": ["acara peradilan", "pengadilan", "putusan pengadilan"],
    "content": "Menurut Yan Pramadya Puspa dalam Kamus Hukum, pro justitia artinya demi hukum, untuk hukum atau undang-undang. Dalam praktiknya, istilah pro justitia terdapat dalam dokumen atau surat resmi kepolisian dalam proses penyelidikan dan penyidikan...",
    "category": "Pidana"
  },
  {
    "theme": "pidana",
    "title": "Jerat Hukum Akses WiFi Ilegal",
    "link": "[https://www.hukumonline.com/klinik/wifi-ilegal](https://www.hukumonline.com/klinik/wifi-ilegal)",
    "publish_date": "2024-05-15T09:00:00+00:00",
    "tags": ["UU ITE", "pencurian", "akses ilegal"],
    "content": "Menggunakan WiFi orang lain tanpa izin dapat dikategorikan sebagai tindak pidana. Hal ini diatur dalam Pasal 30 ayat (1) dan (2) UU ITE tentang akses ilegal terhadap sistem elektronik yang merugikan orang lain...",
    "category": "Teknologi"
  }
]


5. Execution Instructions

Install & Setup: pip install -r requirements.txt.

Ingest Data: Jalankan ingestion.py. Pastikan script mengekstrak tags dan publish_date ke dalam metadata FAISS.

Run Main: Jalankan main.py dan perhatikan bagaimana LLM memanfaatkan tanggal/tags dalam jawabannya.

Developer Note:
Pastikan fungsi format_docs_with_metadata benar-benar digunakan saat mengirim konteks ke LLM. Informasi tanggal sangat penting untuk menghindari hallucination aturan hukum yang sudah kadaluarsa.