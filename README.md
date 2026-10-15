# Puskesmas Scraper

Paket instalasi aplikasi scraping SIMPUS, penyimpanan hasil, dan pengiriman ke API. Aplikasi Rust sudah dikompilasi menjadi binary di dalam image Docker. Python browser dan script scraping tersedia di dalam image untuk Playwright/Scrapling dan Loop Agent. Source Rust dan compiler Rust tidak disertakan.

## Prasyarat

- Docker Engine/Desktop dengan Docker Compose v2.
- Image Linux/amd64; untuk pilot gunakan RAM 4 GB, 2 vCPU, dan disk 40 GB sebagai titik awal.
- Koneksi keluar ke portal SIMPUS, API tujuan, dan provider LLM bila digunakan.
- VPS publik: domain yang Anda kuasai, DNS A menuju IP VPS, serta port 80/443 untuk HTTPS.

Jika Docker belum terpasang pada Ubuntu, ikuti [panduan resmi Docker](https://docs.docker.com/engine/install/ubuntu/).

## Unduh image

Unduh empat aset dari [GitHub Release](https://github.com/farrellaa/puskesmas-scraper/releases/tag/v0.2.0):

- `puskesmas-scraper-rust-hybrid.tar.gz`
- `puskesmas-scraper-deployment.zip`
- `SHA256SUMS`
- `release-manifest.json`

Di Linux, dari direktori unduhan:

```sh
sha256sum --check SHA256SUMS
docker image load --input puskesmas-scraper-rust-hybrid.tar.gz
unzip puskesmas-scraper-deployment.zip
cd puskesmas-scraper
```

Semua checksum harus menampilkan `OK`. Alternatif ZIP: clone repository ini untuk memperoleh file deployment, kemudian tetap unduh dan muat image dari release. Jangan jalankan build; repository ini tidak memuat source Rust.

## Instalasi VPS

```sh
sh deploy.sh prepare
nano .env.local
```

Pertahankan `POSTGRES_PASSWORD` acak yang sudah dibuat. Sesuaikan:

```dotenv
WEB_PORT=3003
COOKIE_SECURE=true
ALLOW_PRIVATE_DESTINATIONS=false
SCRAPER_DOMAIN=scraper.domainanda.com
SYSTEM_MAX_CONCURRENT_JOBS=1
APP_MEMORY_LIMIT=2g
```

Nama domain bebas: domain utama atau subdomain, tanpa kewajiban awalan tertentu. Isi hostname saja tanpa `https://`, port, atau path. DNS harus mengarah ke VPS. Jangan membagikan `.env.local`.

Buat administrator melalui terminal VPS:

```sh
sh deploy.sh init
```

Isi username pilihan Anda dan password minimal 8 karakter saat diminta; password tidak ditampilkan. Akun disimpan di database instalasi, tanpa akun atau password bawaan. Jika username sudah ada, perintah ini memperbarui password dan menjadikannya Admin.

```sh
sh deploy.sh up
sh deploy.sh status
sh deploy.sh doctor
```

Buka `https://` diikuti domain Anda. Caddy menangani HTTPS. Database dan port aplikasi terikat ke loopback; jangan membuka port 5435, 3003, atau 8111 ke internet.

Jika VPS sudah memakai reverse proxy pada port 80/443, jalankan `docker compose --env-file .env.local up -d --no-build --wait postgres app`, lalu arahkan reverse proxy HTTPS yang ada ke `http://127.0.0.1:3003`. Pertahankan `COOKIE_SECURE=true`.

## Menjalankan lokal

Linux/macOS: jalankan `sh deploy.sh prepare`, lalu atur `.env.local` menjadi `WEB_PORT=3002`, `COOKIE_SECURE=false`, dan `ALLOW_PRIVATE_DESTINATIONS=true` bila memakai dummy. Setelah `sh deploy.sh init`, jalankan `sh deploy.sh local`. Buka `http://localhost:3002`.

PowerShell, setelah image dimuat dan paket diekstrak:

```powershell
$dbPassword = [guid]::NewGuid().ToString('N') + [guid]::NewGuid().ToString('N')
@("POSTGRES_PASSWORD=$dbPassword", 'WEB_PORT=3002', 'COOKIE_SECURE=false', 'ALLOW_PRIVATE_DESTINATIONS=true') | Set-Content -Encoding utf8 .env.local
Remove-Variable dbPassword
docker compose --env-file .env.local up -d --wait postgres
docker compose --env-file .env.local run --rm app init --username admin
docker compose --env-file .env.local --profile test up -d --no-build --wait --wait-timeout 180
```

Perintah `init` meminta dan mengonfirmasi password secara tersembunyi. Jangan ulangi pembuatan `.env.local` pada instalasi yang sudah memiliki database. Dummy lokal tersedia di `http://localhost:8111`; dari aplikasi gunakan `http://dummy:8110/ingest`. Dummy ditujukan untuk data sintetis.

## Penggunaan

1. Login; Admin dapat membuat akun tambahan melalui Kelola akun.
2. Tambahkan Puskesmas dan kredensial SIMPUS yang diizinkan.
3. Jalankan scraping per tanggal; lihat progres pada Scrape Jobs dan data pada Hasil Scraping.
4. Tambahkan tujuan API, lalu kirim hasil melalui Sync ke API.
5. Atur Cron untuk scraping, POST, atau scraping lalu POST. LLM diperlukan hanya untuk Loop Agent.

Hasil, jobs, jadwal, dan tujuan API dipisahkan per akun. Puskesmas dan konfigurasi LLM digunakan bersama dalam satu instalasi. Akun lokal tidak otomatis tersedia di VPS.

## Status, backup, dan update

```sh
sh deploy.sh status
sh deploy.sh doctor
docker compose --env-file .env.local logs --tail 100 app
```

Backup harus memasangkan database, volume runtime/master key, dan `.env.local`. Buat direktori backup dengan akses terbatas terlebih dahulu, lalu saat tidak ada job aktif:

```sh
sh deploy.sh backup /lokasi/backup
```

Perintah menghentikan app sementara dan menyalakannya kembali setelah backup. Backup mengandung data rahasia; simpan terenkripsi di luar repository dan di luar VPS. Restore memerlukan database dan runtime/master key dari pasangan yang sama.

Untuk update: tunggu jobs selesai, backup, unduh aset versi baru, verifikasi checksum, muat image, perbarui file deployment dengan mempertahankan `.env.local`, lalu jalankan `sh deploy.sh up`. Migrasi database berjalan saat app mulai; tidak ada rollback database otomatis. Jangan menjalankan `docker compose down -v` selama data masih diperlukan.

Binary Rust tetap dapat dianalisis; distribusi binary tidak menjamin perlindungan mutlak terhadap rekayasa balik. Source Rust tetap dikelola pada repository pengembangan internal.


## Perubahan versi 0.2.0

- Pengambilan pasien mencakup data medis di luar CKG, mengikuti layanan dan data yang tersedia pada portal sumber.
- Export per pasien, JSON dengan nama field yang jelas, serta FHIR R4 JSON/NDJSON. Identifier dan referensi FHIR memakai UUID stabil dengan pemisahan per akun, Puskesmas, dan sumber. Data sumber dan kesenjangan pemetaan tetap dapat ditelusuri; export parsial tidak menyatakan semua persyaratan profil nasional sudah terpenuhi.
- Scrape ulang memperbarui record kunjungan pasien yang sama pada tanggal yang sama. Kunjungan, tanggal, atau akun berbeda tetap dipisahkan.
- Timeout scraping 90 menit per tanggal kunjungan. Rentang tanggal memakai timer baru tiap tanggal; timeout mempertahankan data tersimpan lalu melanjutkan tanggal berikutnya. Job tetap gagal bila ada tanggal gagal; rincian tercatat di log. Cancel menghentikan sisa rentang.
- Admin dapat membuka **Pengaturan scraper** (`/scraper-settings`) untuk default worker pasien dan tab per pasien. Default 5 x 3; masing-masing 1–8 dengan perkalian maksimal 32. Berlaku untuk job baru tanpa restart. Mulai dengan 2 x 2 pada perangkat terbatas dan sesuaikan berdasarkan pengamatan CPU/RAM serta respons portal.
- Deployment mendukung `SYSTEM_MAX_CONCURRENT_JOBS` (1–4, default 1) dan `APP_MEMORY_LIMIT` (default `2g`). Perubahan parameter deployment memerlukan pembuatan ulang container ketika job tidak aktif.
- Controller container memakai capability `KILL` agar dapat menghentikan worker Linux UID 10002 pada cancel/timeout. UID worker adalah akun proses, terpisah dari identifier pasien.
- Halaman record menyediakan JSON berindentasi, warna sintaks, buka/lipat struktur, tampilan teks, dan salin JSON tanpa mengubah angka atau identifier.

Release ini menyediakan image Linux/amd64 `puskesmas-scraper:0.2.0` dan alias `puskesmas-scraper:rust-hybrid`. Compose memakai tag versi tetap. Saat upgrade, perbarui juga Compose dan README dari paket deployment; jangan hanya memuat image baru dengan Compose lama. Pertahankan `.env.local`, database, volume runtime, dan master key.
