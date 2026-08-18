# Monitoring Pusat — Portal Monitoring ASS

**Current Stable Build:** `Pusat V1.7.7.4 — PROCESS PERIOD AUTHORITY`  
**Status:** Internal / UAT  
**Final Regression:** PASS  
**Authentication:** Shared token sementara — migrasi Supabase Auth/JWT ditunda

---

## 1. Tentang Sistem

**Monitoring Pusat** adalah portal monitoring nasional untuk membaca dan menganalisis data yang dipublish dari **Website ASS**.

Arsitektur runtime:

```text
Website ASS
   ↓ Publish
Supabase
   ↓
Edge Function monitoring-pusat-read
   ↓
LiveDataStore
   ↓
ApiDataProvider
   ↓
DashboardService
   ↓
Renderer / UI Monitoring Pusat
```

Prinsip utama:

- Website ASS adalah **calculation authority** untuk KPI final yang memang dihitung di Website ASS.
- Monitoring Pusat tidak membuat ulang formula bisnis yang sudah final di Website ASS.
- PROCESS dan RESULT memiliki periode masing-masing dan tidak dipaksa menggunakan bulan yang sama.
- Hanya batch aktif (`upload_batches.is_active = true`) yang digunakan.
- Master Depot adalah hierarchy authoritative untuk Regional → Cabang → Depo.
- Tampilan waktu menggunakan `Asia/Jakarta` (WIB).

---

## 2. Current Stable Build

### Build Pusat V1.7.7.4

Fokus build:

- PROCESS KPI Summary memakai **period authority** dari Website ASS.
- Hari Ini, 7 Hari, dan Bulan Ini membaca source berbeda sesuai periodenya.
- `SC vs JKS` yang ditampilkan adalah **ON ROUTE ONLY**.
- Parent Regional/Cabang/Nasional memakai prinsip **1 Depo = 1 suara**.
- KPI Summary PROCESS tidak mengikuti filter Coverage Salesman/Project.
- RESULT Daily Depot V2 tetap menjadi authority untuk Monitoring Harian dan Trend Hasil Kerja.
- Trend Hasil Kerja desktop memakai 2 kolom penuh: **Omset + Avg EC**.
- Existing Ranking, Informasi, Anomali, Master Salesman, Master Depot, dan RESULT MTD tetap dipertahankan.

---

## 3. Menu Utama

### 3.1 Proses Kerja

KPI utama:

- SC vs JKS · On Route
- Geo Match · On Route
- Geo Match · Off Route
- Hit Rate EC
- Avg Visit
- Avg Check In
- Avg Check Out
- Avg Durasi Jalan

Fitur tambahan:

- Monitoring Harian
- Trend Proses
- Rekap Anomali
- Pola Berulang
- Drill-down organisasi
- Export JPG

### 3.2 Hasil Kerja

Fitur utama:

- KPI Summary
- Pencapaian MTD
- Monitoring Harian
- Trend Hasil Kerja
- Detail MTD
- MTD per Eceran
- Kontribusi Omset
- Drill-down organisasi
- Export JPG

### 3.3 Ranking

Ranking menggabungkan indikator PROCESS dan RESULT per Salesman.

| KPI | Bobot |
|---|---:|
| SC vs JKS | 15% |
| Geo Match | 15% |
| ROA | 15% |
| Hit Rate EC | 10% |
| Golden EC | 15% |
| Omset | 20% |
| EPB | 10% |

Kategori Final:

| Score | Kategori |
|---:|---|
| ≥ 95 | Excellent |
| ≥ 85 | Very Good |
| ≥ 75 | Good |
| ≥ 70 | Need Improvement |
| < 70 | Critical |

Ranking final hanya aktif bila PROCESS dan RESULT berada pada bulan yang sama serta TAT valid.

### 3.4 Informasi

Menampilkan status update data per Depo untuk:

- PROCESS
- RESULT
- HK
- Periode
- Waktu update
- Status keterlambatan / freshness

---

## 4. PROCESS Period Authority

### 4.1 Sumber Data

Website ASS publish extension berikut pada:

```text
upload_batches.source_meta.process_period_summary
```

Struktur:

```text
process_period_summary
├── today
├── last_7_workdays
└── mtd
```

Metadata utama:

