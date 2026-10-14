# ICA · Label, Feature Engineering & Feature Selection

Dokumen ini menjelaskan definisi label, fitur yang dibuat sendiri (feature engineering), dan cara memilih fitur untuk tiga model ML di ICA.
Semua kode ada di notebook `ICA_End_to_End.ipynb`, **section 6** (feature engineering) dan **section 7** (set fitur, split, seleksi).

---

## 1. Definisi label (target)

| Model | Label | Bernilai 1 (positif) jika | Unit (1 baris) |
|---|---|---|---|
| Credit Risk · New (`cr_new`) | `default_12m` | Dalam **12 bulan setelah kredit cair**, nasabah pernah menunggak **≥ 90 hari** (DPD ≥ 90) | 1 aplikasi pemohon baru |
| Credit Risk · Existing (`cr_existing`) | `default_12m` | Definisi **sama persis** dengan New | 1 aplikasi nasabah lama |
| Early Warning (`ew`) | `deterioration_next_1m` | **Bucket DPD bulan depan lebih buruk** daripada bulan ini | 1 nasabah × 1 bulan |

### 1.1 Credit Risk (New & Existing)

- Label New dan Existing sama. Yang berbeda hanya **sumber fitur**:
  - **New** belum punya riwayat di bank, sehingga fitur diambil dari profil, form aplikasi, dan 6 label interview.
  - **Existing** punya riwayat, sehingga fitur diambil dari perilaku bayar 6 snapshot bulanan sebelum tanggal aplikasi.
- Batas 90 hari mengikuti batas kredit bermasalah (NPL). Di POJK 40/2019, tunggakan lebih dari 90 hari masuk Kolektibilitas 3 (Kurang Lancar) ke atas.
- Guardrail di section 2 memastikan `default_12m = 1` selalu sama dengan akun berstatus `Default`.
- Rate positif: `cr_new` 11,8% · `cr_existing` 8,7%.

### 1.2 Early Warning

Bucket DPD: **0 / 1–29 / 30–59 / 60–89 / 90+**.

| Bulan ini | Bulan depan | Label |
|---|---|---|
| 0 (lancar) | 1–29 | **1** (memburuk) |
| 30–59 | 30–59 | 0 (tetap) |
| 30–59 | 1–29 | 0 (membaik) |
| Bulan terakhir data (tidak ada bulan depan) | – | dibuang dari training |
| Sudah 90+ (default) | – | dibuang dari training |

- EW tidak memprediksi default, tetapi **tanda awal memburuk sebulan ke depan**, supaya bank bisa bertindak sebelum nasabah jatuh ke NPL.
- Rate positif: 9,3%.
- Keterbatasan: kenaikan sekecil 0 → 1–29 sudah dihitung memburuk, sehingga alert ringan cukup banyak. Ke depan bisa dipisah per tingkat keparahan.
- `default_next_3m` ada di data, tetapi **tidak dipakai sebagai fitur**. Kolom ini hanya dipakai di dashboard dampak bisnis.

---

## 2. Aturan anti-leakage

| Aturan | Implementasi |
|---|---|
| Point-in-time Existing | Fitur perilaku hanya dari snapshot `snapshot_date < application_date`. Guardrail mengecek jeda median snapshot terakhir 16 hari sebelum aplikasi |
| Point-in-time EW | Fitur hanya memakai data sampai bulan t (lag, delta, rolling) |
| Outcome tidak dipakai | `account_status`, `days_past_due`, `outstanding` akhir, `disbursement_date` dilarang |
| Split sebelum seleksi | Feature selection hanya melihat data train |
| Split EW per nasabah | `GroupShuffleSplit`: nasabah yang sama tidak ada di train dan test sekaligus |
| Fairness | `gender`, `city`, `marital_status` tidak pernah jadi fitur, hanya untuk audit |

---

## 3. Feature engineering

Total ada **59 fitur turunan** yang dibuat dari kolom mentah, dibagi 4 kelompok.

### 3.1 Fitur aplikasi: 13 fitur (section 6.1, dipakai New & Existing)

Fungsi: `add_application_features()`.

