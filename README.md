# 🏠 Kosan Kipli — Panduan Penghuni Multi-Bilik Server

Selamat datang di **Kosan Kipli** (`48.193.45.144`), server VPS Ubuntu multi-tenant dengan isolasi kamar per-user (`chmod 711`), Nginx Reverse Proxy, PHP 8.5-FPM, Node.js v22 LTS, PM2, dan Fail2Ban.

Setiap penghuni memiliki **kunci kamar (user Linux) masing-masing**, bebas deploy aplikasi mandiri tanpa akses `sudo`, dan tidak bisa mengintip isi kamar penghuni lain.

---

## ⚠️ Penting: Belum Punya Kamar? Daftar Dulu!

Sebelum menjalankan perintah SSH, deploy, maupun setup CI/CD di bawah ini, **kamu wajib terdaftar sebagai penghuni kamar terlebih dahulu** dan sudah menerima **username**, **password awal**, serta **jatah port internal** dari Juragan Kos (`pli`).

### Cara Mengajukan Kamar Baru:
1. Buka halaman utama dashboard di **`http://48.193.45.144/`**.
2. Klik tombol **"📩 Ajukan Kamar Baru"** pada panel pendaftaran.
3. Isi nama panggilan, username yang diinginkan, stack proyek, dan port yang tersedia.
4. Klik **"📋 Salin Pesan Pengajuan"** lalu kirimkan formatnya ke WhatsApp atau Discord Juragan Kos (**`pli`**).
5. Tunggu konfirmasi dari Juragan Kos bahwa akun Linux dan jalur Nginx kamarmu sudah aktif sebelum lanjut ke langkah berikutnya.

---

## 📋 Daftar Kamar & Jatah Jalur Publik

| Kamar | Penghuni | Username | Jatah Port Internal | URL Web Statis / PHP | URL App Node.js (PM2) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Kantor Kos** | Admin (`pli`) | `pli` | `3000` (3001–3009) | `http://48.193.45.144/` | — |
| **Kamar 01** | Mas Nawawow | `nawawow` | `3010` (3011–3019) | `http://48.193.45.144/~nawawow/` | `http://48.193.45.144/nawawow/` |
| **Kamar 02** | Afad-Senpai | `afad` | `3020` (3021–3029) | `http://48.193.45.144/~afad/` | `http://48.193.45.144/afad/` |
| **Kamar 03** | Arza | `arza` | `3030` (3031–3039) | `http://48.193.45.144/~arza/` | `http://48.193.45.144/arza/` |
| **Kamar 04** | Deon | `deon` | `3040` (3041–3049) | `http://48.193.45.144/~deon/` | `http://48.193.45.144/deon/` |
| **Kamar 05** | *(Tersedia)* | `<username>` | `3050` (3051–3059) | `http://48.193.45.144/~<username>/` | `http://48.193.45.144/<username>/` |

---

## 🔑 1. Login ke Kamar Masing-Masing

*(Pastikan kamarmu sudah diaktifkan oleh Juragan Kos sebelum mencoba login).*

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

## 📜 Tata Tertib Kosan Kipli (House Rules)

> [!CAUTION]
> **<span style="color: #ef4444;">1. DILARANG KERAS MENJALANKAN `npm run build` LANGSUNG DI SERVER!</span>**
> **<span style="color: #ef4444;">Spesifikasi server hanya memiliki RAM 1 GiB. Menjalankan proses build (Next.js, Vite, React, dll.) secara langsung di terminal VM akan memicu *Out of Memory (OOM)* dan membuat server *hang* bagi seluruh penghuni. Wajib lakukan `npm run build` di komputer lokal atau lewat runner GitHub Actions!</span>**

2. **<span style="color: #ef4444;">Gunakan port sesuai jatah kamar</span>** — dilarang keras memakai port milik kamar lain atau membunuh proses PM2 penghuni lain.
3. **<span style="color: #ef4444;">Selalu jalankan `npm audit fix`</span>** di lokal sebelum mengirim proyek Node.js/React ke server.
4. **<span style="color: #ef4444;">Jangan ubah permission folder `/home/<username>`</span>** ke `755` atau `777` (biarkan tetap `711`) agar privasi file proyekmu terlindungi dari penghuni lain.