Ini file **`README.md`** utuh yang sudah dirapikan dalam satu blok penuh supaya bisa langsung kamu salin (*copy*) dan simpan ke root repository **`kosan-kipli`**:

```markdown
# 🏠 Kosan Kipli — Panduan Penghuni Multi-Bilik Server

Selamat datang di **Kosan Kipli** (`48.193.45.144`), server VPS Ubuntu multi-tenant dengan isolasi kamar per-user (`chmod 711`), Nginx Reverse Proxy, PHP 8.5-FPM, Node.js v22 LTS, PM2, dan Fail2Ban.

Setiap penghuni memiliki **kunci kamar (user Linux) masing-masing**, bebas deploy aplikasi mandiri tanpa akses `sudo`, dan tidak bisa mengintip isi kamar penghuni lain.

---

## 📋 Daftar Kamar & Jatah Jalur Publik

| Kamar | Penghuni | Username | Jatah Port Internal | URL Web Statis / PHP | URL App Node.js (PM2) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Kantor Kos** | Admin (`pli`) | `pli` | `3000` (3001–3009) | `http://48.193.45.144/` | — |
| **Kamar 01** | Mas Nawawow | `nawawow` | `3010` (3011–3019) | `http://48.193.45.144/~nawawow/` | `http://48.193.45.144/nawawow/` |
| **Kamar 02** | *(Tersedia)* | `<username>` | `3020` (3021–3029) | `http://48.193.45.144/~<username>/` | `http://48.193.45.144/<username>/` |

---

## 🔑 1. Login ke Kamar Masing-Masing

Masuk lewat terminal SSH menggunakan username dan password yang diberikan Juragan Kos (`pli`):

```bash
ssh <username>@48.193.45.144
```

> **⚠️ Peringatan Keamanan:** Server dilindungi **Fail2Ban**. Salah memasukkan password sebanyak **5 kali dalam 10 menit** akan otomatis memblokir IP kamu sementara.

---

## 🌐 2. Cara Deploy Web Statis (HTML/CSS/JS) atau PHP

Semua file yang diletakkan di dalam folder `~/public_html` otomatis tayang di `http://48.193.45.144/~<username>/` tanpa perlu me-restart server.

```bash
# Contoh membuat halaman tes di kamar sendiri
echo "<h1>Halo dari Kamar <username>!</h1>" > ~/public_html/index.html
```

* Mendukung file **`.html`** maupun **`.php`** (PHP 8.5).
* **Catatan Keamanan PHP:** Eksekusi file `.php` di dalam subfolder `uploads/`, `files/`, `media/`, atau `images/` otomatis diblokir (`403 Forbidden`) demi mencegah celah *upload-to-shell*.

---

## ⚡ 3. Cara Deploy Aplikasi Node.js (Express / Next.js)

Gunakan **PM2** dan wajib jalankan aplikasi di **port jatah kamarmu** (misal `3010` untuk Kamar 01) serta bind ke `127.0.0.1`:

### Contoh Express.js
Pastikan file utama membaca `process.env.PORT`:

```js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3010; // Sesuaikan dengan port kamarmu

app.get('/', (req, res) => {
  res.json({ status: 'online', kamar: 'Kamar 01' });
});

app.listen(PORT, '127.0.0.1', () => {
  console.log(`Berjalan di [http://127.0.0.1](http://127.0.0.1):${PORT}`);
});
```

Jalankan dengan PM2 di terminal kamarmu:

```bash
cd ~/nama-project
npm install --omit=dev
PORT=3010 pm2 start index.js --name "app-<username>"
pm2 save
```

> **Catatan untuk Next.js:** Karena aplikasi diakses lewat sub-path (`/<username>`), tambahkan `basePath: '/<username>'` di dalam `next.config.mjs` sebelum `npm run build` agar aset CSS/JS tidak patah.

---

## 🤖 4. Setup Auto-Deploy CI/CD (GitHub Actions)

Karena RAM server terbatas (1 GiB), sangat disarankan melakukan proses `npm run build` di runner GitHub Actions lalu mengirim hasilnya ke kamarmu via SSH.

### Langkah A: Buat SSH Key di Kamar Sendiri
Jalankan di terminal `<username>@48.193.45.144`:

```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
cat ~/.ssh/github_actions
```

Salin seluruh output Private Key (`-----BEGIN OPENSSH PRIVATE KEY-----` sampai `-----END OPENSSH PRIVATE KEY-----`).

### Langkah B: Tambahkan Repository Secrets di GitHub
Buka repository proyekmu di GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

* **`SSH_HOST`**: `48.193.45.144`
* **`SSH_USER`**: `<username>` *(contoh: `nawawow`)*
* **`SSH_PRIVATE_KEY`**: *(Tempel Private Key dari Langkah A)*

### Langkah C: Buat File `.github/workflows/deploy.yml`

#### Template 1: Web Statis / Vite / React Build (`~/public_html`)
```yaml
name: 🚀 Deploy Static ke Kosan Kipli

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm ci && npm run build
      - name: Upload hasil build ke public_html
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/*"
          target: "~/public_html"
          strip_components: 1
```

#### Template 2: Backend Express / Node.js (PM2)
```yaml
name: ⚡ Deploy Node.js ke Kosan Kipli

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Kirim kode ke kamar
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ".,!node_modules,!.git"
          target: "~/app-project"
      - name: Install dependencies & Restart PM2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/app-project
            npm install --omit=dev
            pm2 restart app-nawawow || PORT=3010 pm2 start index.js --name "app-nawawow"
            pm2 save
```

---

## 📜 Tata Tertib Kosan Kipli

1. **Jangan jalankan `npm run build` berat langsung di server** (seperti Next.js skala besar) tanpa mengandalkan GitHub Actions agar RAM 1 GiB tidak kehabisan memori.
2. **Gunakan port sesuai jatah kamar** — dilarang memakai port milik kamar lain.
3. **Selalu jalankan `npm audit fix`** sebelum deploy proyek Node.js/React ke server.
4. **Jangan ubah permission folder `/home/<username>`** ke `755` atau `777` (biarkan `711`) agar privasi file proyekmu tetap terlindungi dari penghuni lain.

```

Cukup klik tombol copy di pojok kanan atas blok kode di atas, paste ke file `README.md`, lalu `git push` ke repo `kosan-kipli`!