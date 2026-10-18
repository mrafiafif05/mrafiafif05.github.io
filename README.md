# Aliria Vault

Aplikasi web single-file untuk **menyimpan kredensial database Odoo per project** — terenkripsi di browser sebelum disinkronkan ke GitHub Gist lewat Cloudflare Worker. Tanpa server backend, tanpa build tool: vanilla HTML + Web Crypto API.

---

## Fitur

- **Enkripsi klien** — semua field (master password, password DB, email, phone, dll.) dienkripsi AES-GCM-256 dengan kunci dari PBKDF2 (310k iterasi) sebelum meninggalkan browser. Gist/localStorage hanya berisi blob terenkripsi.
- **Model per project (`v:2`)** — nama project sebagai judul kartu; dalam kartu: identitas & field master (host, GitHub, go-live, **pitchdeck**, email, password email, phone, **PIC**), accordion Overview/Plan, catatan, lalu **segmented tab** berisi **General Key** (baris datar key/value — nama db = key, master password Odoo = value; **＋ Tambah** & editor inline per baris) dan **Tagihan** (nama layanan + harga — format **Rp otomatis**, boleh **negatif** untuk refund/kredit — lalu badge jatuh tempo, ✎ edit / 🗑 hapus per baris). Pilihan tab diingat **per project** (default General Key). Tagihan = **3 field**, jatuh tempo = **radio periode** yang mengisi datepicker **dari tanggal awal** (nilai datepicker saat dibuka / hasil edit manual; radio yang sudah aktif tidak mengubah tanggal) saat diklik: **Sekali bayar +0** = kembali ke tanggal awal (datepicker bebas, boleh lampau), **Bulanan +30 hari**, **Tahunan +365 hari** — datepicker selalu editable; pengulangan tagihan tetap per kalender (anchor tempo), chip ⟳ tampil untuk tagihan berulang.
- **Satu peran** — `PASS_ADMIN` (kode akses tunggal: lihat + tambah/edit/hapus + kirim notif uji).
- **Sinkron multi-device** — localStorage (instan) + Cloudflare Worker read-merge-write ke Gist (anti data-hilang saat edit bersamaan), soft-delete tombstone.
- **Sesi unlock** — tetap terbuka selama tab aktif (termasuk setelah refresh); tutup tab = login lagi. Lock manual dengan ikon 🔒.
- **Push notif siaga** — toggle subscribe OneSignal + tombol "Kirim notif uji" (admin).
- **Pengingat tagihan harian** — cron GitHub Actions meng-decrypt data di workflow lalu kirim push via OneSignal maksimal **1× per tagihan** dalam jendela N hari sebelum jatuh tempo (atur toggle/N hari/jam kirim di panel 🔔).
- Dark/light theme, search, copy-to-clipboard per field.

---

## Struktur Repo

```
/
├── index.html              ← Seluruh aplikasi (UI + crypto + sync)
├── 404.html                ← Halaman 404 (Firebase Hosting)
├── OneSignalSDKWorker.js   ← Service worker push OneSignal
├── WORKER.md               ← Kode & dokumentasi Cloudflare Worker
└── .github/workflows/
    └── daily-notif.yml     ← Cron harian: pengingat tagihan (decrypt → dedup → OneSignal)
```

---

## Deploy

### 1. GitHub Pages

Repo → **Settings → Pages → Source: Deploy from branch → branch: main → / (root)**
Situs live di `https://<username>.github.io`.

### 2. GitHub Gist (penyimpanan data)

1. Buat Gist baru di [gist.github.com](https://gist.github.com) — filename `alteco-data.json`, content:
   ```json
   {"projects":[],"meta":null,"orgName":"Aliria Vault","settings":{"pushNotifEnabled":false,"billingReminderEnabled":false,"reminderDaysBefore":3,"reminderTime":"07:00"}}
   ```
2. **Create secret gist** → salin **Gist ID** dari URL.
3. Pasang ke `index.html`:
   ```javascript
   const GIST_ID    = 'GIST_ID_DISINI';
   const WORKER_URL = 'WORKER_URL_DISINI';
   ```

### 3. GitHub PAT

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)** → scope centang **`gist`** saja.
2. Simpan token (hanya tampil sekali). **Jangan** taruh di kode/browser — hanya di Worker secret & GitHub Actions secret.

### 4. Cloudflare Worker (write proxy)

Kode lengkap ada di [`WORKER.md`](WORKER.md). Ringkas:

