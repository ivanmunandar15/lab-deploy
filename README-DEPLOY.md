# Deploy: 4 container (postgres, api, web, caddy)

Dua aplikasi **tidak** digabung: `api` (Express + Prisma, port 3000) dan `web` (Next.js, port 3001) adalah container
terpisah. Hanya Caddy (80/443) yang terbuka ke luar; database dan api hanya bisa dijangkau lewat jaringan Docker.

```
Browser ──HTTPS──▶ Caddy ──▶ web (Next.js, BFF) ──http://api:3000──▶ api ──▶ postgres
```

## 0. Prasyarat yang HARUS dipenuhi dulu: migrasi Prisma
Backend tidak punya folder `prisma/migrations` (dan entrypoint api menolak start tanpa itu). Buat sekali di komputer
dengan Node + Docker, lalu simpan (commit) folder hasilnya:
```bash
cd lab-equipment-api
npm install
npm run db:up                         # Postgres lokal dari docker-compose backend
cp .env.example .env
npm run db:migrate -- --name init     # menghasilkan prisma/migrations/
```

## 1. Konfigurasi
```bash
cd lab-deploy
cp .env.example .env
# isi: PUBLIC_URL, SITE_ADDRESS, POSTGRES_PASSWORD, JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, AUTH_SECRET,
#      SEED_PASSWORD, SEED_ADMIN_EMAIL (lihat bagian 3)
# tiap secret:  openssl rand -hex 32   (JWT_* dan AUTH_SECRET harus berbeda-beda)
```
DNS `SITE_ADDRESS` harus mengarah ke server dan port 80/443 terbuka (HTTPS otomatis). HTTPS wajib: kamera pemindai QR
tidak jalan di HTTP biasa.

## 2. Jalankan
```bash
docker compose up -d --build
docker compose ps                     # tunggu semua healthy
docker compose logs -f api web
```

## 3. Data awal (otomatis atau manual)
Ada dua cara; hasilnya sama. Pilih salah satu.

### A. Manual (`SEED_ON_START=false`)
Setelah `docker compose up -d --build` dan semua container healthy (migrasi sudah jalan), isi data dengan satu perintah:
```bash
docker compose exec \
  -e SEED_MODE=if-empty \
  -e SEED_PASSWORD='<password awal, min. 12 karakter, huruf + angka>' \
  -e SEED_ADMIN_EMAIL='admin@contoh.com' \
  api node dist-seed/prisma/seed.js
```
- `if-empty`: hanya mengisi jika database kosong; kalau sudah berisi, perintah ini melaporkan "Seed dilewati" dan tidak mengubah apa pun.
- Dijalankan di dalam container `api` yang sedang hidup, jadi memakai database dan `NODE_ENV=production` yang sama; aturan produksi berlaku
  (password min. 12 karakter, tanpa password bawaan).
- Satu transaksi: gagal = tidak ada yang tersimpan; pesan error tampil di terminal.
- Mengisi ulang dari nol (menghapus SEMUA data, hanya untuk server uji): ganti `-e SEED_MODE=if-empty` dengan
  `-e SEED_MODE=reset -e SEED_FORCE=true`. **Jangan** dilakukan di server yang sudah dipakai.

### B. Otomatis (`SEED_ON_START=true`)
Dengan `SEED_ON_START=true` di `.env`, container `api` mengisi data awal **sekali**, saat start pertama ketika database masih kosong:
7 akun (admin + 4 teknisi + 2 SPV), 13 ruangan, 689 alat utama, 126 alat pendukung (dari `lab-equipment-api/prisma/data/*.csv`).
- Berjalan setelah migrasi, dalam satu transaksi (gagal = tidak ada yang tersimpan, container berhenti dan log menjelaskan sebabnya).
- Start berikutnya otomatis dilewati karena database sudah berisi. **Tidak pernah menghapus data.** Aman untuk update/restart.
- Isi di `.env`: `SEED_PASSWORD` (wajib, min. 12 karakter; satu password awal untuk semua akun) dan `SEED_ADMIN_EMAIL` (alamat nyata;
  kosong = `admin@lab.local`, alamat palsu).
