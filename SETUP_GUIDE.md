# Setup Guide — Dashboard Proyek Cilodang

Dashboard ini terdiri dari 3 bagian yang saling terhubung:

```
Google Sheet (database)  →  Apps Script (Code.gs)  →  index.html (dashboard)
                                    │
                                    └──> Gemini API (analisa) → Email (notifikasi)
```

Ikuti langkah-langkah di bawah secara berurutan. Estimasi waktu total: 20–30 menit.

---

## 1. Buat Google Sheet

1. Buka [sheets.google.com](https://sheets.google.com) → **Blank spreadsheet**.
2. Beri nama file: `Dashboard Cilodang - PT SAL2`.
3. Buat **5 sheet/tab** dengan nama PERSIS seperti berikut (huruf besar/kecil harus sama):
   - `ProjectInfo`
   - `WBS_Bobot`
   - `KurvaS_Mingguan`
   - `Milestone`
   - `AI_Insights`
4. Untuk tiap tab, import file CSV yang sesuai dari folder `google-sheets-template/`:
   - Klik tab yang sesuai → **File → Import → Upload** → pilih file `.csv` → pilih **"Replace current sheet"** → Import data.
   - Lakukan untuk kelima file: `ProjectInfo.csv`, `WBS_Bobot.csv`, `KurvaS_Mingguan.csv`, `Milestone.csv`, `AI_Insights.csv`.
5. `AI_Insights` sengaja hanya berisi header — baris datanya akan terisi otomatis oleh Apps Script.

> **Catatan:** Data di `WBS_Bobot.csv` dan `KurvaS_Mingguan.csv` adalah hasil perhitungan yang sudah kita buat sebelumnya (lihat `wbs_cilodang.xlsx`). `RealisasiKumulatif` di `KurvaS_Mingguan` sengaja diisi 0 — update manual setiap minggu sesuai progres aktual di lapangan.

---

## 2. Pasang Google Apps Script

1. Di Google Sheet yang sama: **Extensions → Apps Script**.
2. Hapus kode default (`function myFunction() {}`), lalu tempel seluruh isi file `google-apps-script/Code.gs`.
3. Klik **Save** (ikon disket), beri nama project misalnya `Dashboard Backend`.

### 2a. Isi Script Properties (API key & email)

1. Di editor Apps Script: **Project Settings** (ikon gerigi di kiri) → scroll ke **Script Properties**.
2. Klik **Add script property**, tambahkan dua baris:
   | Property | Value |
   |---|---|
   | `GEMINI_API_KEY` | API key dari [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey) |
   | `NOTIFY_EMAILS` | Email penerima notifikasi, pisahkan koma jika lebih dari satu, contoh: `nama@astra-agro.co.id,manajer@astra-agro.co.id` |
3. Save.

### 2b. Deploy sebagai Web App (agar dashboard bisa membaca data)

1. Di editor Apps Script: **Deploy → New deployment**.
2. Klik ikon gerigi di samping "Select type" → pilih **Web app**.
3. Isi:
   - **Execute as:** Me (akun kamu)
   - **Who has access:** Anyone (data yang ditampilkan bukan data sensitif/kredensial, hanya progres proyek — jika ingin dibatasi, pilih "Anyone within [organisasi]" bila workspace mendukung)
4. Klik **Deploy**. Google akan minta otorisasi akses ke Sheet — klik **Authorize access**, pilih akun, klik **Advanced → Go to (nama project) → Allow**.
5. Salin **Web app URL** yang muncul (formatnya `https://script.google.com/macros/s/xxxxx/exec`).

### 2c. Buat trigger otomatis (cek update tiap 15 menit)

1. Masih di editor Apps Script, pilih fungsi `setupTriggers` di dropdown dekat tombol **Run**.
2. Klik **Run**. Ini akan otomatis membuat trigger time-driven yang menjalankan `checkForUpdatesAndNotify` setiap 15 menit.
3. Verifikasi: menu **Triggers** (ikon jam di kiri) — harus muncul satu trigger untuk `checkForUpdatesAndNotify`.

### 2d. Uji coba analisa AI + email

1. Pilih fungsi `manualRunAnalysis` di dropdown → **Run**.
2. Cek tab `AI_Insights` di Google Sheet — harus muncul baris baru berisi ringkasan.
3. Cek inbox email yang didaftarkan di `NOTIFY_EMAILS` — harus ada email masuk dengan subjek `[Dashboard Cilodang] Update Data Proyek...`.
4. Jika gagal, buka **Executions** (ikon di kiri) untuk melihat log error (biasanya karena API key salah atau kuota Gemini habis).

---

## 3. Hubungkan Dashboard (index.html) ke Google Sheet

1. Buka `index.html` di text editor.
2. Cari bagian `CONFIG` di dalam tag `<script>`:
   ```js
   const CONFIG = {
     SHEET_API_URL: "PASTE_YOUR_APPS_SCRIPT_WEB_APP_URL_HERE",
     AUTO_REFRESH_MS: 60000
   };
   ```
3. Ganti `SHEET_API_URL` dengan Web App URL dari langkah **2b**.
4. Simpan file, lalu buka `index.html` langsung di browser (double-click), atau host di:
   - **Google Drive** (klik kanan file → "Buka dengan" tidak akan render HTML — sebaiknya pakai opsi di bawah)
   - **GitHub Pages** (gratis, cocok untuk dashboard internal)
   - **Google Sites** (embed via HTML box)
   - Server internal perusahaan / intranet

Setelah `SHEET_API_URL` terisi, banner "Mode Demo" akan hilang otomatis dan dashboard menampilkan data asli dari Google Sheet, refresh otomatis tiap 60 detik.

---

## 4. Alur kerja rutin setelah setup selesai

| Yang terjadi | Siapa/apa yang melakukan |
|---|---|
| Update progres mingguan (isi `RealisasiKumulatif`, ubah `Status` milestone) | Tim PSM/lapangan, langsung di Google Sheet |
| Dashboard menampilkan data terbaru | Otomatis, dashboard fetch ulang tiap 60 detik |
| Deteksi perubahan data | Otomatis, trigger `checkForUpdatesAndNotify` tiap 15 menit |
| Analisa risiko & rekomendasi | Otomatis, dipanggil ke Gemini API |
| Notifikasi ke manajemen | Otomatis, email terkirim ke `NOTIFY_EMAILS` |

---

## 5. Troubleshooting

- **Dashboard tetap menunjukkan "Mode Demo"** → pastikan `SHEET_API_URL` sudah diganti dan tidak ada typo, dan deployment Web App statusnya aktif (Deploy → Manage deployments).
- **Fetch error / CORS** → pastikan "Who has access" di deployment Web App adalah **Anyone**, dan URL yang dipakai adalah URL yang diakhiri `/exec`, bukan `/dev`.
- **Email tidak terkirim** → cek kuota harian `MailApp` (100 email/hari untuk akun Gmail biasa, lebih tinggi untuk Google Workspace), dan pastikan `NOTIFY_EMAILS` terisi benar.
- **Gemini API error 429** → kuota API terlampaui; cek kuota di [aistudio.google.com](https://aistudio.google.com).
- **Trigger tidak jalan otomatis** → jalankan ulang fungsi `setupTriggers` sekali dari editor Apps Script.

---

## 6. File dalam paket ini

```
dashboard/
├── index.html                          → Dashboard (buka di browser / host di web)
├── SETUP_GUIDE.md                      → Panduan ini
├── google-apps-script/
│   └── Code.gs                         → Tempel ke Extensions > Apps Script
└── google-sheets-template/
    ├── ProjectInfo.csv                 → Import ke tab "ProjectInfo"
    ├── WBS_Bobot.csv                   → Import ke tab "WBS_Bobot"
    ├── KurvaS_Mingguan.csv             → Import ke tab "KurvaS_Mingguan"
    ├── Milestone.csv                   → Import ke tab "Milestone"
    └── AI_Insights.csv                 → Import ke tab "AI_Insights" (kosong, auto-terisi)
```
