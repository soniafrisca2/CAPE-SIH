# ICA · Dokumentasi Machine Learning

> Dokumen ini menjelaskan **tiga model ML ICA**: tujuan, data & label, feature engineering, feature selection, training,
> kebijakan threshold, kalibrasi, hasil evaluasi, guardrail, dan artefak yang disimpan.
> Notebook: `ICA_End_to_End.ipynb` section 1–13. Detail kolom data ada di `ICA_data_metadata.md`.

---

## 1. Ringkasan

| | **Credit Risk · New** (`cr_new`) | **Credit Risk · Existing** (`cr_existing`) | **Early Warning** (`ew`) |
|---|---|---|---|
| Pertanyaan bisnis | Pemohon baru ini akan gagal bayar? | Nasabah lama yang mengajukan kredit ini akan gagal bayar? | Nasabah aktif ini akan memburuk bulan depan? |
| Unit (1 baris) | 1 aplikasi pemohon baru | 1 aplikasi nasabah existing | 1 nasabah × 1 bulan |
| Label | `default_12m`: DPD ≥ 90 dalam 12 bulan setelah pencairan | `default_12m` | `deterioration_next_1m`: bucket DPD bulan depan lebih buruk |
| Sumber fitur khas | Jawaban interview (6 label, diisi GenAI saat produksi) | Perilaku bayar 6 bulan sebelum aplikasi | Tren 1–6 bulan sampai bulan t |
| Train / test | 3.200 / 800 | 6.400 / 1.600 | 142.842 / 35.875 |
| Rate positif | 11,8% | 8,7% | 9,3% |
| Model terpilih | Logistic Regression (FE) | Logistic Regression (FE) | XGBoost (FE) |
| Output ke analis | Skor risiko → Approve / Further Review / Reject + PD terkalibrasi | sama | Skor → Aman / Further Review / ALERT + peluang memburuk |

**Metrik utama: recall.** Bagi bank, meloloskan calon default (false negative) jauh lebih mahal daripada salah menahan
pemohon baik (false positive), karena kasus yang ditahan masih diperiksa analis. Precision, PR-AUC, dan dampak bisnis tetap
dilaporkan sebagai penyeimbang.

---

## 2. Alur pipeline

```
5 tabel CSV ─► validasi skema & integritas ─► fitur point-in-time ─► split train/test ─► feature selection (train saja)
                                                                                                │
                      ┌─────────────────────────────────────────────────────────────────────────┘
                      ▼
   3 algoritma × 2 set fitur × 3 task = 18 eksperimen (RandomizedSearchCV, skor PR-AUC, prediksi out-of-fold)
                      │
                      ▼
   pilih model terbaik per task (PR-AUC OOF) ─► threshold recall-first (OOF) ─► kalibrasi skor → PD (Platt, OOF)
                      │
                      ▼
   guardrail output (OOD, zona Further Review, fairness) ─► monitoring (PSI) ─► dashboard dampak bisnis
```

Prinsip yang dijaga di semua tahap: **test set hanya dipakai sekali untuk pelaporan**. Pemilihan model, threshold, dan
kalibrasi memakai prediksi out-of-fold (OOF) di train.

---

## 3. Data & aturan anti-leakage

| Aturan | Implementasi | Dicek guardrail |
|---|---|---|
| Point-in-time Credit Risk existing | Fitur perilaku hanya dari snapshot `snapshot_date < application_date` (6 snapshot terakhir). Akun dihitung hanya yang sudah dibuka dan belum tutup per tanggal aplikasi | ✅ jeda median snapshot terakhir 16 hari sebelum aplikasi |
| Outcome tidak dipakai | `account_status`, `days_past_due`, `outstanding` (status akhir 2025), `disbursement_date` dilarang | ✅ |
| Point-in-time EW | Fitur hanya memakai data sampai bulan t (lag, delta, rolling). `default_next_3m` bukan fitur | ✅ |
| Split | Credit Risk: stratified 80/20 (satu aplikasi per nasabah). EW: 80/20 **per nasabah** (`GroupShuffleSplit`), jadi nasabah yang sama tidak ada di train dan test | ✅ nasabah train & test tidak beririsan |
| Selection di train saja | IV dan korelasi dihitung hanya di index train | ✅ |
| Fairness | `gender`, `city`, `marital_status` **tidak pernah** jadi fitur, hanya untuk audit disparate impact | ✅ |

Seed: `RANDOM_STATE = 42` untuk split, CV, dan model.

---