| Fitur | Rumus | Alasan |
|---|---|---|
| `dsr` | (cicilan baru + cicilan lain) / penghasilan | Rasio utama kemampuan bayar, dipakai semua bank |
| `installment_to_income` | cicilan baru / penghasilan | DSR dipecah: beban dari pinjaman yang diajukan |
| `existing_debt_to_income` | cicilan lain / penghasilan | DSR dipecah: beban hutang lama |
| `residual_income` | penghasilan − semua cicilan | Sisa uang untuk hidup, dalam rupiah |
| `residual_income_per_capita` | residual / (1 + tanggungan) | Sisa uang 10 juta untuk sendiri berbeda dengan untuk 5 orang |
| `lti` | plafon / (penghasilan × 12) | Besar pinjaman relatif terhadap penghasilan setahun |
| `log_income` | log(penghasilan) | Penghasilan sangat miring; log membuat LogReg lebih stabil |
| `log_amount` | log(plafon) | Sama, untuk plafon |
| `tenor_years` | tenor / 12 | Skala lebih mudah dibaca |
| `short_tenure` | masa kerja < 12 bulan | Flag: baru bekerja, pekerjaan belum stabil |
| `young_borrower` | umur < 25 | Flag: peminjam muda, riwayat kredit tipis |
| `has_business_income` | sumber penghasilan bukan gaji | Flag: penghasilan usaha lebih fluktuatif |
| `secured_loan` | KPR atau kredit kendaraan | Flag: ada agunan |

### 3.2 Fitur interview: 5 fitur (section 6.2, khusus New)

Fungsi: `add_interview_features()`. Saat produksi, 6 label interview diisi oleh GenAI dari jawaban analis.

| Fitur | Isi | Alasan |
|---|---|---|
| `income_stability_ord` | high = 0, medium = 1, low = 2 | Label bertingkat dijadikan angka berurutan, sehingga model tahu "low" lebih berisiko dari "medium" |
| `existing_debt_level_ord` | none = 0, low = 1, medium = 2, high = 3 | Sama |
| `repayment_issue_ord` | none = 0, occasional = 1, frequent = 2 | Sama |
| `interview_neg_signals` | Jumlah jawaban yang mengarah ke risiko (0–6) | Satu skor ringkas dari seluruh interview. Termasuk fitur terkuat `cr_new` (IV > 0,5) |
| `debt_underreported` | Hutang yang diakui di interview < hutang di data aplikasi/SLIK | Sinyal kejujuran pemohon |

Komponen `interview_neg_signals` (masing-masing bernilai 1):
- `income_trend` = declining atau volatile
- `income_stability` = low
- `existing_debt_level` = high
- `repayment_issue` ≠ none
- `financial_change` ≠ stable
- `business_condition` = declining

Catatan:
- Tiga label nominal (`income_trend`, `financial_change`, `business_condition`) hanya di-one-hot, bukan fitur baru.
- `debt_underreported` dibuang dari model karena IV ≈ 0, tetapi tetap dipakai sebagai **catatan konsistensi** di laporan analis.

### 3.3 Fitur perilaku: 19 fitur (section 6.3, khusus Existing)

Fungsi: `behaviour_features()`. Diambil dari **6 snapshot bulanan terakhir sebelum tanggal aplikasi**.
Baseline hanya memakai 1 snapshot terakhir apa adanya (`snap_*`). FE menambah tren dan agregat 6 bulan.

| Kelompok | Fitur | Arti |
|---|---|---|
| Utilisasi limit | `beh_util_last`, `beh_util_mean6`, `beh_util_trend6` | Nilai terakhir, rata-rata, dan kemiringan (slope regresi) 6 bulan. Utilisasi yang terus naik menandakan butuh dana |
| Payment ratio | `beh_pr_last`, `beh_pr_mean6`, `beh_pr_min6` | Porsi tagihan yang dibayar. Titik terendah sering lebih informatif daripada rata-rata |
| Keterlambatan | `beh_dpd_last`, `beh_dpd_max6`, `beh_months_late6`, `beh_late_6m`, `beh_miss_6m` | Telat terakhir, telat terparah, jumlah bulan pernah telat, hitungan telat dan tidak bayar |
| Credit hungry | `beh_newacc_6m` | Jumlah kredit baru di lembaga lain dalam 6 bulan |
| Beban | `beh_outstanding_growth6`, `beh_overdue_to_due`, `beh_due_last` | Pertumbuhan sisa hutang, porsi tunggakan dari tagihan, tagihan terakhir |
| Akun saat aplikasi | `acc_active_at_app`, `acc_cards_at_app`, `acc_limit_at_app` | Hanya akun yang sudah dibuka dan belum ditutup per tanggal aplikasi |
| Lama jadi nasabah | `months_on_book` | (tanggal aplikasi − nasabah sejak) dalam bulan |