```text
process_period_summary_available = true
process_period_summary_version = 1
process_period_summary_source = WEBSITE_ASS_FINAL_KPI
process_period_summary_scope = DEPOT_PERIOD
```

### 4.2 Mapping Periode

Monitoring Pusat membaca:

```text
Hari Ini
→ process_period_summary.today

7 Hari
→ process_period_summary.last_7_workdays

Bulan Ini
→ process_period_summary.mtd
```

Untuk backward compatibility:

```text
MTD
→ fallback ke process_depot_summary bila process_period_summary.mtd belum tersedia
```

Hari Ini dan 7 Hari **tidak boleh fallback** ke formula `process_daily` lama untuk 7 KPI Summary.

### 4.3 Definisi Periode

```text
today
= latest available report date

last_7_workdays
= last 7 canonical work dates

mtd
= month to latest available report date
```

Jadi **Hari Ini** berarti tanggal operasional terbaru yang tersedia pada data, bukan tanggal kalender browser.

### 4.4 SC vs JKS

SC vs JKS pada KPI Summary adalah:

```text
ON ROUTE ONLY
```

Basis:

```text
Y / (Y + JKS Not Visited)
```

OFF ROUTE tidak masuk denominator.

### 4.5 Parent Aggregation

Untuk scope Regional / Cabang / Nasional:

```text
1 Depo = 1 suara
```

KPI persentase/rata-rata dihitung sebagai arithmetic mean antar Depo valid.

Missing metric:

- tidak dianggap 0;
- dikeluarkan dari denominator.

Counter additive seperti total visit tetap dapat dijumlahkan.

---

## 5. RESULT Daily Depot V2

### 5.1 Sumber Data

Monitoring Harian dan Trend Hasil Kerja membaca:

```text
upload_batches.source_meta.result_daily_depot
```

Kontrak:

```text
result_daily_depot_version = 2
result_daily_date_basis = ORDER_DATE
```

Grain:

```text
Depo × Tanggal Order
```

Metric yang dipublish:

- Omset
- Avg EC

### 5.2 Tanggal Authority

Tanggal harian RESULT hanya memakai **Tanggal Order**.

Tidak boleh fallback ke:

- Delivery Date
- Tanggal Kiriman
- Tanggal Upload
- Tanggal Publish
- Cutoff Date

### 5.3 Parent Aggregation

**Omset**

```text
Parent = SUM Omset Depo
```

**Avg EC**

```text
Parent = arithmetic mean Avg EC antar Depo valid
```

Prinsip:

```text
1 Depo = 1 suara
```

### 5.4 Drill-down

```text
Regional
→ Cabang
→ Depo
→ STOP
```

Daily Salesman tidak dipublish ke Monitoring Pusat.

### 5.5 Coverage Salesman

Monitoring Harian dan Trend Hasil Kerja adalah **TOTAL DEPO**.

Keduanya:

- mengikuti Monitoring Scope;
- tidak mengikuti Coverage Salesman / Project.

RESULT MTD dan Ranking tetap dapat mengikuti Coverage Salesman/Project.

---

## 6. Master Depot & Organization

Master Depot adalah sumber hierarchy authoritative.

```text
Regional
→ Cabang
→ Depo
→ ASS
→ Salesman
```

Primary join:

```text
depot_code
```

Frontend tidak boleh membuat alias Depo production secara hardcoded.

---

## 7. Master Salesman & Project

Master Salesman adalah master lokal untuk:

- Salesman Code
- Salesman Name
- Project
- Status
- ASS
- ASS Code bila tersedia

Primary identity:

```text
1 SALESMAN CODE = 1 SALESMAN ENTITY
```

Salesman Code wajib diperlakukan sebagai **string** agar leading zero tetap aman.

Project/Coverage digunakan untuk subset seperti `TX2DA` dan berlaku untuk fitur salesman-level seperti Ranking dan detail terkait.

---

## 8. RESULT MTD Authority

KPI MTD Hasil Kerja membaca:

```text
public.result_mtd
```

Nilai MTD tidak direkonstruksi dari daily.

Metric utama mencakup:

- Actual Omset
- Return Omset
- RO
- ROA
- EC
- Avg EC MTD
- EPB
- NOO
- Target

Target kosong/null tidak boleh dianggap target 0. UI harus menampilkan `–` bila target memang belum tersedia.

---

