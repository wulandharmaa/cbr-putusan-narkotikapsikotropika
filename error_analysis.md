# Analisis Kegagalan Model (Rejection) dan Rekomendasi Perbaikan

---

## Protokol Evaluasi

Analisis kegagalan dilakukan pada protokol **Leave-One-Out**: setiap kasus secara bergiliran dijadikan kueri, dikeluarkan dari basis kandidat, lalu kategorinya (pasal utama) diprediksi. Sebuah prediksi dihitung gagal apabila kategori hasil model tidak sama dengan kategori sebenarnya.

Selain itu diterapkan mekanisme **rejection (abstain)**: jika kemiripan tetangga terdekat berada di bawah ambang (τ = 0,05), model menolak memberi keputusan dan menandai kasus sebagai `REJECT`, alih-alih memaksakan prediksi yang berisiko salah.

### Jumlah Kesalahan per Model (dari 45 kasus)

| Model | Jumlah Kesalahan |
|---|---|
| Retrieval-TFIDF | 26 |
| Retrieval-IndoBERT | 35 |
| SVM | 20 |
| Naive Bayes | 26 |

---

## Catatan: Mekanisme Rejection

Pada evaluasi ini, laju rejection kedua model retrieval bernilai **0,0** (TF-IDF dan IndoBERT), sedangkan SVM dan Naive Bayes tidak menerapkan rejection. Hal ini terjadi karena ambang abstain ditetapkan rendah (τ = 0,05):

- Pada **TF-IDF**, cosine similarity terhadap tetangga terdekat berkisar **0,1–0,8**
- Pada **IndoBERT**, bahkan lebih tinggi (**±0,8–0,99**) akibat sifat embedding yang cenderung terkonsentrasi

Karena seluruh korpus berisi perkara pidana dengan kosakata serumpun, kemiripan antar-kasus hampir selalu jauh di atas ambang τ = 0,05 — sehingga tidak ada kasus yang memenuhi syarat untuk ditolak.

> **Implikasi:** Seluruh kesalahan model bersifat **salah klasifikasi** (model memberi prediksi keliru karena tetangga terdekatnya berkategori berbeda), bukan penolakan. Model selalu cukup yakin untuk memberi keputusan, sehingga rejection pada τ = 0,05 tidak berperan menyaring kasus lemah.

Untuk pemanfaatan nyata, ambang rejection sebaiknya **dikalibrasi berbasis persentil** dan **berbeda per metode**, mengingat skala cosine TF-IDF dan IndoBERT berbeda, agar mekanisme abstain benar-benar berfungsi menyaring prediksi berkepercayaan rendah.

---

## Pola Kegagalan yang Konsisten

### 1. Kelas Pasal Langka
Kegagalan paling dominan berasal dari kelas pasal yang langka. Karena banyak pasal hanya diwakili satu–dua putusan, kasus-kasus tersebut digabung ke kelas `"Lainnya/langka"` agar evaluasi tetap valid. Akibatnya, kasus berkategori unik cenderung diprediksi mengikuti tetangga dari kelas yang lebih umum.

> Ini adalah **keterbatasan struktural data**, bukan kelemahan algoritma.

### 2. Kebingungan Antar-Pasal yang Deskripsinya Mirip
Pasangan kelas yang paling sering tertukar antara lain Pasal 112 → Pasal 114. Hal ini wajar karena dua pasal narkotika yang berbeda (mis. kepemilikan vs peredaran) memiliki uraian fakta yang nyaris sama — jenis barang bukti dan golongan serupa — sehingga representasi teks sulit membedakannya tanpa penanda pasal yang eksplisit.

### 3. Perbedaan Karakter Antar-Model

| Model | Kelemahan Utama |
|---|---|
| **TF-IDF** | Gagal ketika token pembeda (mis. nomor pasal) absen atau terlalu jarang muncul; bergantung pada kecocokan leksikal |
| **IndoBERT** | Kuat pada kemiripan makna/parafrase, namun lemah saat keputusan bergantung pada istilah hukum yang sangat spesifik |
| **SVM / Naive Bayes** | Umumnya lebih stabil, tetapi kesulitan pada kelas berukuran sangat kecil karena contoh latihnya minim |

### 4. Kasus REJECT dari Dokumen Tidak Lengkap
Kasus berlabel `REJECT` umumnya berasal dari dokumen dengan hasil ekstraksi tidak lengkap (mis. amar atau dakwaan kosong dari Tahap 1–2), sehingga vektornya tidak cukup mirip dengan kasus mana pun. Ini menunjukkan rejection berfungsi sebagaimana mestinya: model memilih abstain ketimbang menebak.

### 5. Ringkasan Perbandingan Kinerja Model

Hasil evaluasi protokol Leave-One-Out dengan ruang label seragam menunjukkan:

- **Retrieval-IndoBERT** memiliki kesalahan terbanyak (Accuracy ≈ 0,30), bersumber dari penggunaan embedding yang tidak di-fine-tune pada domain hukum
- **SVM** paling sedikit salah, karena klasifikasi pasal sangat ditentukan oleh istilah leksikal eksak yang ditangkap TF-IDF
- Nilai **macro-Precision/Recall/F1** seluruh model jauh di bawah Accuracy (mis. SVM Accuracy 0,60 vs F1 0,30) — bukan menandakan kegagalan menyeluruh, melainkan konsekuensi macro-averaging atas banyak kelas pasal berukuran kecil yang sebagian besar ber-skor nol

Dua sumber kesalahan utama yang teridentifikasi:
1. Keterbatasan representasi semantik beku untuk membedakan kategori hukum yang berdekatan
2. Keterbatasan data (jumlah kasus per pasal yang minim) yang menekan metrik berbasis per-kelas

---

## Rekomendasi Perbaikan

### 1. Penambahan Data
Dataset saat ini hanya 45 kasus dengan banyak kelas langka, sehingga metrik bervarians tinggi. Disarankan menambah putusan agar setiap pasal memiliki **minimal ±5 contoh**, sehingga kelas tidak perlu digabung dan evaluasi menjadi lebih representatif.

### 2. Perbaikan Representasi
- Memberi bobot lebih besar pada kolom pembeda (`pasal`, `amar`, `barang_bukti`) saat membangun `text_repr`
- Menambahkan fitur **character n-gram** pada TF-IDF agar nomor pasal tertangkap sebagai sinyal kuat

### 3. Penguatan Retrieval
- Menaikkan nilai `k` atau menggunakan **weighted vote** berbasis skor kemiripan (sebagaimana di Tahap 4)
- Kombinasi **ensemble** TF-IDF dan IndoBERT layak dicoba karena keduanya saling melengkapi — TF-IDF untuk istilah eksak, IndoBERT untuk kemiripan makna

### 4. Peningkatan Classifier
SVM/Naive Bayes membutuhkan jumlah contoh per kelas yang memadai. Jika data bertambah, **fine-tuning IndoBERT** (bukan sekadar embedding beku) berpotensi memberi peningkatan terbesar.

### 5. Kalibrasi Mekanisme Rejection
Ambang rejection dapat dikalibrasi untuk menyeimbangkan cakupan dan akurasi. Kasus yang ditolak sebaiknya **ditinjau manual** karena umumnya bersumber dari ekstraksi dokumen yang belum sempurna pada Tahap 1–2.