1. [workers.cloudflare.com](https://workers.cloudflare.com) → **Create Worker** (mis. `alteco-writer`).
2. Edit code → paste kode dari `WORKER.md` → pastikan daftar `ALLOWED` di kode CORS berisi domain GitHub Pages-mu (dan `http://localhost` bila perlu testing lokal).
3. Tambah 4 secrets: `GIST_ID`, `GIST_PAT`, `ONESIGNAL_APP_ID`, `ONESIGNAL_REST_KEY`.
4. Deploy → salin URL worker (`https://alteco-writer.<subdomain>.workers.dev`) → pasang ke `WORKER_URL` di `index.html`.

### 5. OneSignal (push notif)

1. [onesignal.com](https://onesignal.com) → **New App** → platform **Web** → Integration **Custom Code** → Site URL = domain GitHub Pages-mu.
2. Ambil **App ID** (Settings → Keys & IDs) → pasang ke `ONESIGNAL_APP_ID` di `index.html`.
3. File `OneSignalSDKWorker.js` di root sudah siap (scope root, tidak perlu diubah).

### 6. GitHub Actions (pengingat tagihan)

Workflow `.github/workflows/daily-notif.yml` jalan tiap hari pukul **06:45 WIB** (schedule `45 23 * * *`):

1. Ambil Gist → **decrypt di workflow** dengan `VAULT_PASS` (PBKDF2 310k + AES-GCM, sama dengan app) → cek `meta.check`.
2. Untuk tiap bill: hitung **jatuh tempo berikutnya** (`nextOccurrence` — `due` yang **masih ≥ hari ini dipakai apa adanya**; `once` lampau = `due`; `bulanan` lampau = hari `due` di bulan berikutnya, clamp akhir bulan; `tahunan` lampau = tanggal/bulan `due` tahun berikutnya) lalu kumpulkan yang `0 ≤ (jatuh tempo berikutnya − hari ini) ≤ reminderDaysBefore` (default 3).
3. Dedup via **`reminderLog` top-level** di Gist (bukan di `settings` — sengaja di luar supaya tidak tertimpa save browser; key = hash `id|domain|<tanggal occurrence>`) — **maksimal 1 notif per tagihan per periode** (bulanan = 1×/bulan, tahunan = 1×/tahun); entri >45 hari di-prune. Bill `once` tanpa field `freq` berperilaku persis sebelumnya (key dedup lama tetap sah).
4. Kirim OneSignal (bundle bila >1 tagihan; isi nominal diformat `Rp 150.000` — termasuk negatif `Rp -5.000` — + hint `⟳ setiap bulan/tahun`) dengan `send_after` = `reminderTime` WIB, lalu PATCH Gist menandai sudah terkirim.

Wajib 5 repository secrets: `GIST_ID`, `GIST_PAT`, `ONESIGNAL_APP_ID`, `ONESIGNAL_REST_KEY`, **`VAULT_PASS`** (isi nilai `PASS_ADMIN`). Tanpa `VAULT_PASS` workflow error agar tidak mengirim sembarangan. Bisa dijalankan manual via **workflow_dispatch** dengan input `force=true` (kirim ulang tanpa dedup). Notifikasi hanya keluar bila `pushNotifEnabled` **dan** `billingReminderEnabled` aktif (dua-duanya diatur dari panel 🔔).

---

## Cara Pakai

1. Buka situs → masukkan **kode akses** (`PASS_ADMIN`).
2. Vault kosong terbentuk otomatis pada kunjungan pertama — klik **＋ Tambah** untuk menyimpan project Odoo pertama.
3. Isi form: nama project (wajib), host/GitHub/go-live/pitchdeck (link — baru bisa diklik bila diawali `https://`), overview/plan (teks, bisa dilipat di kartu), email + password email + phone + **PIC**, lalu catatan. **General Key** dan **Tagihan** tidak diisi lewat form — kelola lewat **tab** di dalam kartu (tab diingat **per project**, default General Key): **＋ Tambah** / **✎** / **🗑** per baris; tagihan = 3 field — nama layanan (wajib), harga (format **Rp otomatis**, **boleh negatif** untuk refund/kredit), dan jatuh tempo berupa **radio periode** yang **mengisi datepicker dari tanggal awal** (tersimpan saat dibuka / hasil edit manual; radio yang sudah aktif tidak mengubah tanggal) — **Sekali bayar +0** kembali ke tanggal awal (datepicker bebas, boleh lampau), **Bulanan +30 hari**, **Tahunan +365 hari** — datepicker tetap bebas diedit; pengulangan tetap per kalender dari anchor tempo).
4. **Sesi**: setelah unlock, sesi disimpan per-tab (`sessionStorage`) — refresh tetap masuk otomatis; **tutup tab** atau klik **🔒 Lock** untuk keluar.

### Ubah kode akses

Kode akses ada di `index.html`:
```javascript
const PASS_ADMIN = '<kode-akses>';
```
Ganti sesuai kebutuhan, commit & push. `PASS_LEGACY` hanya kunci migrasi otomatis dari kode lama — bisa dihapus setelah semua perangkat pernah membuka vault dengan kode baru.

> **Penting**: kunci enkripsi vault diturunkan dari `PASS_ADMIN`. Mengganti kode kini **bermigrasi otomatis** saat unlock pertama (re-encrypt seluruh data + salt baru; kode lama langsung tidak berlaku lagi). **Segera update secret `VAULT_PASS` di GitHub Actions ke kode baru** supaya workflow notifikasi tetap bisa mendekripsi Gist, dan backup Gist sebelum commit.

---

## Model Keamanan (baca ini)

| Aspek | Detail |
|-------|--------|
| Enkripsi | AES-GCM-256; key = PBKDF2-SHA256(`PASS_ADMIN`, salt acak 16B, 310k iterasi) |
| Yang plaintext di Gist | Hanya `id`, `updatedAt`, `deleted`, `iv`, `meta.salt` (dibutuhkan merge & unlock) |
| Yang terenkripsi | Semua field project (model `v:2`): nama, host, github, goLive, pitchdeck, overview, plan, email, emailPassword, phone, pic, `dbs[]` (key/value — dinormalisasi dari bentuk lama dbName/masterPass saat dibuka), `bills[]` (domain/nominal/freq/due/url), notes |
| Kunci | Hidup di memori JS selama sesi unlock; tidak pernah ditulis ke localStorage/Gist |
| Sesi unlock | Disimpan di `sessionStorage` (per-tab, hilang saat tab ditutup) agar refresh tidak perlu login ulang; tombol Lock menghapusnya |
| Validasi password | `meta.check` — blob terenkripsi berisi magic string; gagal decrypt = "vault rusak" |
| Batasan jujur | Source code repo publik ⇒ pembaca source + Gist ID **bisa** menurunkan kunci sendiri. Enkripsi ini melindungi dari pembaca Gist kasual, penyimpanan lokal browser, dan file backup — sesuai threat model komunitas internal. Untuk proteksi lebih kuat: gunakan master password rahasia yang tidak di-commit, atau pindah ke storage ber-auth (Worker + KV/D1). |

---

## Troubleshooting

| Gejala | Penyebab / solusi |
|--------|-------------------|
| "Butuh koneksi aman" | Web Crypto butuh HTTPS/localhost — jangan akses via IP LAN |
| Banner "Gagal tersinkron" | Worker down / CORS origin belum diperbarui di kode Worker → cek daftar `ALLOWED` (harus memuat origin halaman, tanpa trailing slash) |
| List beda antar browser | Save gagal (CORS) sehingga tiap browser hanya punya cache lokal → perbaiki CORS Worker lalu **hard refresh** kedua browser. Item dari vault lama otomatis di-re-key ke vault server saat unlock |
| Banner "Vault lokal berbeda dari server" | Pernah ada dua vault dengan salt berbeda — item lokal otomatis dipindahkan (re-key) ke vault server saat unlock. Jika muncul "N item tidak terbaca", item itu dibuat dengan kunci yang sudah tidak tersedia di perangkat ini |
| "Vault rusak" | `meta.check` gagal didekripsi — salt/check di Gist tidak cocok (jangan timpa Gist; pulihkan dari backup) |
| Selalu minta login saat refresh | Seharusnya tidak terjadi — sesi ada di `sessionStorage`. Cek apakah browser memblokir storage (mode incognito tertentu) |
| Notif uji gagal | Cek 4 secrets Worker (`ONESIGNAL_*`, `GIST_*`) dan CORS |
| Pengingat tagihan tidak masuk | Cek 5 secrets Actions (termasuk `VAULT_PASS`), status workflow di tab Actions, dan kedua toggle di panel 🔔 (`pushNotifEnabled` + `billingReminderEnabled`) |
| Project lama (model sebelum revisi) tidak tampil | Sengaja — hanya item `v:2` yang dibaca ("Mulai baru"); data lama masih utuh terenkripsi di Gist. Bila perlu, salin isinya manual: identitas lewat form Tambah, isi tab General Key / Tagihan lewat tombol **＋** di kartu |
| Data lama Kelola Alteco masih menempel di Gist | Aman — item lama tidak punya `iv`/`payload`, difilter dari tampilan. Kosongkan file Gist manual jika ingin bersih |

---

## Sinkronisasi Antar Browser

Agar list sama di beberapa browser (mis. VS Code browser & Chrome):

1. **Perbaiki CORS Worker sekali** — paste kode dari `WORKER.md` (daftar `ALLOWED` sudah memuat `https://mrafiafif05.github.io` + `http://localhost*`) lalu Deploy.
2. **Hard refresh** (`Ctrl+Shift+R`) kedua browser agar memakai kode terbaru.
3. Unlock di masing-masing browser — jika perangkat punya vault lokal dengan salt berbeda, app **otomatis memindahkan (re-key) item lokal ke vault server** dan menyinkronkannya. Banner akan memberi tahu hasilnya.
4. Setelah itu list menyatu (union) di semua browser. Bila ada duplikat (item yang sempat dibuat terpisah di dua perangkat), hapus salah satu lewat tombol 🗑.

> Jika muncul "N item tidak terbaca" — item itu dibuat oleh vault dengan kunci yang tidak tersedia di perangkat tersebut; salin ulang datanya manual lewat form Tambah (identitas) dan tombol **＋** di tab General Key / Tagihan.

---

## Backup

Data = Gist `alteco-data.json` (terenkripsi) + salt di dalamnya. Untuk backup penuh: unduh isi Gist secara berkala dan simpan di tempat aman. Tanpa `meta.salt`, backup blob tidak bisa dibuka.