## 9. Ranking & TAT

### 9.1 TAT

```text
TAT = HK Berjalan / Total HK Bulan × 100%
```

HK Berjalan berasal dari RESULT aktif pada Monitoring Scope. Total HK diinput manual.

Jika `HK Berjalan > Total HK`, Ranking harus diblok sampai data diperbaiki.

### 9.2 Formula Umum

Untuk KPI yang memakai TAT:

```text
Achievement = Actual / Target × 100
Score Basis = Achievement / TAT × 100
```

SC vs JKS, Geo Match, dan Hit Rate tidak memakai TAT.

---

## 10. Anomali

Severity canonical:

```text
INFO
WARNING
HIGH
CRITICAL
```

Kategori utama mencakup:

- Geo Salah
- Super Cepat
- Perjalanan Melebihi Estimasi
- Order Remote
- Visit Lama — Output Rendah
- Menumpuk

Route yang tidak memiliki bukti granular tidak boleh ditebak menjadi ON/OFF.

---

## 11. Environment Variable

### 11.1 Edge Function — Server Side

Secret/server variable **tidak boleh** masuk browser atau GitHub.

| Variable | Keterangan |
|---|---|
| `SUPABASE_URL` | URL project |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role — server only |
| `SUPABASE_ANON_KEY` | Client/public key sesuai kebutuhan auth |
| `PUSAT_READ_AUTH_MODE` | `jwt` atau `token` |
| `PUSAT_READ_ACCESS_TOKEN` | Wajib bila masih memakai mode token |
| `PUSAT_READ_ALLOWED_EMAILS` | Optional allowlist |
| `PUSAT_READ_ALLOWED_DOMAINS` | Optional allowlist |
| `PUSAT_READ_ALLOWED_ORIGINS` | Optional CORS allowlist |

**Dilarang menaruh di HTML/GitHub:**

- Service Role Key
- Database Password
- JWT Signing Secret
- Secret backend lain

### 11.2 Frontend

Konfigurasi frontend berada di:

```js
MONITORING_PUSAT_CONFIG
```

Status saat ini:

```text
AUTH_MODE = token
```

Mode ini masih dipakai untuk **Internal/UAT**. Migrasi ke Supabase Auth/JWT **ditunda**.

> Repository wajib PRIVATE selama shared token masih berada di frontend. Jangan aktifkan GitHub Pages/public deployment sebelum mekanisme auth diperbaiki.

---

## 12. Read Gateway

Edge Function:

```text
monitoring-pusat-read
```

Gateway bersifat **READ ONLY**.

Action utama:

```text
bootstrap
process
result
information
health
```

Gateway tidak boleh melakukan INSERT, UPDATE, DELETE, RPC write, atau dynamic SQL dari input client.

---

## 13. LIVE vs DEMO

Mode production:

```text
LIVE DATA · SUPABASE
```

Jika LIVE gagal:

```text
DATA ERROR
```

Aplikasi tidak boleh melakukan silent fallback ke data demo.

Demo hanya boleh aktif secara eksplisit untuk development UI.

---

## 14. UAT / Regression Status

### Stable Regression — V1.7.7.4

Status terakhir:

```text
PASS
```

Yang sudah diverifikasi secara operasional:

- PROCESS Hari Ini / 7 Hari / Bulan Ini memakai authority berbeda dan benar.
- SC vs JKS tampil sebagai ON ROUTE.
- KPI PROCESS MTD match Website ASS.
- RESULT Daily Depot V2 tampil dari Supabase.
- Omset dan Avg EC Daily/Trend sesuai source.
- Trend Hasil Kerja tampil 2 kolom penuh.
- Coverage Salesman tidak mengubah KPI managerial PROCESS.
- Coverage Salesman tidak mengubah Daily/Trend RESULT.
- Ranking tetap berjalan.
- Informasi update data tetap berjalan.
- Scope/filter tetap berjalan.
- Export JPG tetap tersedia.

### Benchmark PROCESS Tangsel — Agustus 2026

#### Hari Ini — 12 Agustus 2026

| KPI | Nilai |
|---|---:|
| SC vs JKS On Route | 100,0% |
| Geo Match On Route | 96,7% |
| Geo Match Off Route | 59,1% |
| Hit Rate EC | 47,8% |
| Avg Visit | 38 |
| Avg Check In | 08:35 |
| Avg Check Out | 15:45 |
| Avg Durasi Jalan | 2 jam 58 menit |

