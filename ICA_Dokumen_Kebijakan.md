# ICA · Dokumentasi Data Dokumen Kebijakan (RAG)

> Dokumen ini menjelaskan **knowledge base kebijakan & SOP kredit** yang dipakai ICA lewat tool agent `policy_search`:
> isi dokumen, format file, cara dokumen diproses (chunking, embedding, retrieval), set evaluasi, dan cara memperbaruinya.
> Data tabel nasabah (5 CSV) dijelaskan terpisah di `ICA_data_metadata.md`.

---

## 1. Ringkasan

| | |
|---|---|
| **Isi** | 11 dokumen kebijakan, SOP, playbook, pedoman, dan FAQ kredit retail (±3.750 kata) |
| **Sifat** | **Dummy / fiktif**, ditulis khusus untuk capstone. Bank: *Bank Sentosa Raya (fiktif)*. Tidak menyalin kebijakan bank mana pun |
| **Format** | Markdown (`.md`) dengan *front matter* metadata, judul `#`, dan pasal/bagian `##` |
| **Lokasi** | Folder `rag_docs/` + `rag_eval_set.json` (26 pertanyaan uji). Zip di Google Drive, file id `1wbn3jfUbHhKLKRwZls9RS39aeHIT2XIY` |
| **Dipakai oleh** | Tool ke-4 agent investigasi: `policy_search` (notebook section 18.2), tab **3 · Kebijakan (RAG)** di aplikasi Gradio |
| **Hubungan dengan data nasabah** | **Tidak ada.** Dokumen berlaku untuk semua nasabah, tidak memakai `customer_id`, dan tidak dipakai model ML |

**Satu kalimat:** saat analis bertanya "DSR nasabah ini masih di bawah batas KTA?", agent mengambil angka DSR nasabah dari
tool data (`credit_exposure`), mencari batas DSR KTA di dokumen ini (`policy_search`), lalu membandingkan keduanya dengan
menyebut sumber `[KBK-KTA]`.

---

## 2. Daftar dokumen

| `doc_id` | Judul | Jenis | Versi · berlaku | Isi utama (pasal/bagian) |
|---|---|---|---|---|
| `KBK-UMUM` | Kebijakan Umum Perkreditan | kebijakan | 4.0 · 2026-01-01 | Prinsip 5C, **DSR umum**, **kewenangan memutus**, penggunaan sistem ICA, larangan kredit, non-diskriminasi |
| `KBK-KTA` | Kebijakan Produk Kredit Tanpa Agunan | kebijakan | 3.1 · 2026-02-01 | Kriteria pemohon, plafon & tenor, **DSR KTA 35%**, bunga per grade risiko, dokumen wajib, pelunasan dipercepat |
| `KBK-KKB` | Kebijakan Kredit Kendaraan Bermotor | kebijakan | 2.3 · 2026-01-15 | Objek pembiayaan, **uang muka minimum**, tenor & usia kendaraan, DSR KKB, asuransi, kendaraan bekas |
| `KBK-KPR` | Kebijakan Kredit Pemilikan Rumah | kebijakan | 5.0 · 2026-03-01 | **LTV**, tenor & usia, DSR KPR, appraisal agunan (KJPP), suku bunga |
| `KBK-KMK` | Kebijakan Kredit Modal Kerja Usaha Kecil | kebijakan | 1.8 · 2025-11-01 | Kriteria usaha, kebutuhan modal kerja & plafon, **DSCR**, laporan keuangan, agunan, tenor |
| `SOP-ANALISA` | SOP Analisa Kredit dengan Sistem ICA | sop | 2.0 · 2026-04-01 | Langkah kerja analis, **SLA keputusan**, penanganan Further Review, flag guardrail, **kode override OVR-01–05** |
| `SOP-INTERVIEW` | SOP Interview Pemohon Baru | sop | 1.2 · 2026-04-01 | 6 pertanyaan baku, aturan follow-up, **larangan meminta PIN/OTP**, indikasi manipulasi, penyimpanan rekaman |
| `SOP-ANTIFRAUD` | SOP Verifikasi Dokumen dan Anti-Fraud | sop | 1.5 · 2026-01-01 | Verifikasi wajib (callback), **red flags** dokumen palsu, eskalasi, kerahasiaan |
| `PB-EW` | Playbook Penanganan Early Warning | playbook | 1.4 · 2026-05-01 | **Tindakan per kondisi DPD**, SLA tindak lanjut alert, alert berulang, asumsi biaya penanganan |
| `PB-RESTRU` | Pedoman Restrukturisasi & Kolektibilitas Internal | pedoman | 2.1 · 2026-01-01 | Kolektibilitas 1–5, syarat & skema restrukturisasi, kewenangan, status setelah restrukturisasi |
| `FAQ-ANALIS` | FAQ Analis Kredit tentang ICA | faq | 1.0 · 2026-06-01 | Skor vs PD, zona Further Review, kenapa EW tidak untuk pemohon baru, siapa menetapkan threshold, dll. |

