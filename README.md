# Sistem Penjejakan Jangka Hayat Mesin (Machine Lifetime Tracker)

## Gambaran Keseluruhan

Sistem ini adalah aplikasi web untuk menjejak jangka hayat mesin dan status kesihatan berdasarkan tarikh pembelian dan rekod penyelenggaraan. Ia menggunakan **Supabase** sebagai pangkalan data backend dan boleh di-embed dalam laman web WordPress.

---

## 🎯 Komponen Utama

Sistem ini terdiri daripada 3 komponen utama:

### 1. **Widget Jangka Hayat Mesin** (`machine-lifetime-widget.html`)
Widget utama untuk pengurusan penuh mesin dan penyelenggaraan.

### 2. **Widget Status Kesihatan** (`machine-health-status-widget.html`)
Widget khusus untuk memaparkan status kesihatan satu mesin sahaja (untuk halaman individu).

### 3. **Pangkalan Data Supabase**
Menggunakan 3 jadual utama: `users`, `machines`, `maintenances`

---

## 📊 Formula & Kiraan Jangka Hayat

### Konsep Asas
- **Jangka hayat asas**: Setiap mesin bermula dengan 100% dan menurun secara linear ke 0% dalam **10 tahun** (3,650 hari)
- **Formula linear**: `Peratus = 100% - (Hari Berlalu / 3650 hari × 100%)`

### Kesan Penyelenggaraan (Maintenance Effects)
Penyelenggaraan menambah jangka hayat mesin:

| Jenis Penyelenggaraan | Peningkatan Peratus | Bersamaan Hari Ditambah |
|----------------------|---------------------|------------------------|
| **Kendiri** (Self maintenance) | +0.001% | ~0.365 hari (~9 jam) |
| **Major** (Professional service) | +0.01% | ~3.65 hari (~88 jam) |

### Pengiraan "Smooth Effect" (6 Bulan)
Kesan penyelenggaraan tidak segera penuh, tetapi meningkat secara beransur-ansur menggunakan **ease-in-out sine curve** selama 6 bulan:

