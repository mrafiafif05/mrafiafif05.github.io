# Cloudflare Worker — alteco-writer

Write proxy untuk Aliria Vault. Semua perubahan data dari browser dikirim ke Worker ini, bukan langsung ke GitHub Gist API. Worker juga menjadi proxy untuk pengiriman push notification OneSignal.

> Catatan: seluruh field sensitif (password, master password, email, phone) sudah **terenkripsi di browser** (AES-GCM) sebelum dikirim ke Worker/Gist — Worker hanya memindahkan blob terenkripsi dan melakukan merge by `id` + `updatedAt`.

---

## Secrets yang Dibutuhkan

Tambahkan di: **Cloudflare Dashboard → Workers & Pages → `alteco-writer` → Settings → Variables and Secrets → Add**

| Secret | Nilai | Keterangan |
|--------|-------|-----------|
| `GIST_ID` | Gist ID | ID dari Gist `alteco-data.json` |
| `GIST_PAT` | GitHub PAT (scope: `gist`) | Untuk baca & tulis Gist |
| `ONESIGNAL_APP_ID` | OneSignal App ID | Untuk kirim push notification |
| `ONESIGNAL_REST_KEY` | OneSignal REST API Key | Untuk otorisasi OneSignal API |

---

## Routes

| Method | Path | Fungsi |
|--------|------|--------|
| `POST /` | Default | Read-merge-write ke Gist (simpan data) |
| `POST /notify` | Notify | Forward push notification ke OneSignal |
| `OPTIONS *` | CORS preflight | Respons CORS untuk browser |

---

## Cara Deploy