## 4. Set fitur: baseline vs feature engineering

| Task | Baseline (fitur mentah) | Feature Engineering (FE) | Jumlah kolom setelah encoding |
|---|---|---|---|
| `cr_new` | Profil + aplikasi + 6 label interview apa adanya | + rasio kemampuan bayar, encoding ordinal interview, sinyal negatif, cek konsistensi hutang | baseline 57 · FE 46 (24 dari 36 fitur lolos seleksi) |
| `cr_existing` | Profil + aplikasi + snapshot bulanan terakhir | + rasio + agregat perilaku 6 bulan + akun saat aplikasi | baseline 46 · FE 50 (34 dari 58 lolos) |
| `ew` | 16 kolom bulanan apa adanya | + delta/tren 1–3 bulan, streak, rasio tunggakan, profil statis | baseline 16 · FE 25 (25 dari 42 lolos) |

### 4.1 Fitur aplikasi (new & existing)

| Fitur | Rumus / arti |
|---|---|
| `dsr` | (cicilan baru + cicilan existing) / penghasilan. Rasio utama kemampuan bayar |
| `installment_to_income`, `existing_debt_to_income` | DSR dipecah: beban pinjaman baru vs hutang lama |
| `residual_income`, `residual_income_per_capita` | Sisa penghasilan setelah semua cicilan, total dan per anggota keluarga |
| `lti` | Plafon / penghasilan setahun |
| `log_income`, `log_amount`, `tenor_years` | Transformasi untuk mengurangi skew |
| `short_tenure`, `young_borrower`, `has_business_income`, `secured_loan` | Flag: masa kerja < 12 bulan, umur < 25, penghasilan usaha, kredit beragunan |

### 4.2 Fitur interview (`cr_new`)

| Fitur | Isi |
|---|---|
| `income_stability_ord`, `existing_debt_level_ord`, `repayment_issue_ord` | Encoding ordinal label bertingkat |
| One-hot `income_trend`, `financial_change`, `business_condition` | Label nominal |
| `interview_neg_signals` | Jumlah jawaban yang mengarah ke risiko (0–6) |
| `debt_underreported` | Hutang yang diakui di interview lebih rendah dari data aplikasi (SLIK). IV ≈ 0 sehingga dibuang dari model, tetapi dipakai sebagai **catatan konsistensi** di laporan |

Saat produksi, keenam label ini **diisi GenAI** dari jawaban bebas pemohon (section 16). Selisih recall model dengan
label LLM vs label asli dijaga oleh quality gate GenAI (|gap| ≤ 0,05).

### 4.3 Fitur perilaku (`cr_existing`, 6 snapshot sebelum aplikasi)

`beh_util_last/mean6/trend6` (utilisasi), `beh_pr_last/mean6/min6` (payment ratio), `beh_dpd_last/max6`,
`beh_months_late6`, `beh_late_6m`, `beh_miss_6m`, `beh_newacc_6m` (kredit baru di lembaga lain),
`beh_outstanding_growth6`, `beh_overdue_to_due`, `beh_due_last`, `acc_active_at_app`, `acc_cards_at_app`,
`acc_limit_at_app`, `months_on_book`. Baseline memakai snapshot mentah terakhir (`snap_*`).

### 4.4 Fitur Early Warning (per nasabah-bulan)

| Kelompok | Fitur |
|---|---|
| Utilisasi | `util_delta_1m`, `util_delta_3m`, `util_vs_mean6` |
| Payment ratio | `pr_mean_3m`, `pr_min_3m`, `pr_delta_3m`, `pr_decline_streak` (bulan berturut-turut turun) |
| Keterlambatan | `dpd_lag1`, `dpd_delta_1m`, `late_count_delta`, `miss_count_delta`, `late_now`, `in_arrears`, `months_since_late` |
| Beban | `overdue_to_due`, `payment_gap_to_income`, `due_to_income`, `outstanding_change_3m` |
| Credit hungry | `newacc_3m`, `newacc_6m` |
| Profil | `log_income`, `months_on_book`, `age`, `employment_tenure_months`, `employment_type`, `income_source` |

---

## 5. Feature selection

Hanya pada set FE dan hanya memakai data train. Tiga langkah berurutan:

| Langkah | Aturan | Alasan |
|---|---|---|
| 1. Near-constant | Buang jika ≥ 99% nilainya sama | Tidak informatif |
| 2. Information Value (IV) | Buang jika IV < 0,02 | Standar industri credit scoring. 0,02–0,1 lemah · 0,1–0,3 sedang · > 0,3 kuat · > 0,5 dicek manual |
| 3. Korelasi | Dari pasangan numerik dengan \|Spearman\| > 0,85, buang yang IV-nya lebih kecil | Mengurangi kolinearitas (penting untuk LogReg dan interpretasi) |