Dokumen dirancang **konsisten dengan cara kerja ICA**. Contoh:
- `KBK-UMUM` Pasal 5: rekomendasi ICA bukan keputusan, Further Review wajib diputus minimal oleh Analis Kredit Senior.
- `FAQ-ANALIS` menjelaskan beda skor risiko dan PD terkalibrasi.
- `PB-EW` menjadi acuan tindakan saat ICA mengeluarkan ALERT.

### Contoh isi (angka kunci yang sering ditanya)

| Topik | Ketentuan | Sumber |
|---|---|---|
| DSR maksimum umum | Penghasilan < Rp 10 jt: **40%** · ≥ Rp 10 jt: **50%** | `KBK-UMUM` Pasal 3 |
| DSR maksimum KTA | **35%** untuk semua tingkat penghasilan (lebih ketat dari DSR umum) | `KBK-KTA` Bagian 4 |
| Kewenangan memutus | ≤ Rp 100 jt Analis Kredit Senior · ≤ Rp 500 jt Kepala Unit Kredit · ≤ Rp 2 M Komite Cabang · > Rp 2 M Komite Pusat | `KBK-UMUM` Pasal 4 |
| Kode override | OVR-01 agunan tambahan · OVR-02 penjamin · OVR-03 data ICA tidak lengkap · **OVR-04 relasi bisnis strategis (wajib persetujuan Kepala Unit Kredit)** · OVR-05 koreksi input | `SOP-ANALISA` Bagian 6 |
| Uang muka mobil bekas | minimal **25%** | `KBK-KKB` |
| LTV KPR rumah kedua | maksimal **80%** | `KBK-KPR` |
| DSCR modal kerja | minimal **1,25** | `KBK-KMK` |
| Kolektibilitas 3 | DPD **91–120** hari | `PB-RESTRU` |
| Penyimpanan transkrip interview | **5 tahun** | `SOP-INTERVIEW` |

---

## 3. Format file

Setiap dokumen adalah satu file `.md`:

```markdown
---
doc_id: KBK-UMUM
judul: Kebijakan Umum Perkreditan
jenis: kebijakan            # kebijakan | sop | playbook | pedoman | faq
versi: 4.0
berlaku: 2026-01-01
pemilik: Divisi Manajemen Risiko Kredit
bank: Bank Sentosa Raya (fiktif)
---

# Kebijakan Umum Perkreditan

## Pasal 3 Batas Debt Service Ratio (DSR) Umum
DSR adalah total cicilan per bulan ... dibagi penghasilan bersih per bulan.

| Penghasilan bersih per bulan | DSR maksimum umum |
|---|---|
| di bawah Rp 10 juta | 40% |
...
```

| Elemen | Fungsi saat diproses |
|---|---|
| *Front matter* (`doc_id`, `judul`, `jenis`, `versi`, `berlaku`, ...) | Menjadi metadata setiap chunk. `doc_id` dipakai untuk sitasi `[doc_id]` dan filter |
| `#` judul dokumen | Disimpan sebagai `judul_dok`. Teks pembuka sebelum `##` pertama (judul + disclaimer) dibuang |
| `##` Pasal / Bagian | Batas utama chunk. Disimpan sebagai `bagian`, misalnya "Pasal 3 Batas Debt Service Ratio (DSR) Umum" |
| Tabel & daftar | Dipertahankan apa adanya di dalam chunk |
| Rujukan antar-dokumen | Ditulis eksplisit, misalnya "lebih ketat daripada batas DSR umum pada KBK-UMUM Pasal 3" |

---

## 4. Cara dokumen diproses (pipeline RAG)

```
rag_docs/*.md ──► parse front matter ──► split per ## (pasal/bagian) ──► split 800 karakter (overlap 120)
                                                                              │
                                     header konteks "[doc_id · judul · bagian]" ditempel di awal chunk
                                                                              │
                      ┌───────────────────────────────────────────────────────┴──────────────┐
                      ▼                                                                      ▼
        Dense: BAAI/bge-m3 (ternormalisasi)                               Sparse: BM25 (tokenizer Indonesia)
        FAISS IndexHNSWFlat (cosine)                                      kode "OVR-04" tetap utuh
                      └──────────────────────► hybrid_retrieve() ◄───────────────────────────┘
                                              Reciprocal Rank Fusion
                                              (α = bobot dense, top-3)
                                                       │
                                     guardrail: ambang cosine, potong 700 karakter, sitasi [doc_id]
                                                       ▼
                                      tool policy_search → agent → jawaban analis
```