### 3.4 Fitur Early Warning: 22 fitur (section 6.4, per nasabah-bulan)

Fungsi: `ew_features()`. Dihitung per nasabah, berurutan waktu, hanya memakai data sampai bulan t.
Inti EW adalah **perubahan**, bukan level: nasabah yang biasa di utilisasi 80% berbeda dengan yang naik dari 30% ke 80%.

| Kelompok | Fitur | Arti |
|---|---|---|
| Perubahan utilisasi | `util_delta_1m`, `util_delta_3m`, `util_vs_mean6` | Kenaikan dibanding 1 bulan lalu, 3 bulan lalu, dan rata-rata 6 bulannya sendiri |
| Payment ratio | `pr_mean_3m`, `pr_min_3m`, `pr_delta_3m` | Rata-rata, titik terendah, perubahan 3 bulan. `pr_min_3m` fitur terkuat EW (IV > 0,5) |
| Streak | `pr_decline_streak` | Berapa bulan berturut-turut payment ratio turun |
| Keterlambatan | `dpd_lag1`, `dpd_delta_1m`, `late_count_delta`, `miss_count_delta` | DPD bulan lalu dan perubahannya |
| Status | `late_now` (DPD 1–29), `in_arrears` (DPD ≥ 30) | Flag posisi sekarang |
| Recency | `months_since_late` | Bulan sejak terakhir telat (24 = belum pernah) |
| Beban | `overdue_to_due`, `payment_gap_to_income`, `due_to_income`, `outstanding_change_3m` | Porsi tunggakan, kekurangan bayar dibanding penghasilan, tagihan dibanding penghasilan, perubahan sisa hutang 3 bulan |
| Credit hungry | `newacc_3m`, `newacc_6m` | Kredit baru di 3 dan 6 bulan terakhir (rolling sum) |
| Profil | `log_income`, `months_on_book` | Penghasilan (log) dan lama jadi nasabah |

Selain itu ada 4 kolom profil statis yang hanya digabung dari tabel nasabah (umur, masa kerja, jenis pekerjaan, sumber penghasilan), sehingga bukan fitur buatan.

---

## 4. Baseline vs Feature Engineering

| Task | Baseline (fitur mentah) | FE (sebelum seleksi) | Fitur buatan | Lolos seleksi |
|---|---|---|---|---|
| `cr_new` | Profil + aplikasi + 6 label interview apa adanya | 36 kolom | 13 aplikasi + 5 interview | 24 |
| `cr_existing` | Profil + aplikasi + 1 snapshot terakhir | 58 kolom | 13 aplikasi + 19 perilaku | 34 |
| `ew` | 16 kolom bulanan apa adanya | 42 kolom | 22 | 25 |

Pesan utama: **baseline melihat kondisi di satu titik, FE menambah rasio kemampuan bayar, tren, dan konsistensi.** Hasilnya PR-AUC naik di 9/9 kombinasi (3 algoritma × 3 task).

Baseline sengaja **tidak diseleksi**, supaya kenaikan performa benar-benar bisa diatribusikan ke feature engineering.

---

## 5. Feature selection

Fungsi: `select_features()` (section 7.3). Hanya pada set FE dan hanya memakai data train. Parameter: `CONST_MAX = 0.99`, `IV_MIN = 0.02`, `CORR_MAX = 0.85`.

### 5.1 Langkah 1: Near-constant (≥ 99% nilai sama → buang)