\`\`\`
Progress = min(1, (BulanSelepas + 1) / 6)
Factor = -(cos(π × Progress) - 1) / 2
KesampaianSebenar = AsasPeratus + (BoostPenyelenggaraan × Factor)
\`\`\`

**Contoh Praktik:**
- Bulan 1 selepas major service: +~0.001% (10% daripada +0.01%)
- Bulan 3 selepas major service: +~0.005% (50% daripada +0.01%)
- Bulan 6 selepas major service: +0.01% (100% penuh)

---

## 🏥 Kiraan Status Kesihatan Mesin

Status kesihatan dikira berdasarkan **2 faktor**:

### Faktor 1: Peratus Jangka Hayat Semasa
\`\`\`
JangkaHayatSemasa = PeratusAsas + Σ(KesampaianPenyelenggaraan)
\`\`\`

### Faktor 2: Masa Sejak Penyelenggaraan Terakhir
\`\`\`
BulanSejakPenyelenggaraan = (TarikhHariIni - TarikhPenyelenggaraanTerakhir) dalam bulan
\`\`\`

### Logik Status Kesihatan

| Status | Kriteria | Warna | Tindakan |
|--------|----------|-------|----------|
| **Baik** | Jangka hayat > 60% **DAN** penyelenggaraan dalam 1 bulan terakhir | 🟢 Hijau | Tiada tindakan |
| **Sederhana (Perlu Perhatian)** | Jangka hayat 40-60% **ATAU** penyelenggaraan 1-12 bulan lepas | 🟡 Kuning | Rancang penyelenggaraan |
| **Buruk (Perlu Penyelenggaraan)** | Jangka hayat < 40% **DAN** tiada penyelenggaraan 12+ bulan | 🔴 Merah | Tindakan segera! |

**Pseudocode:**
\`\`\`
if (jangkaHayat > 60% AND bulanSejakPenyelenggaraan <= 1):
    status = "Baik"
else if (jangkaHayat > 40% OR bulanSejakPenyelenggaraan <= 12):
    status = "Sederhana"
else:
    status = "Buruk"
\`\`\`

---

## 🗄️ Struktur Pangkalan Data

### Jadual 1: `users` (Auth Users)
Diuruskan oleh Supabase Auth secara automatik.

| Kolum | Jenis | Keterangan |
|-------|-------|------------|
| `id` | UUID | Primary key (auto) |
| `email` | TEXT | Email pengguna |
| `created_at` | TIMESTAMPTZ | Tarikh daftar |

---

### Jadual 2: `machines`
Menyimpan maklumat mesin.

| Kolum | Jenis | Keterangan | Contoh |
|-------|-------|------------|--------|
| `id` | UUID | Primary key (auto) | `a1b2c3d4-...` |
| `user_id` | UUID | Foreign key ke `auth.users` | `e5f6g7h8-...` |
| `machine_id` | TEXT | ID unik mesin (ditentukan pengguna) | `"SMAW-001"` |
| `name` | TEXT | Nama mesin | `"SMAW 1"` |
| `purchase_date` | DATE | Tarikh pembelian | `2020-03-15` |
| `created_at` | TIMESTAMPTZ | Tarikh dicipta dalam sistem | `2025-01-07T10:30:00Z` |
| `updated_at` | TIMESTAMPTZ | Tarikh dikemaskini | `2025-01-07T10:30:00Z` |

**Constraints:**
- `UNIQUE(user_id, machine_id)` — Seorang pengguna tidak boleh ada 2 mesin dengan ID yang sama
- Row Level Security (RLS) aktif — Pengguna hanya boleh akses mesin sendiri

---

### Jadual 3: `maintenances`
Menyimpan rekod penyelenggaraan.

| Kolum | Jenis | Keterangan | Contoh |
|-------|-------|------------|--------|
| `id` | UUID | Primary key (auto) | `j9k0l1m2-...` |
| `user_id` | UUID | Foreign key ke `auth.users` | `e5f6g7h8-...` |
| `machine_id` | UUID | Foreign key ke `machines` | `a1b2c3d4-...` |
| `maintenance_date` | DATE | Tarikh penyelenggaraan dilakukan | `2024-06-20` |
| `maintainer_name` | TEXT | Nama orang yang menyelenggara | `"Ahmad bin Ali"` |
| `maintainer_role` | TEXT | Peranan penyelenggara | `"Student"` / `"Staff"` / `"Technician"` |
| `maintenance_type` | TEXT | Jenis penyelenggaraan | `"kendiri"` / `"major"` |
| `notes` | TEXT | Nota tambahan (optional) | `"Tukar bearing motor"` |
| `created_at` | TIMESTAMPTZ | Tarikh dicipta | `2024-06-20T14:20:00Z` |

**Constraints:**
- Row Level Security (RLS) aktif — Pengguna hanya boleh akses maintenance records sendiri

---

## 🔄 Aliran Proses (Process Flows)

### 🔐 Flow 1: Pendaftaran & Log Masuk Pengguna

\`\`\`
┌─────────────────┐
│ Pengguna Buka   │
│ Widget          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│ Check Auth      │─NO──▶│ Tunjuk Skrin     │
│ (Supabase)      │      │ Login/Register   │
└────────┬────────┘      └────────┬─────────┘
         │                        │
         │ YES                    │
         ▼                        ▼
┌─────────────────┐      ┌──────────────────┐
│ Load User's     │◀─────│ User Login/      │
│ Machines        │      │ Register         │
└─────────────────┘      └──────────────────┘
\`\`\`

**Input:**
- Email pengguna
- Password (minimum 6 characters)
- Username (optional, untuk display)

**Output:**
- Pengguna berjaya log masuk
- Session token disimpan dalam browser
- Redirect ke skrin utama aplikasi

**Error Handling:**
- Email sudah wujud → Guna log masuk
- Email belum verified → Tunjuk mesej "Check email untuk verification"
- Password salah → Tunjuk error message

---

### ⚙️ Flow 2: Tambah / Simpan Mesin Baru

\`\`\`
┌─────────────────┐
│ User Isi Form:  │
│ - Nama Mesin    │
│ - ID Mesin      │
│ - Tarikh Beli   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Click "Tambah/  │
│ Simpan Mesin"   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ Supabase INSERT/UPSERT      │
│ ke jadual 'machines'        │
│ ON CONFLICT (user_id,       │
│ machine_id) DO UPDATE       │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│ Success?        │─NO──▶│ Tunjuk Error     │
└────────┬────────┘      └──────────────────┘
         │ YES
         ▼
┌─────────────────┐
│ Refresh List    │
│ Mesin & Auto    │
│ Select Mesin    │
│ Baru            │
└─────────────────┘
\`\`\`

**Input:**
- `name` (TEXT): Nama mesin, contoh "SMAW 1", "CMM"
- `machine_id` (TEXT): ID unik, contoh "SMAW-001"
- `purchase_date` (DATE): Tarikh pembelian dalam format YYYY-MM-DD

**Output:**
- Mesin baru disimpan ke database
- Mesin muncul dalam dropdown "Mesin Sedia Ada"
- Chart kosong ditunjukkan (tiada penyelenggaraan lagi)

**Validasi:**
- Semua field wajib diisi
- `machine_id` mesti unik untuk user tersebut (enforced by database)

---

### 📈 Flow 3: Papar Jangka Hayat & Status Kesihatan

\`\`\`
┌─────────────────┐
│ User Pilih      │
│ Mesin dari      │
│ Dropdown        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Load Machine    │
│ Details dari DB │
└────────┬────────┘
         │
         ▼
┌──────────────────────────┐
│ Load Maintenance Records │
│ untuk mesin tersebut     │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Kira Jangka Hayat:       │
│ 1. Baseline (linear)     │
│ 2. Boost (maintenance)   │
│ 3. Smooth over 6 months  │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Kira Status Kesihatan:   │
│ - Check peratus jangka   │
│ - Check last maintenance │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Render Chart.js Graph    │
│ & Info Panel             │
└──────────────────────────┘
\`\`\`

**Input:**
- `machine_id` (UUID) mesin yang dipilih

**Output:**
\`\`\`javascript
{
  // Maklumat Mesin
  machineName: "SMAW 1",
  machineID: "SMAW-001",
  purchaseDate: "2020-03-15",
  
  // Kiraan Jangka Hayat
  baselineLifetimeToday: 45.67,        // % (linear tanpa maintenance)
  adjustedLifetimeToday: 52.34,        // % (dengan maintenance)
  baselineEOL: "2030-03-15",           // Tarikh tamat hayat asas
  adjustedEOL: "2031-01-20",           // Tarikh tamat hayat selepas maintenance
  
  // Status Kesihatan
  healthStatus: "Sederhana (Perlu Perhatian)",  // "Baik" / "Sederhana" / "Buruk"
  healthStatusClass: "health-sederhana",        // CSS class untuk styling
  
  // Maintenance Info
  lastMaintenanceDate: "2024-11-15",
  totalMaintenanceRecords: 12,
  monthsSinceLastMaintenance: 2
}
\`\`\`

**Visual Output:**
- **Chart Graf**: Garis biru (adjusted lifetime) vs garis putus-putus kelabu (baseline)
- **Card Status**: Kotak berwarna menunjukkan status kesihatan
- **Info Panel**: Details tarikh beli, jangka hayat semasa, dll.

---

### 🔧 Flow 4: Tambah Rekod Penyelenggaraan

\`\`\`
┌─────────────────┐
│ User Isi Form:  │
│ - Tarikh        │
│ - Nama          │
│ - Role          │
│ - Jenis         │
│ - Nota          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Validasi:       │
│ - Tarikh < beli?│
│ - Field kosong? │
└────────┬────────┘
         │ Valid
         ▼
┌─────────────────────────────┐
│ Supabase INSERT ke jadual   │
│ 'maintenances'              │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────┐
│ Auto Refresh:   │
│ - Reload logs   │
│ - Recalculate   │
│   jangka hayat  │
│ - Update chart  │
└─────────────────┘
\`\`\`

**Input:**
\`\`\`javascript
{
  machine_id: "a1b2c3d4-...",           // UUID (auto dari selected machine)
  user_id: "e5f6g7h8-...",              // UUID (auto dari logged user)
  maintenance_date: "2024-12-05",       // DATE
  maintainer_name: "Ahmad bin Ali",     // TEXT
  maintainer_role: "Student",           // "Student" | "Staff" | "Technician"
  maintenance_type: "kendiri",          // "kendiri" | "major"
  notes: "Tukar oil & bersih filter"    // TEXT (optional)
}
\`\`\`

**Output:**
- Rekod disimpan ke database
- Log baru muncul dalam "Sejarah Penyelenggaraan"
- Chart dikemaskini dengan marker baru (bulatan hijau untuk kendiri, triangle merah untuk major)
- Jangka hayat mesin meningkat sesuai dengan jenis penyelenggaraan
- Status kesihatan mungkin berubah ke "Baik" jika memenuhi kriteria

**Validasi:**
- `maintenance_date` tidak boleh sebelum `purchase_date` mesin
- `maintainer_name` wajib diisi
- Tarikh tidak boleh masa depan (optional check)

---

### 📊 Flow 5: Widget Status Kesihatan (Halaman Individu)

\`\`\`
┌─────────────────────────────┐
│ User Embed Widget di        │
│ WordPress Page dengan:      │
│ window.MACHINE_NAME="SMAW 1"│
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Widget Load & Detect        │
│ Machine Name dari Variable  │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Query Supabase:             │
│ SELECT * FROM machines      │
│ WHERE name ILIKE 'SMAW 1'   │
└────────┬────────────────────┘
         │
         ▼
┌──────────────┐      ┌──────────────────┐
│ Machine      │─NO──▶│ Tunjuk "Data     │
│ Wujud?       │      │ Belum            │
└────────┬─────┘      │ Dikemaskini"     │
         │ YES        └──────────────────┘
         ▼
┌─────────────────────────────┐
│ Load Maintenance Records    │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Kira Status Kesihatan       │
│ (Sama seperti Flow 3)       │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Render Status Card:         │
│ - Status (Baik/Sederhana/   │
│   Buruk)                    │
│ - ID Mesin                  │
│ - Tarikh Beli               │
│ - Jangka Hayat %            │
│ - Last Maintenance          │
└─────────────────────────────┘
\`\`\`

**Input:**
- `window.MACHINE_NAME` (STRING): Nama mesin yang ditetapkan dalam WordPress

**Output (Contoh untuk SMAW 1):**
\`\`\`html
<div class="health-status-card health-sederhana">
  Sederhana (Perlu Perhatian)
</div>
<div>ID Mesin: SMAW-001</div>
<div>Tarikh Pembelian: 2020-03-15</div>
<div>Jangka Hayat Semasa: 52.34%</div>
<div>Penyelenggaraan Terakhir: 2024-11-15</div>
<div>Jumlah Penyelenggaraan: 12 rekod</div>
\`\`\`

**Cara Guna di WordPress:**
1. Tambah Custom HTML Block
2. Copy kod dari `machine-health-status-widget.html`
3. Tambah di atas kod: `<script>window.MACHINE_NAME = "SMAW 1";</script>`
4. Publish

**Kelebihan:**
- Ringan (no login required)
- Auto-update bila data berubah
- Boleh embed di mana-mana page (SMAW 1 page, SMAW 2 page, dll)

---

## 🛠️ Fungsi Tambahan

### Generate Dummy Past Records
\`\`\`
┌─────────────────┐
│ User Click      │
│ "Generate Dummy"│
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ Generate Monthly 'kendiri'  │
│ + Semiannual 'major' dari   │
│ purchase date → today       │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Bulk INSERT ke 'maintenances'│
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Auto refresh chart & logs   │
└─────────────────────────────┘
\`\`\`

**Tujuan:** Untuk testing atau populate data lama secara cepat

**Output:**
- Rekod bulanan `kendiri` (setiap bulan)
- Rekod `major` setiap 6 bulan
- Data dari tarikh beli hingga hari ini

---

## 🔒 Keselamatan (Row Level Security)

### Polisi RLS untuk `machines`
\`\`\`sql
-- Users can only see their own machines
CREATE POLICY "Users can view their own machines"
  ON machines FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert/update/delete their own machines
CREATE POLICY "Users can insert their own machines"
  ON machines FOR INSERT
  WITH CHECK (auth.uid() = user_id);
\`\`\`

### Polisi RLS untuk `maintenances`
\`\`\`sql
-- Users can only see maintenance records for their machines
CREATE POLICY "Users can view their own maintenance records"
  ON maintenances FOR SELECT
  USING (auth.uid() = user_id);
\`\`\`

**Implikasi:**
- User A tidak boleh lihat mesin User B
- Semua query automatik filtered by `user_id`
- Widget health status (public) masih boleh access kerana ia query by `name` (bukan protected by RLS jika mesin set sebagai public — currently setup untuk private only)

---

## 📦 Cara Setup & Deploy

### 1. Setup Supabase
1. Buat project baru di [supabase.com](https://supabase.com)
2. Run semua SQL scripts dalam folder `/scripts`:
   - `001_create_users_table.sql`
   - `002_create_machines_table.sql`
   - `003_create_maintenances_table.sql`
   - `004_create_profile_trigger.sql`

3. Dapatkan credentials:
   - `SUPABASE_URL` dari Settings → API
   - `SUPABASE_ANON_KEY` dari Settings → API

### 2. Update HTML Files
Replace dalam kedua-dua file `machine-lifetime-widget.html` dan `machine-health-status-widget.html`:

\`\`\`javascript
const SUPABASE_URL = 'YOUR_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY';
\`\`\`

### 3. Embed di WordPress
#### Widget Utama (Full Management)
1. Edit halaman WordPress
2. Tambah block "Custom HTML"
3. Copy & paste semua kod dari `machine-lifetime-widget.html`
4. Publish

#### Widget Status (Per Machine Page)
1. Edit halaman mesin (contoh: "SMAW 1" page)
2. Tambah block "Custom HTML"
3. Tambah di atas sekali:
   \`\`\`html
   <script>window.MACHINE_NAME = "SMAW 1";</script>
   \`\`\`
4. Copy & paste kod dari `machine-health-status-widget.html`
5. Publish

---

## 🎨 Customization

### Tukar Warna Status
Edit dalam `<style>` section:

\`\`\`css
.health-baik {
  background-color: #dcfce7;  /* Hijau muda */
  color: #166534;             /* Hijau gelap */
}

.health-sederhana {
  background-color: #fefce8;  /* Kuning muda */
  color: #854d0e;             /* Kuning gelap */
}

.health-buruk {
  background-color: #FFF2F2;  /* Merah muda */
  color: #E22627;             /* Merah */
}
\`\`\`

### Tukar Kriteria Status Kesihatan
Edit dalam JavaScript function `showMachineInfo()`:

\`\`\`javascript
// Semasa:
if (adjustedNow > 60 && monthsSinceLastMaint <= 1) {
    healthStatus = 'Baik';
}

// Contoh perubahan: Longgarkan syarat "Baik"
if (adjustedNow > 50 && monthsSinceLastMaint <= 3) {
    healthStatus = 'Baik';
}
\`\`\`

### Tukar Kesan Penyelenggaraan
Edit nilai dalam JavaScript:

\`\`\`javascript
// Semasa:
const EFFECTS = { kendiri: 0.001, major: 0.01 };

// Contoh: Tingkatkan kesan maintenance
const EFFECTS = { kendiri: 0.005, major: 0.05 };
\`\`\`

---

## 🐛 Troubleshooting

### Issue: "Could not find table 'public.machines'"
**Sebab:** Database tables belum dicipta  
**Penyelesaian:** Run semua SQL scripts dalam folder `/scripts`

### Issue: "Email verification required"
**Sebab:** Supabase Auth wajibkan email verification  
**Penyelesaian:** 
1. Check inbox/spam untuk email dari Supabase
2. Click link verify
3. Log masuk semula

### Issue: Widget status tunjuk "Data Belum Dikemaskini"
**Sebab:** Nama mesin dalam `window.MACHINE_NAME` tidak match dengan database  
**Penyelesaian:**
1. Check spelling — case-sensitive! ("SMAW 1" ≠ "smaw 1")
2. Pastikan mesin sudah didaftar dalam widget utama
3. Check browser console untuk error messages

### Issue: Chart tidak keluar / canvas blank
**Sebab:** Chart.js library tidak load  
**Penyelesaian:** Pastikan line ini ada sebelum `</script>`:
\`\`\`html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.3.0/dist/chart.umd.min.js"></script>
\`\`\`

---

## 📞 Support & Dokumentasi

Untuk bantuan lanjut:
- Check browser console (F12) untuk error messages
- Verify Supabase credentials betul
- Pastikan semua SQL scripts sudah run
- Check RLS policies aktif di Supabase dashboard

---

## 📝 Changelog

### Version 1.0 (Current)
- ✅ Authentication dengan Supabase Auth
- ✅ Machine management (add, view, update)
- ✅ Maintenance logging
- ✅ Linear baseline + smooth maintenance effects
- ✅ Health status calculation
- ✅ Chart visualization
- ✅ Standalone health status widget
- ✅ WordPress embeddable
- ✅ Row Level Security (RLS)

---

**Dicipta untuk Penjejakan Jangka Hayat Mesin Industri**  
**Menggunakan:** Next.js, Supabase, Chart.js, WordPress Integration