| Tahap | Pengaturan | Alasan |
|---|---|---|
| Chunking 1 | `MarkdownHeaderTextSplitter` pada `#` dan `##` | Satu pasal/bagian = satu kesatuan aturan, jangan dipotong di tengah |
| Chunking 2 | `RecursiveCharacterTextSplitter`, 800 karakter, overlap 120 | Pasal panjang dipecah tanpa kehilangan konteks. Chunk < 40 karakter dibuang |
| Header konteks | `[doc_id · judul · bagian]` di awal setiap chunk | Chunk tetap bermakna saat berdiri sendiri. Membantu BM25 dan embedding mengenali topik |
| Hasil | 11 dokumen → **68 chunk** (`chunk_id` = `DOC_ID#nn`) | |
| Embedding | `BAAI/bge-m3`, `normalize_embeddings=True` | Multibahasa (bagus untuk Bahasa Indonesia). Vektor ternormalisasi → inner product = cosine |
| Index dense | FAISS `IndexHNSWFlat`, M = 32, efConstruction = 200, efSearch = 64 | Graph HNSW: pencarian cepat dan akurat. Untuk 68 vektor sebenarnya flat index cukup, HNSW dipilih agar siap untuk ribuan dokumen |
| Sparse | `BM25Retriever` + tokenizer Indonesia (stopword, kode seperti `OVR-04`/`KBK-KTA` disimpan utuh **dan** dipecah) | Menangkap kata kunci persis: kode override, angka, istilah (DSCR, LTV, PTP) |
| Fusion | `hybrid_retrieve(query, k=3, alpha=0.5)`: RRF dari 20 kandidat teratas masing-masing (k_rrf = 60) | Dense kuat di parafrase ("mobil second" ≈ "kendaraan bekas"), sparse kuat di kode & angka. `alpha` = bobot dense (0,5 = setara) |
| Guardrail | Skor cosine teratas di bawah ambang → "tidak ada ketentuan terkait di knowledge base" | Mencegah agent mengarang aturan untuk pertanyaan di luar cakupan |
| Guardrail | Isi chunk dibatasi 700 karakter, sitasi `[doc_id]` wajib, kutipan aturan boleh tapi agent dilarang membuat keputusan sendiri | Output ringkas, bisa ditelusuri, tetap *human-in-the-loop* |

**Ambang cosine** (`KB_DENSE_MIN`, awal 0,40) dikalibrasi otomatis dari eval set: titik tengah antara skor terendah
pertanyaan dalam cakupan dan skor tertinggi pertanyaan di luar cakupan.

---

## 5. Set evaluasi (`rag_eval_set.json`)

26 pertanyaan dengan jawaban yang diketahui, dipakai untuk mengukur kualitas retrieval dan mengkalibrasi ambang cosine.

| Field | Isi |
|---|---|
| `id` | E01–E26 |
| `type` | `angka` (9) · `parafrase` (10) · `istilah` (3) · `kode` (2) · `di_luar_cakupan` (2) |
| `question` | Pertanyaan dengan gaya analis, termasuk bahasa informal ("mobil second", "nggak") |
| `gold_doc` | `doc_id` yang seharusnya ditemukan (`null` untuk di luar cakupan) |
| `gold_text` | Potongan teks yang harus ada di chunk yang ditemukan |
| `answer_keywords` | Kata/angka yang harus muncul di jawaban akhir (evaluasi generasi) |

Contoh:

| id | Tipe | Pertanyaan | Dokumen | Jawaban harus memuat |
|---|---|---|---|---|
| E01 | angka | Berapa batas DSR maksimum untuk produk KTA? | KBK-KTA | 35% |
| E02 | parafrase | Kalau beli mobil second, uang mukanya minimal berapa persen? | KBK-KKB | 25% |
| E03 | kode | Kode override OVR-04 itu untuk alasan apa dan butuh persetujuan siapa? | SOP-ANALISA | relasi bisnis, Kepala Unit Kredit |
| E07 | parafrase | Nasabah sudah telat bayar 45 hari, apa yang harus dilakukan tim collection? | PB-EW | SP1, kunjungan |
| E15 | istilah | Apa yang dimaksud PTP dalam penanganan alert? | PB-EW | promise to pay |
| E25 | di_luar_cakupan | Berapa harga emas hari ini? | – | (harus ditolak) |