- Lihat hasilnya: `docker compose logs api | grep -i seed`.
- Mengubah CSV sesudah database terisi **tidak** memuat ulang data (seed hanya jalan di database kosong).

> **Password awal harus diganti.** Satu `SEED_PASSWORD` dipakai semua akun seed. Tiap pengguna mengganti passwordnya sendiri di
> **menu profil (kanan atas) → Akun saya & ubah password**; setelah berhasil, sesinya berakhir dan ia masuk lagi dengan password baru.
> Admin dapat membuat password sementara untuk pengguna lain di **Users → ikon kunci** (tampil sekali; semua sesi pengguna itu diakhiri).
> Sarankan semua pengguna mengganti password pada login pertama, dan bagikan `SEED_PASSWORD` lewat saluran aman.

**Tanpa seed** (`SEED_ON_START=false`): buat admin pertama dengan skrip ini, lalu isi data lewat aplikasi/impor.
```bash
read -rsp "Password admin: " ADMIN_PASSWORD; echo
docker compose run --rm -e ADMIN_EMAIL=admin@contoh.com -e ADMIN_NAME="Administrator" \
  -e ADMIN_PASSWORD="$ADMIN_PASSWORD" -v "$PWD/create-admin.mjs:/app/create-admin.mjs:ro" api node create-admin.mjs
```
**Jangan** menjalankan `db:seed` mode reset di produksi (menghapus semua data).

## 4. Impor inventaris.csv lewat skrip (hanya jika TIDAK memakai seed)
Jika `SEED_ON_START=true`, langkah ini **tidak perlu** (data sudah ada; menjalankannya hanya akan melewati kode yang sama).
Api tidak dipublikasikan, jadi skrip dijalankan di jaringan Docker yang sama:
```bash
docker run --rm --network lab_default -v "$PWD/../lab-management:/work" -w /work \
  -e API_URL=http://api:3000/api/v1 -e MIGRATE_EMAIL=admin@contoh.com -e MIGRATE_PASSWORD="$ADMIN_PASSWORD" \
  node:20-slim node scripts/migrate-inventory.mjs            # dry run
# jika sesuai, tambahkan --commit di akhir perintah
```
Skrip ini hanya memetakan alat utama (belum Datang/kalibrasi terakhir dan alat pendukung seperti seed).

## Operasional
- **Update**: `git pull && docker compose up -d --build` (migrasi database otomatis saat api start).
- **Ubah PUBLIC_URL**: wajib `docker compose up -d --build web` (nilai QR tertanam saat build).
- **Backup**: `docker compose exec postgres pg_dump -U lab lab_equipment > backup.sql` **dan** volume `lab_uploads`
  (dokumen/sertifikat). Uji restore sebelum dibutuhkan.
- **IoT**: perangkat sensor mengirim ke `/api/v1/monitoring/readings` dengan `X-Sensor-Key`; aktifkan blok `handle`
  di `Caddyfile` jika dipakai (hanya jalur itu yang dibuka).

## Batasan yang perlu diketahui
- Satu replika `web` dan satu replika `api`. `api` menjalankan job terjadwal (ganda jika di-scale) dan `web` menyatukan
  refresh token hanya di dalam satu proses; memakai ulang refresh token lama >10 detik mencabut semua sesi pengguna.
  Untuk beberapa replika `web` perlu store bersama (mis. Redis) lebih dulu.
- Rate limit login backend (10 gagal / 15 menit) dihitung per IP klien: `web` meneruskan IP dari `X-Forwarded-For` yang
  diisi Caddy. Jika memakai proxy lain di depan Caddy (mis. Cloudflare), IP yang diteruskan adalah IP proxy itu sampai
  `trusted_proxies` Caddy dikonfigurasi.
- Impor massal dari UI dibatasi 1 MB JSON oleh backend (sekitar 2.000 baris).