1. Buka [workers.cloudflare.com](https://workers.cloudflare.com) → login
2. **Workers & Pages → Create application → Create Worker**
3. Beri nama: `alteco-writer` → **Deploy**
4. Klik **Edit code** → hapus isi default → paste kode di bawah
5. Pastikan daftar `ALLOWED` di kode berisi domain GitHub Pages-mu + origin lokal yang dipakai (lihat bagian Kode Worker)
6. Klik **Deploy**
7. Tambahkan 4 secrets (lihat tabel di atas)
8. Salin URL Worker (format: `https://alteco-writer.<subdomain>.workers.dev`)
9. Paste ke `WORKER_URL` di `index.html` (konstanta `WORKER_URL`)

---

## Kode Worker

```javascript
export default {
  async fetch(request, env) {
    const ALLOWED = [
      'https://mrafiafif05.github.io',
      'http://localhost',
      'http://localhost:8080',
      'http://127.0.0.1:8080',
    ];
    const origin = request.headers.get('Origin') || '';
    const cors = {
      'Access-Control-Allow-Origin': ALLOWED.includes(origin) ? origin : ALLOWED[0],
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };
    if (request.method === 'OPTIONS') return new Response(null, { headers: cors });
    if (request.method !== 'POST') return new Response('Method not allowed', { status: 405 });

    const url = new URL(request.url);
    const ghHdr = {
      'Authorization': `Bearer ${env.GIST_PAT}`,
      'Content-Type': 'application/json',
      'Accept': 'application/vnd.github+json',
      'User-Agent': 'alteco-worker',
    };

    try {
      if (url.pathname === '/notify') {
        const payload = await request.json();
        const r = await fetch('https://onesignal.com/api/v1/notifications', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json', 'Authorization': `Key ${env.ONESIGNAL_REST_KEY}` },
          body: JSON.stringify({ ...payload, app_id: env.ONESIGNAL_APP_ID })
        });
        const result = await r.json();
        if (!r.ok) return new Response(JSON.stringify({ error: result }), { status: r.status, headers: { ...cors, 'Content-Type': 'application/json' } });
        return new Response(JSON.stringify({ ok: true, id: result.id }), { headers: { ...cors, 'Content-Type': 'application/json' } });
      }

      // Default: read-merge-write to Gist
      const browserDb = await request.json();

      const getResp = await fetch(`https://api.github.com/gists/${env.GIST_ID}`, { headers: ghHdr });
      const gistData = await getResp.json();
      const currentContent = gistData.files?.['alteco-data.json']?.content;
      const gistDb = currentContent ? JSON.parse(currentContent) : browserDb;

      function mergeArr(gistArr = [], browserArr = []) {
        const map = new Map();
        for (const item of gistArr) map.set(item.id, item);
        for (const item of browserArr) {
          const ex = map.get(item.id);
          if (!ex || (item.updatedAt || 0) >= (ex.updatedAt || 0)) map.set(item.id, item);
        }
        // Prefer browser order (preserves reordering); gist-only items appended at end
        const bIds = new Set(browserArr.map(x => x.id));
        return [
          ...browserArr.map(x => map.get(x.id)).filter(Boolean),
          ...[...map.values()].filter(x => !bIds.has(x.id)),
        ];
      }

      function mergeTasks(gt = [], bt = []) {
        const map = new Map();
        for (const t of gt) map.set(t.id, t);
        for (const t of bt) {
          const ex = map.get(t.id);
          if (!ex || (t.updatedAt || 0) >= (ex.updatedAt || 0)) map.set(t.id, t);
        }
        return [...map.values()];
      }

      function mergeMembers(gm = [], bm = []) {
        const gMap = new Map(gm.map(m => [m.id, m]));
        const bMap = new Map(bm.map(m => [m.id, m]));
        const ids = new Set([...gMap.keys(), ...bMap.keys()]);
        return [...ids].map(id => {
          const g = gMap.get(id), b = bMap.get(id);
          if (!g) return b;
          if (!b) return g;
          return { ...g, ...b, tasks: mergeTasks(g.tasks || [], b.tasks || []) };
        });
      }

      function mergeProjects(gp = [], bp = []) {
        const gMap = new Map(gp.map(p => [p.id, p]));
        const bMap = new Map(bp.map(p => [p.id, p]));
        const ids = new Set([...gMap.keys(), ...bMap.keys()]);
        return [...ids].map(id => {
          const g = gMap.get(id), b = bMap.get(id);
          if (!g) return b;
          if (!b) return g;
          return { ...g, ...b, members: mergeMembers(g.members || [], b.members || []) };
        });
      }

      function mergeRegulasi(gr, br) {
        const base = { visi: '', misi: '', syaratAnggota: [], syaratTim: [], keuntungan: [], sanksi: [] };
        gr = { ...base, ...gr };
        br = { ...base, ...br };
        return {
          visi: br.visi,
          misi: br.misi,
          syaratAnggota: mergeArr(gr.syaratAnggota, br.syaratAnggota),
          syaratTim:     mergeArr(gr.syaratTim,     br.syaratTim),
          keuntungan:    mergeArr(gr.keuntungan,    br.keuntungan),
          sanksi:        mergeArr(gr.sanksi,         br.sanksi),
        };
      }

      const merged = {
        ...gistDb,
        ...browserDb,
        kelompoks: mergeArr(gistDb.kelompoks || [], browserDb.kelompoks || []),
        members:   mergeArr(gistDb.members   || [], browserDb.members   || []),
        tasks:     mergeArr(gistDb.tasks     || [], browserDb.tasks     || []),
        assets:    mergeArr(gistDb.assets    || [], browserDb.assets    || []),
        events:    mergeArr(gistDb.events    || [], browserDb.events    || []),
        cultures:  mergeArr(gistDb.cultures  || [], browserDb.cultures  || []),
        projects:  mergeProjects(gistDb.projects || [], browserDb.projects || []),
        regulasi:  mergeRegulasi(gistDb.regulasi || gistDb.komunitas || {}, browserDb.regulasi || {}),
        settings:  browserDb.settings,
        orgName:   browserDb.orgName,
      };

      const r = await fetch(`https://api.github.com/gists/${env.GIST_ID}`, {
        method: 'PATCH', headers: ghHdr,
        body: JSON.stringify({ files: { 'alteco-data.json': { content: JSON.stringify(merged) } } })
      });
      const result = await r.json();
      if (!r.ok) return new Response(JSON.stringify({ error: result.message }), { status: r.status, headers: { ...cors, 'Content-Type': 'application/json' } });
      return new Response(JSON.stringify({ ok: true }), { headers: { ...cors, 'Content-Type': 'application/json' } });

    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), { status: 500, headers: cors });
    }
  }
};
```

---

## Merge Strategy

Worker tidak langsung timpa Gist — melakukan **read-merge-write**:

| Tipe data | Strategi merge |
|-----------|---------------|
| Array top-level (`tasks`, `members`, `kelompoks`, `assets`, `events`, `cultures`) | Merge by `id`, item dengan `updatedAt` lebih baru menang |
| `projects[].members[].tasks[]` | Deep merge 3 level — per-task `updatedAt` menang |
| `regulasi.syaratAnggota` / `regulasi.syaratTim` / `regulasi.keuntungan` / `regulasi.sanksi` | Merge by `id` + `updatedAt` |
| `settings`, `orgName`, `visi`, `misi` | Browser always wins (last write) |

Ini mencegah data hilang saat dua user edit data berbeda secara bersamaan (race condition).

### Soft Delete (Tombstone)

Semua operasi hapus di browser **tidak** membuang item dari array — melainkan menandai `deleted: true` + memperbarui `updatedAt`. Worker tidak perlu perubahan: item bertanda `deleted` dengan `updatedAt` lebih baru akan menang di `mergeArr` dan masuk ke Gist. Render di browser sudah memfilter item dengan `deleted: true`.

---

## Update Worker

Jika ada perubahan kode Worker di masa depan:

1. Cloudflare → `alteco-writer` → **Edit code**
2. Replace seluruh kode → **Deploy**
3. Secrets tidak perlu diubah kecuali ada rotasi token
