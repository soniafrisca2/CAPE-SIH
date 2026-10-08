# ICA: AI Guardrail Framework

Empat tahap guardrail (Input, Process, Output, Monitoring/Audit) di tiga lapisan (Data & ML, GenAI, Agentic AI).
Prinsip: **fail-closed** (cek kritis gagal = pipeline berhenti), **human-in-the-loop** (keputusan Approve / Reject / Further Review tetap di credit analyst), **traceable** (semua cek tercatat di `audit_log.jsonl` dengan run_id).

Status: lapisan Data & ML **sudah diimplementasikan** di `ICA_Preprocessing_to_Dashboard.ipynb` (100 cek, fungsi `guard()`, `AuditLog`, `mask_pii()`). Lapisan GenAI & Agentic AI direncanakan, dengan memakai ulang fungsi yang sama.

| Tahap | Data & ML (implemented) | GenAI (planned) | Agentic AI (planned) |
|---|---|---|---|
| Input | Kontrak skema, rentang & kategori; integritas relasi; deteksi & masking PII; fingerprint SHA-256 | `mask_pii()` sebelum ke LLM; deteksi prompt injection; batas panjang input | Klasifikasi intent; RBAC analis; validasi customer_id |
| Process | Point-in-time (fitur < tanggal aplikasi); larangan kolom outcome/ID/atribut sensitif; split per nasabah; seleksi fitur hanya di train; quality gate (recall ≥ 0,60, PR-AUC ≥ 2× base, gap OOF-test ≤ 0,10) | Prompt template terkunci; temperature 0; output JSON + validasi skema; konteks hanya evidence (skor, SHAP, data) | Hanya 3 tools read-only; allowlist; batas langkah & timeout; tanpa aksi tulis |
| Output | Skor finite [0,1]; threshold dalam [0,05; 0,95]; flag OOD (≥ 2 fitur di luar P0,5–P99,5 train); zona Further Review (± 0,05 dari threshold atau OOD); disparate impact gender/kota ≥ 0,8 | Grounding check (angka narasi = evidence); tanpa PII; bahasa netral; bukan keputusan final; confidence rendah → Further Review | Jawaban wajib menyertakan evidence & tool sumber; disclaimer keputusan di analis |
| Monitoring & Audit | Audit log JSONL; PSI baseline skor & top fitur (< 0,10 stabil); model card; fingerprint data | Log prompt/respons ter-masking; tingkat blokir; akurasi ekstraksi vs ground truth; feedback analis | Log setiap tool call + parameter; jejak keputusan analis; alert anomali |

Hasil run uji (data v4): zona Further Review berisi ±10–20% kasus dengan rate positif di antara Approve dan Reject (CR new: Approve 5,5% · Review 18,2% · Reject 31,4%), jadi kasus yang ragu-ragu benar-benar diarahkan ke analis.