- Fitur yang hampir selalu bernilai sama tidak bisa membedakan siapa pun, dan IV-nya tidak stabil karena hanya ditopang sedikit baris.
- Di data saat ini langkah ini kemungkinan tidak membuang apa-apa. Fungsinya **jaring pengaman**: di data baru atau produksi, sebuah flag bisa tidak pernah menyala dan langsung tertangkap. Biayanya nol.

### 5.2 Langkah 2: Information Value (IV < 0,02 → buang)

IV mengukur seberapa berbeda distribusi fitur antara nasabah default dan tidak default. Fitur numerik dipotong menjadi 10 kelompok (desil), lalu tiap kelompok dibandingkan porsi "bad" dan "good":

```
IV = Σ (%bad − %good) × ln(%bad / %good)
```

| IV | Kekuatan |
|---|---|
| < 0,02 | Tidak berguna (dibuang) |
| 0,02–0,1 | Lemah |
| 0,1–0,3 | Sedang |
| 0,3–0,5 | Kuat |
| > 0,5 | Dicurigai leakage, dicek manual |

Alasan memakai IV:
- **Standar industri credit scoring.** Regulator dan tim risiko bank sudah familiar dengan ambangnya.
- **Satu ukuran untuk numerik dan kategorikal.** `loan_type` dan `dsr` bisa dibandingkan dengan skala yang sama.
- **Menangkap pola tidak linear.** Risiko yang hanya melonjak di desil teratas tetap terdeteksi, karena tiap kelompok dinilai sendiri.
- **Detektor leakage.** Fitur dengan IV > 0,5 sudah dicek manual dan **bukan leakage**, karena semuanya diketahui saat prediksi:
  - `cr_new`: `dsr`, `interview_neg_signals`, `residual_income`, `income_stability_ord`, `residual_income_per_capita`
  - `cr_existing`: `dsr`, `residual_income`, `residual_income_per_capita`
  - `ew`: `pr_min_3m`

Kelemahan: IV bersifat **univariat**, sehingga fitur yang lemah sendirian tetapi kuat bila digabung bisa ikut terbuang. Ambang 0,02 sengaja rendah supaya yang dibuang hanya yang benar-benar nol informasinya, misalnya `loan_type` dan `application_channel` di `cr_new`.

### 5.3 Langkah 3: Korelasi Spearman (|ρ| > 0,85 → buang yang IV-nya lebih kecil)

Tujuannya membuang fitur kembar yang isinya hampir sama. Contoh:
- `log_amount` vs `requested_amount` (ρ = 1,00)
- `monthly_income` vs `residual_income` (ρ = 0,90)

Kenapa perlu:
- **Logistic Regression** (model terpilih untuk kedua task CR) menjadi tidak stabil bila ada fitur kembar. Koefisiennya bisa saling menutupi, bahkan tandanya terbalik.
- **Penjelasan ke analis jadi bersih.** Narasi LLM mengambil faktor pendorong teratas. Bila ada fitur kembar, faktor yang sama muncul dua kali dengan nama berbeda.

Kenapa Spearman, bukan Pearson:
- Berbasis urutan (ranking), sehingga tahan outlier dan data miring seperti penghasilan dan plafon.
- Menangkap hubungan monoton yang tidak lurus. Contoh: `log_income` vs `monthly_income` berkorelasi Spearman 1, sedangkan Pearson tidak.

Kenapa yang dipertahankan yang IV-nya lebih besar: dari dua fitur kembar, simpan yang paling memisahkan default dan tidak default.

### 5.4 Kenapa metode filter, bukan RFE / SHAP / L1

- **Satu set fitur untuk 3 algoritma.** LogReg, Random Forest, dan XGBoost dibandingkan dengan fitur yang sama, sehingga perbandingannya adil. Seleksi berbasis model akan memihak model tertentu.
- **Mudah dijelaskan dan diaudit.** Tiap fitur punya alasan dibuang yang jelas ("IV < 0,02", "korelasi 0,90 dengan residual_income"). Ini penting untuk tata kelola model di bank.
- **SHAP dipakai untuk menjelaskan model, bukan untuk seleksi.** Bila dipakai untuk seleksi, SHAP harus dihitung di dalam CV supaya tidak bocor, dan itu menambah kompleksitas tanpa jaminan recall naik.
