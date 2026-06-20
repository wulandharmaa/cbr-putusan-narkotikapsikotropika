# CBR Putusan Narkotika Mahkamah Agung RI

Sistem **Case-Based Reasoning (CBR)** untuk retrieval dan prediksi putusan perkara narkotika berdasarkan putusan Mahkamah Agung Republik Indonesia. Pipeline mencakup ekstraksi PDF, representasi kasus, retrieval berbasis TF-IDF & IndoBERT, prediksi solusi, hingga evaluasi model.

## Anggota

Wulan Dharma Putri - 202310370311279

Rajwaa Rindu Izzany - 202310370311282

---

## Struktur Repository

```
├── data/
│   ├── raw/              # Teks hasil ekstraksi PDF putusan MA RI (.txt)
│   └── processed/        # Hasil representasi kasus
│       ├── cases.csv
│       └── cases_cleaned.csv
├── notebooks/
│   ├── 01_membangun_case_base.ipynb
│   ├── 02_case_representation.ipynb
│   ├── 03_case_retrieval.ipynb
│   ├── 04_case_solution_reuse.ipynb
│   └── 05_model_evaluation.ipynb
└── README.md
```

---

## Dataset

Dataset berasal dari putusan perkara narkotika Mahkamah Agung RI yang telah diekstraksi dari PDF menjadi file teks.