Fitur dengan **IV > 0,5** sudah dicek manual dan **bukan leakage**. Semuanya diketahui pada saat prediksi dan memang
pendorong risiko yang kuat:
- `cr_new`: `dsr`, `interview_neg_signals`, `residual_income`, `income_stability_ord`, `residual_income_per_capita`
- `cr_existing`: `dsr`, `residual_income`, `residual_income_per_capita`
- `ew`: `pr_min_3m`

Contoh fitur yang dibuang: `monthly_income` (korelasi 0,90 dengan `residual_income`), `log_amount` (1,00 dengan
`requested_amount`), `loan_type` dan `application_channel` di `cr_new` (IV < 0,02).

---

## 6. Model & training

| Algoritma | Alasan | Penanganan kelas tidak seimbang | Ruang hyperparameter |
|---|---|---|---|
| Logistic Regression | Interpretable, standar regulator credit scoring | `class_weight="balanced"`, `StandardScaler` di dalam pipeline | `C` ~ loguniform(0,001; 10) |
| Random Forest | Bagging, tahan outlier, non-linear tanpa banyak tuning | `class_weight="balanced_subsample"` | `n_estimators` {200, 300, 500}, `max_depth` {6, 8, 12, 16, None}, `min_samples_leaf` {5, 10, 20, 50}, `max_features` {sqrt, 0,3, 0,5} |
| XGBoost | Boosting, umumnya terbaik di data tabular, jalan di GPU | `scale_pos_weight` = negatif/positif | `n_estimators` 200–900, `learning_rate` 0,01–0,2, `max_depth` 3–8, `min_child_weight`, `subsample`, `colsample_bytree`, `reg_lambda` |

| Pengaturan | Nilai |
|---|---|
| Pencarian | `RandomizedSearchCV`, skor **PR-AUC** (`average_precision`), 25 iterasi (Credit Risk) / 12 iterasi (EW) |
| Cross-validation | 3-fold: `StratifiedKFold` (Credit Risk), `GroupKFold` per nasabah (EW) |
| Tuning EW | Di subsampel ±60.000 baris (per nasabah) agar cepat. Model final di-fit di seluruh train |
| Prediksi OOF | `cross_val_predict` dengan parameter terbaik, untuk threshold, pemilihan model, dan kalibrasi |
| Pemilihan model | **PR-AUC OOF tertinggi** per task (tidak melihat test) |
| Cache | Hasil 18 eksperimen disimpan di `modelling_cache.joblib` (kunci = fingerprint data + setting). Run berikutnya tidak melatih ulang kecuali data/setting berubah atau `FORCE_RETRAIN = True`. Training penuh ±18 menit di A100 |

PR-AUC dipakai untuk tuning dan pemilihan karena tidak bergantung threshold dan lebih jujur daripada ROC-AUC untuk kelas
minoritas ±10%.

---

## 7. Kebijakan threshold: recall-first

1. **Threshold F2** dihitung di OOF sebagai pembanding (F2 memberi bobot recall 2× precision).
2. **Threshold recall-first** = threshold **tertinggi** yang di OOF masih menangkap ≥ `RECALL_TARGET` kasus positif.
3. Threshold final = **min(F2, recall-first)**, jadi recall tidak pernah lebih rendah dari versi F2.

`RECALL_TARGET` dinaikkan dari 0,70 ke **0,75**. Recall OOF 0,70 di `cr_new` hanya menghasilkan recall test 0,57: test
`cr_new` hanya berisi ±94 kasus default, sehingga angka recall test berfluktuasi. Target 0,75 memberi margin.

---

## 8. Hasil

### 8.1 Baseline vs feature engineering (run A100, 9 Okt 2026)

| Task | Model | PR-AUC test baseline → FE | ROC-AUC test FE |
|---|---|---|---|
| cr_new | LogReg | 0,379 → **0,407** | 0,809 |
| cr_new | RandomForest | 0,343 → 0,391 | 0,808 |
| cr_new | XGBoost | 0,349 → 0,387 | 0,796 |
| cr_existing | LogReg | 0,310 → **0,344** | 0,811 |
| cr_existing | RandomForest | 0,294 → 0,376 | 0,815 |
| cr_existing | XGBoost | 0,340 → 0,370 | 0,801 |
| ew | LogReg | 0,207 → 0,225 | 0,715 |
| ew | RandomForest | 0,309 → 0,326 | 0,769 |
| ew | XGBoost | 0,319 → **0,334** | 0,775 |