#### 7 Hari Kerja — 5–12 Agustus 2026

| KPI | Nilai |
|---|---:|
| SC vs JKS On Route | 94,3% |
| Geo Match On Route | 94,7% |
| Geo Match Off Route | 28,8% |
| Hit Rate EC | 39,2% |
| Avg Visit | 34 |
| Avg Check In | 08:47 |
| Avg Check Out | 16:17 |
| Avg Durasi Jalan | 3 jam 15 menit |

#### Bulan Ini — 1–12 Agustus 2026

| KPI | Nilai |
|---|---:|
| SC vs JKS On Route | 92,0% |
| Geo Match On Route | 94,5% |
| Geo Match Off Route | 31,5% |
| Hit Rate EC | 35,9% |
| Avg Visit | 34 |
| Avg Check In | 08:49 |
| Avg Check Out | 16:15 |
| Avg Durasi Jalan | 3 jam 21 menit |

---

## 15. Known Limitations

1. Authentication masih memakai shared token untuk Internal/UAT.
2. Supabase Auth/JWT belum diaktifkan.
3. GitHub Pages/public deployment belum direkomendasikan.
4. PROCESS authority period hanya tersedia pada batch yang sudah dipublish menggunakan Website ASS yang mendukung `process_period_summary`.
5. Batch PROCESS lama mungkin hanya memiliki `process_depot_summary` MTD.
6. RESULT Daily Salesman tidak dipublish; Monitoring Harian RESULT berhenti di level Depo.

---

## 16. Rollback

### Frontend

Rollback cukup mengganti file HTML ke build stable sebelumnya.

### Edge Function

Gateway tidak perlu dihapus untuk rollback UI dan dapat dikelola terpisah.

### Data

Rollback frontend tidak mengubah:

- Supabase schema
- publish batch history
- active revision
- Publish Engine
- Website ASS

---

## 17. Version History

| Build | Perubahan Utama |
|---|---|
| V1.6.6 | Resolved Organization Graph + JPG Export |
| V1.7.0 | Live Data Foundation |
| V1.7.1 | Production Schema Alignment |
| V1.7.2 | WIB Timezone + Multi-Regional Trial Scope |
| V1.7.3 | Live Anomaly Matrix Alignment |
| V1.7.4 | Ranking TAT + HK Hypercare |
| V1.7.5 | Ranking Source Alignment + Master Salesman Project |
| V1.7.5.1 | RESULT ROA UI Alignment |
| V1.7.6 | Master Salesman ASS + NOO Cleanup |
| V1.7.6.1 | Global Salesman Identity + Project-Specific ASS |
| V1.7.7 | Website ASS PROCESS Authority + RESULT Daily Depot V2 |
| V1.7.7.1 | Scope & Period Safety |
| V1.7.7.2 | PROCESS MTD Authority Routing |
| V1.7.7.3 | PROCESS Authority All Period + UI Polish |
| **V1.7.7.4** | **PROCESS Period Authority: Today / Last 7 Workdays / MTD** |

---

## 18. GitHub Recommendation

Untuk kondisi saat ini:

```text
Repository: PRIVATE
GitHub Pages: OFF
Public Deployment: OFF
```

Struktur minimal repository:

```text
monitoring-ass-pusat/
├── index.html
└── README.md
```

Sebelum public deployment:

1. migrasikan authentication ke Supabase Auth/JWT;
2. revoke/rotate shared token lama;
3. pastikan tidak ada secret di frontend atau Git history;
4. lakukan final security regression.

---

## 19. Definition of Done — Current Stable

Build `Pusat V1.7.7.4` dapat dianggap stable untuk Internal/UAT bila:

- PROCESS period authority bekerja untuk Hari Ini / 7 Hari / Bulan Ini;
- RESULT Daily Depot V2 tetap berjalan;
- Ranking berjalan tanpa duplicate entity;
- Informasi update data berjalan;
- Scope dan Coverage bekerja sesuai domain masing-masing;
- tidak ada regression pada fitur lama;
- repository tetap private selama shared token frontend masih digunakan.

---

**Monitoring Pusat — Operational Command Center**  
**Stable Build: V1.7.7.4**