**Metrik:**

| Metrik | Definisi | Target | Tempat |
|---|---|---|---|
| Hit@3 | Proporsi pertanyaan dalam cakupan yang `gold_doc` + `gold_text`-nya ada di 3 chunk teratas `hybrid_retrieve` | ≥ 0,80 | Output section 18.2, guardrail `policy_search: Hit@3`, scorecard section 23 |
| Penolakan di luar cakupan | Pertanyaan di luar cakupan berada di bawah ambang cosine (dipakai juga untuk kalibrasi ambang) | semua (2) | Section 18.2 |
| Tool correctness & faithfulness kasus kebijakan | 5 kasus kebijakan di evaluasi agent (DeepEval) | ≥ 0,90 · ≥ 0,80 | Section 22, scorecard |

Notebook RAG terpisah (`RAG_Kebijakan_Kredit_Hybrid.ipynb`) membandingkan dense vs BM25 vs hybrid dan men-*tuning* α
(0,2–0,8) dengan set evaluasi yang sama.

---

## 6. Cara dipakai di ICA

| Jalur | Cara kerja |
|---|---|
| **Agent investigasi (tab 2)** | Pertanyaan berisi kata kunci kebijakan (batas, syarat, kewenangan, SLA, override, playbook, restrukturisasi, LTV, uang muka, ...) → `policy_search` dipanggil otomatis (prefetch), atau dipilih sendiri oleh LLM. Pertanyaan yang membandingkan kondisi nasabah dengan aturan juga memanggil tool data nasabah |
| **Pencarian langsung (tab 3)** | Analis mengetik topik → 3 potongan dokumen teratas beserta `[doc_id] judul · bagian` |
| **Pemohon baru** | `policy_search` boleh dipakai (tidak butuh histori nasabah). Tool data nasabah tidak |
| **Format jawaban** | Sitasi `[doc_id]` di teks + baris `Kebijakan: KBK-KTA · Bagian 4 Batas DSR KTA` di bawah jawaban |
| **Batas peran** | Aturan kebijakan adalah hal yang perlu **dicek** analis. Agent menulis "menurut [KBK-KTA], DSR 42% tidak memenuhi ketentuan", bukan "wajib ditolak". Kutipan aturan dari dokumen diizinkan oleh guardrail, kalimat keputusan buatan agent tidak |

---

## 7. Menambah atau memperbarui dokumen

1. Buat file `.md` baru di `rag_docs/` dengan *front matter* lengkap (minimal `doc_id`, `judul`, `jenis`, `versi`, `berlaku`).
2. Tulis isi dengan judul `#` dan setiap aturan sebagai `## Pasal/Bagian ...`. Satu aturan per bagian.
3. Untuk perubahan aturan: naikkan `versi` dan ubah `berlaku`. Jangan simpan dua versi aktif dokumen yang sama.
4. Tambahkan 1–2 pertanyaan uji ke `rag_eval_set.json` untuk dokumen baru (`gold_doc`, `gold_text`, `answer_keywords`).
5. Jalankan ulang section 18.2. Index dibangun ulang otomatis (< 1 menit) dan Hit@3 + ambang cosine dihitung ulang.

---

## 8. Batasan

| # | Hal | Dampak |
|---|---|---|
| 1 | Dokumen fiktif dan ringkas (±250–600 kata per dokumen) | Kebijakan bank sungguhan jauh lebih panjang dan berlapis. Chunking & ambang perlu dikalibrasi ulang |
| 2 | Angka kebijakan tidak diselaraskan dengan distribusi data sintetis | Contoh: median DSR di data 0,44, di atas batas KTA 35%. Kebijakan dipakai untuk pengecekan analis, **bukan** sebagai fitur atau aturan model ML |
| 3 | Eval set kecil (26 pertanyaan, 24 dalam cakupan) | Hit@3 punya ketidakpastian besar. Cukup untuk regresi cepat, belum untuk klaim akurasi |
| 4 | Tidak ada pengecekan tanggal berlaku saat retrieval | Kalau ada dua versi, keduanya bisa terambil. Aturan: hanya satu versi aktif di folder |
| 5 | Rujukan antar-dokumen tidak diikuti otomatis | Jika jawaban butuh dua pasal yang saling merujuk, keduanya harus masuk top-3. Pengembangan: graph rujukan antar-pasal |