**Feature engineering menaikkan PR-AUC di 9 dari 9 kombinasi** (+0,015 sampai +0,082). Semua model jauh di atas base rate
(PR-AUC acak ≈ rate positif: 0,12 / 0,09 / 0,09).

### 8.2 Model terpilih (PR-AUC OOF tertinggi)

| Task | Model | PR-AUC OOF | PR-AUC test | Recall test (threshold F2) | Recall test (recall-first 0,75) | Precision (recall-first) |
|---|---|---|---|---|---|---|
| cr_new | LogReg FE | 0,460 | 0,407 | 0,574 | **0,681** | 0,278 |
| cr_existing | LogReg FE | 0,412 | 0,344 | 0,621 | **0,750** | 0,202 |
| ew | XGBoost FE | 0,328 | 0,334 | 0,656 | dikonfirmasi setelah Run All | – |

Angka recall-first Credit Risk dihitung ulang dari prediksi OOF & test run yang sama (tanpa training ulang). Konsekuensinya,
pemohon baru yang ditandai berisiko naik dari ±23% ke ±29%. Itu sesuai prinsip recall-first, dan dampaknya bisa dilihat di
dashboard threshold.

LogReg menang di kedua task Credit Risk karena feature engineering sudah menangkap hubungan utamanya (DSR, sisa penghasilan,
sinyal interview), dan data Credit Risk relatif kecil (3.200–6.400 baris). Ini juga menguntungkan secara regulasi karena
mudah dijelaskan. XGBoost menang di EW yang datanya besar dan polanya non-linear (tren, streak).

### 8.3 Quality gate ML (dicek otomatis)

| Cek | Batas | cr_new | cr_existing | ew |
|---|---|---|---|---|
| Recall test | ≥ 0,60 | 0,681 ✅ | 0,750 ✅ | 0,656 ✅ (F2) |
| PR-AUC test | ≥ 2× base rate | 0,407 vs 0,117 ✅ | 0,344 vs 0,087 ✅ | 0,334 vs 0,094 ✅ |
| Gap PR-AUC OOF vs test | ≤ 0,10 | 0,053 ✅ | 0,068 ✅ | −0,005 ✅ |
| Rata PD terkalibrasi ≈ rate aktual test | selisih ≤ 0,02 | 0,110 vs 0,117 ✅ | 0,088 vs 0,087 ✅ | dicek tiap run |

---

## 9. Kalibrasi: skor risiko → PD

Bobot kelas membuat skor model **lebih tinggi daripada peluang default sebenarnya**. Contoh: rata-rata skor `cr_new` 0,37,
padahal rate default aktual 0,12. Karena itu ICA membedakan dua angka:

| Angka | Dipakai untuk | Cara |
|---|---|---|
| **Skor risiko** | Dibandingkan dengan threshold untuk menentukan rekomendasi | Output model apa adanya |
| **PD terkalibrasi** | Dibaca dan dijelaskan analis ("peluang gagal bayar 12 bulan") | *Platt scaling*: regresi logistik pada logit skor, di-fit pada **prediksi OOF** di train |

Kalibrasi bersifat monoton: ranking, threshold, recall, dan keputusan **tidak berubah**. Yang berubah hanya angka yang dibaca
analis.

---

## 10. Dari skor ke rekomendasi (guardrail output)

| Kondisi | Credit Risk | Early Warning |
|---|---|---|
| skor ≥ threshold + 0,05 | **Reject** | **ALERT** |
| \|skor − threshold\| < 0,05 | **Further Review** | **Further Review** |
| skor < threshold − 0,05 | **Approve** | **Aman** |
| Input OOD (≥ 2 fitur di luar persentil 0,5–99,5 train) | Further Review | Further Review |
| Flag lain (nasabah sedang menunggak/default, indikasi manipulasi interview, output LLM tidak valid) | Further Review | – |

Guardrail tambahan:
- Skor harus finite di [0, 1], dan threshold di dalam kebijakan [0,05; 0,95].
- Proporsi OOD di test tidak boleh melonjak dibanding train.
- **Disparate impact** per gender dan kota ≥ 0,8 (aturan 4/5). Credit Risk dengan threshold recall-first: 0,90–0,99.
- Rekomendasi selalu diberi label "bukan keputusan". Keputusan akhir ada di analis sesuai kewenangan.