| | Keterangan |
|---|---|
| **`data/raw/`** | File `.txt` hasil ekstraksi, sudah tersedia di repository ini |
| **`data/processed/cases.csv`** | Hasil ekstraksi fitur dari Tahap 2 |
| **`data/processed/cases_cleaned.csv`** | Hasil pembersihan data dari Tahap 3 |
| **PDF asli** | Tersimpan terpisah di Google Drive → [Unduh di sini](https://drive.google.com/drive/folders/1s3ov6eTw_IvPqY_H2cL4fvQvZLW2v_5B?usp=sharing) |

> Untuk menjalankan pipeline dari awal (Tahap 1), unduh PDF asli dari tautan Google Drive di atas dan taruh di Google Drive pribadi. File `.txt` di `data/raw/` dapat langsung digunakan mulai dari Tahap 2.

---

## Instalasi & Persyaratan

### Persyaratan Sistem
- Python 3.8+
- Google Colab (direkomendasikan) atau Jupyter Notebook lokal
- Akses Google Drive (untuk mount dataset)

### Library Utama

Semua library diinstal otomatis di cell pertama setiap tahap. Daftar dependensi utama:

```
pdfplumber
pypdf
transformers
torch
pillow<12
scikit-learn
pandas
numpy
matplotlib
indobenchmark/indobert-base-p1
```

Jika menjalankan secara lokal (bukan Colab), instal manual:

```bash
pip install pdfplumber pypdf "pillow<12" transformers torch scikit-learn pandas numpy matplotlib
```

---

## Cara Menjalankan

Jalankan notebook **secara berurutan**. Setiap notebook menghasilkan output yang digunakan oleh notebook berikutnya.
---

### Tahap 1 — Membangun Case Base
**Cell:** 1–9  
**Input:** PDF putusan (dari Google Drive — [unduh di sini](https://drive.google.com/drive/folders/1s3ov6eTw_IvPqY_H2cL4fvQvZLW2v_5B?usp=sharing))  
**Output:** File `case_001.txt` … `case_030.txt` di folder `SUBCPMK-3/raw/` → sudah tersedia

> Tahap ini hanya perlu dijalankan ulang jika ingin memproses PDF baru. File `.txt` sudah tersedia di `data/raw/`, sehingga bisa langsung lanjut ke Tahap 2.

Jika tetap ingin menjalankan Tahap 1:

1. Jalankan **Cell 1** — mount Google Drive:
   ```python
   from google.colab import drive
   drive.mount('/content/drive')
   ```
2. **Cell 2** — install library (`pdfplumber`, `pypdf`).
3. **Cell 3** — konfigurasi path (**WAJIB diisi**):
   ```python
   PDF_FOLDER = "SUBCPMK-3/DATASET"   # folder PDF di Google Drive
   # INPUT_DIR dan OUTPUT_DIR diturunkan otomatis dari nilai ini
   ```
4. **Cell 4** — setup logging (file log disimpan di `SUBCPMK-3/logs/`).
5. **Cell 5** — definisi fungsi ekstraksi & pembersihan teks (watermark, header/footer, disclaimer MA RI).
6. **Cell 6** — cek dan tampilkan daftar PDF yang ditemukan.
7. **Cell 7** — jalankan proses utama: ekstraksi → pembersihan → validasi kelengkapan ≥ 80% → simpan `.txt`.
8. **Cell 8** — tampilkan ringkasan hasil (OK / PERLU_CEK / GAGAL).
9. **Cell 9** *(opsional)* — preview perbandingan teks sebelum dan sesudah pembersihan.

| Status | Arti |
|--------|------|
| ✅ OK | Kelengkapan ≥ 80%, elemen struktural lengkap |
| ⚠️ PERLU CEK | Kelengkapan < 80% atau elemen struktural kurang |
| ❌ GAGAL | Tidak ada teks (PDF terscan / terkunci / rusak) |

---

### Tahap 2 — Case Representation
**Cell:** 10–16  
**Input:** File `.txt` dari Tahap 1 (`SUBCPMK-3/raw/`)  
**Output:** `SUBCPMK-3/processed/cases.csv`

1. **Cell 10** — install tambahan (`pandas`, `scikit-learn`).
2. **Cell 11** — konfigurasi path output (**sesuaikan jika perlu**):
   ```python
   PROCESSED_DIR = Path("/content/drive/MyDrive/SUBCPMK-3/processed")
   ```
   Cell ini juga menampilkan jumlah file `.txt` yang ditemukan sebagai konfirmasi.
3. **Cell 12** — definisi fungsi ekstraksi metadata & konten kunci (nomor perkara, tanggal, pasal, dakwaan, amar, barang bukti, dll.).
4. **Cell 13** — proses semua file `.txt` → ekstraksi fitur → simpan ke list `rows`.
5. **Cell 14** — ekspor ke `cases.csv` (UTF-8 BOM, semua field di-quote untuk mencegah kerusakan CSV).
6. **Cell 15** — preview dan validasi kelengkapan setiap kolom (kolom < 70% terisi ditandai ⚠️).
7. **Cell 16** — statistik ringkas: distribusi jenis perkara, min/max/rata-rata word count.

---

### Tahap 3 — Case Retrieval
**Cell:** 17–28  
**Input:** `SUBCPMK-3/processed/cases.csv`  
**Output:** Fungsi `retrieve(query, k)`, `SUBCPMK-3/eval/queries.json`, script `03_retrieval.py`

1. **Cell 17** — install library (`scikit-learn`, `transformers`, `torch`, `pillow<12`).
   > ⚠️ Jika Pillow masih versi 12.x setelah instalasi, lakukan **Restart session** lalu jalankan ulang Cell 17 sebelum melanjutkan.
2. **Cell 18** — konfigurasi path & muat `cases.csv` (**sesuaikan jika perlu**):
   ```python
   PROCESSED_DIR = Path("/content/drive/MyDrive/SUBCPMK-3/processed")
   ```
3. **Cell 18a** — re-ekstraksi `amar_putusan` & `dakwaan_ringkas` langsung dari teks mentah Tahap 1 (memperbaiki hasil ekstraksi Tahap 2 yang kurang akurat). Jika folder `.txt` tidak ditemukan, cell ini dilewati otomatis.
4. **Cell 18b** — pembersihan kolom kunci (`dakwaan_ringkas`, `barang_bukti`, `amar_putusan`, `top_terms`) dari artefak PDF sebelum dipakai untuk retrieval.
5. **Cell 19** — bangun representasi teks per kasus (`text_repr`) dengan menggabungkan kolom-kolom penting.
6. **Cell 20** — representasi vektor **TF-IDF** (`TfidfVectorizer` dari scikit-learn).
7. **Cell 21** — representasi vektor **IndoBERT** embedding (`indobenchmark/indobert-base-p1`, cosine similarity). Proses ini membutuhkan waktu lebih lama; disarankan menggunakan **GPU runtime** (`Runtime → Change runtime type → T4 GPU`).
8. **Cell 22** — splitting data train/test **80:20** (stratified per `jenis_perkara`).
9. **Cell 23** — model klasifikasi **SVM** (`LinearSVC`) & **Naive Bayes** (`ComplementNB`) pada representasi TF-IDF.
10. **Cell 24** — model retrieval IndoBERT embedding (cosine k-NN).
11. **Cell 25** — definisi fungsi `retrieve(query, k)`: preprocess → vektor → cosine-similarity → top-k.
12. **Cell 26** — bangun `queries.json` berisi 8–10 query uji + ground-truth (protokol NO-LEAK).
13. **Cell 27** — evaluasi fungsi `retrieve()` pada `queries.json`: tampilkan metrik **Hit@k**, **Recall@k**, **MRR**.
14. **Cell 28** — ekspor script `03_retrieval.py` ke Google Drive.

---

### Tahap 4 — Case Solution Reuse
**Cell:** 29–33  
**Input:** Data & model dari Tahap 3 (variabel in-memory: `df`, `retrieve`)  
**Output:** `SUBCPMK-3/results/predictions.csv`, script `04_predict.py`

1. **Cell 29** — ekstrak solusi dari setiap kasus → dict `case_solutions` berisi `verdict`, `penjara_bulan`, `denda` yang diparsing dari `amar_putusan`. Path output:
   ```python
   RESULTS_DIR = Path("/content/drive/MyDrive/SUBCPMK-3/results")
   ```
2. **Cell 30** — implementasi algoritma prediksi: **majority vote** (tiap kasus top-k bersuara sama) dan **weighted similarity** (bobot suara = skor cosine).
3. **Cell 31** — definisi fungsi `predict_outcome(query, k, method, strategy, exclude_case_id)` dengan dukungan leave-one-out.
4. **Cell 32** — demo prediksi **5 kasus uji** (leave-one-out, query dari fakta/pasal tanpa amar) dan simpan hasil ke `predictions.csv`. Tampilkan akurasi verdict & MAE lama penjara.
   > Ubah variabel `STRATEGY = "weighted"` atau `"majority"`, dan `METHOD = "tfidf"` atau `"bert"` sesuai kebutuhan.
5. **Cell 33** — ekspor script `04_predict.py` ke Google Drive.

---

### Tahap 5 — Model Evaluation
**Cell:** 34–40  
**Input:** Semua output dari Tahap 1–4 (variabel in-memory & file CSV)  
**Output:** `eval/retrieval_metrics.csv`, `eval/prediction_metrics.csv`, `eval/model_comparison.png`, script `05_evaluation.py`

1. **Cell 34** — install `matplotlib` & konfigurasi path output:
   ```python
   RETRIEVAL_CSV  = EVAL_DIR / "retrieval_metrics.csv"
   PREDICTION_CSV = EVAL_DIR / "prediction_metrics.csv"
   CHART_PNG      = EVAL_DIR / "model_comparison.png"
   ```
2. **Cell 35** — evaluasi **adil keempat model** (protokol Leave-One-Out, ruang label seragam):
   - `Retrieval-TFIDF` — representasi TF-IDF + cosine k-NN
   - `Retrieval-IndoBERT` — embedding IndoBERT + cosine k-NN
   - `SVM` (LinearSVC) — klasifikasi langsung pada TF-IDF
   - `Naive Bayes` (ComplementNB) — klasifikasi langsung pada TF-IDF
3. **Cell 36** — hitung dan tampilkan tabel metrik retrieval (Accuracy, Precision, Recall, F1, Hit@1, Hit@k, MRR) → simpan ke `retrieval_metrics.csv`.
4. **Cell 37** *(opsional)* — tampilkan bar chart perbandingan performa model → simpan ke `model_comparison.png`.
5. **Cell 38** — evaluasi prediksi Tahap 4 (leave-one-out per kombinasi method × strategy) → simpan ke `prediction_metrics.csv`. Metrik: Accuracy verdict, macro-P/R/F1, MAE lama penjara (bulan).
6. **Cell 39** — analisis kegagalan model (rejection & pasangan kelas yang paling sering tertukar) beserta rekomendasi perbaikan.
7. **Cell 40** — ekspor script `05_evaluation.py` ke Google Drive.

---

## Ringkasan Pipeline

```
PDF Putusan MA RI (Google Drive)
      ↓
[Tahap 1 — Cell 1–9]   Ekstraksi & Pembersihan Teks → raw/*.txt  ← sudah tersedia
      ↓
[Tahap 2 — Cell 10–16] Representasi Kasus → cases.csv  ← titik mulai yang disarankan
      ↓
[Tahap 3 — Cell 17–28] Retrieval (TF-IDF + IndoBERT + SVM + Naive Bayes)
      ↓
[Tahap 4 — Cell 29–33] Prediksi Solusi (Majority Vote & Weighted Similarity)
      ↓
[Tahap 5 — Cell 34–40] Evaluasi Model (Hit@k, Recall@k, MRR, Accuracy, F1)
```

---

## Catatan

- Notebook dirancang untuk **Google Colab**. Jika menjalankan secara lokal, lewati Cell 1 (Mount Google Drive) dan sesuaikan semua path ke direktori lokal.
- Proses IndoBERT embedding pada Tahap 3 (Cell 21) memerlukan waktu cukup lama; disarankan menggunakan **GPU runtime** di Colab (`Runtime → Change runtime type → T4 GPU`).
- Jika setelah Cell 17 versi Pillow masih 12.x, lakukan **Restart session** lalu jalankan ulang Cell 17 sebelum melanjutkan ke Cell 18.
- File `.txt` di `data/raw/` sudah tersedia di repository — pipeline dapat langsung dimulai dari **Tahap 2 (Cell 10)** tanpa perlu mengunduh PDF.
- File PDF asli tidak disertakan dalam repository. Unduh melalui tautan Google Drive di bagian Dataset jika ingin menjalankan Tahap 1 dari awal.
- Tahap 3 (Cell 18a) akan **dilewati otomatis** jika folder teks mentah tidak ditemukan; kolom `amar_putusan` & `dakwaan_ringkas` dari `cases.csv` tetap dipakai.

---

## Informasi Proyek

| | |
|---|---|
| **Metode** | Case-Based Reasoning (CBR) |
| **Domain** | Putusan Perkara Narkotika MA RI |
| **Model Retrieval** | TF-IDF, IndoBERT (cosine similarity), SVM, Naive Bayes |
| **Metrik Evaluasi** | Hit@k, Recall@k, MRR, Accuracy, F1-Score |