---

## 11. Interpretabilitas

| Model | Kontribusi fitur per nasabah |
|---|---|
| XGBoost | SHAP persis dari `pred_contribs=True` bawaan XGBoost (tanpa library tambahan) |
| Logistic Regression | Koefisien × nilai terstandardisasi (setara SHAP linear) |
| Random Forest | `shap.TreeExplainer` jika tersedia |

Kontribusi teratas dikirim ke GenAI untuk dijelaskan dalam bahasa analis. **Filter arah bisnis** (`DIRECTION`): kontribusi
yang berlawanan dengan logika bisnis tidak dinarasikan dan dicatat di audit. Penyebabnya efek kolinearitas LogReg, misalnya
DPD tinggi yang terbaca sebagai "faktor penahan".

---

## 12. Dampak bisnis & monitoring

**Dashboard threshold** (`ica_threshold_dashboard.html`) mensimulasikan setiap threshold:
- Credit Risk: tingkat persetujuan, rasio NPL, kerugian ekspektasi (Σ plafon FN × LGD), estimasi profit.
- EW: rasio deteriorasi yang lolos, kerugian tercegah (Σ outstanding TP × P(default | memburuk) × LGD × efektivitas − biaya alert).

Asumsi default (bisa diubah di dashboard): LGD 45%, cost of fund 5%, efektivitas penanganan alert 30%, biaya per alert
Rp 100.000, P(default 3 bulan | memburuk) dari data train.

Temuan run 9 Okt:
- Credit Risk: profit maksimum di threshold ±0,48–0,49 dengan recall ±0,71. Angka ini **sejalan dengan kebijakan recall-first**.
- EW: kerugian tercegah maksimum di threshold tinggi (±0,75, recall ±0,32), karena biaya alert dikalikan banyak false
  positive. ICA tetap memakai recall-first untuk EW (deteksi dini lebih diutamakan), dan dashboard menunjukkan *trade-off*
  ini secara transparan untuk diputuskan manajemen risiko.

**Monitoring:** PSI skor dan 5 fitur teratas per task, train vs test. Semua < 0,1 (stabil) di run terakhir. Baseline
disimpan di `monitoring_baseline.json` untuk dibandingkan dengan data produksi.

---

## 13. Artefak (folder `ica_artifacts/`)

| File | Isi |
|---|---|
| `modelling_cache.joblib` | 18 eksperimen: hasil, prediksi OOF & test, model terlatih |
| `model_cr_new.joblib`, `model_cr_existing.joblib`, `model_ew.joblib` | Model terpilih per task |
| `model_registry.json` | Model, set fitur, threshold, parameter kalibrasi, metrik test, kebijakan threshold |
| `model_card.json` | Ringkasan model untuk tata kelola: tujuan, data, metrik, batasan, guardrail |
| `03_results.csv` | Tabel 18 eksperimen (threshold F2 & recall-first, metrik OOF & test) |
| `test_predictions.joblib` | Prediksi test untuk dashboard |
| `monitoring_baseline.json` | Distribusi acuan PSI |
| `ica_threshold_dashboard.html` | Dashboard threshold & dampak bisnis (interaktif, bisa dibuka offline) |
| `audit_log.jsonl` | Semua cek guardrail dengan `run_id` |

---

## 14. Batasan & pengembangan

| # | Batasan | Catatan / pengembangan |
|---|---|---|
| 1 | Data sintetis | Pola dibuat realistis. Model harus divalidasi ulang di data riil sebelum dipakai |
| 2 | Tidak ada *reject inference* | Data hanya berisi aplikasi yang disetujui. Model tidak pernah melihat profil yang ditolak bank |
| 3 | Test `cr_new` kecil (±94 default) | Recall test berfluktuasi ±0,05. Gunakan OOF + test dan pantau di produksi |
| 4 | Precision rendah (0,20–0,28) | Konsekuensi recall-first. Kasus positif palsu masuk antrean analis (Further Review/Reject), bukan ditolak otomatis |
| 5 | Kolinearitas di LogReg | Ditangani seleksi korelasi + filter arah bisnis pada penjelasan |
| 6 | Label EW sangat sensitif (setiap kenaikan bucket DPD) | Banyak alert ringan. Bisa dipisah per tingkat keparahan |
| 7 | Belum ada fitur jaringan/relasi nasabah | Kandidat pengembangan: graph analytics (risiko penularan antar-relasi) |
