# Deploy: postgres, api, web (+ Caddy opsional)

Dua aplikasi **tidak** digabung: `api` (Express + Prisma, port 3000) dan `web` (Next.js, port 3001) adalah container
terpisah. Database dan api hanya bisa dijangkau lewat jaringan Docker. Siapa yang menerima trafik 80/443 ada dua pilihan:

| Situasi | Pakai | Perintah |
|---|---|---|
| VPS **sudah punya nginx** (mis. aplikasi Laravel lain) yang memegang port 80/443 | nginx yang ada sebagai reverse proxy | `docker compose up -d --build` |
| Server kosong, tanpa web server lain | Caddy (HTTPS otomatis) | `docker compose --profile caddy up -d --build` |

```
Browser ──HTTPS──▶ nginx host (atau Caddy) ──▶ web 127.0.0.1:3001 (Next.js, BFF) ──http://api:3000──▶ api ──▶ postgres
```
Caddy **tidak** ikut jalan tanpa `--profile caddy`, jadi tidak akan bentrok dengan nginx di port 80/443.

## VPS dengan nginx yang sudah ada (Laravel dll.)
Aplikasi Laravel Anda tidak diubah sama sekali. Yang ditambah hanya satu `server { }` baru dengan `server_name` sendiri.
1. **DNS**: buat A record subdomain (mis. `lab.contoh.com`) ke IP VPS.
2. **Cek apa yang memegang port**: `sudo ss -ltnp | grep -E ':(80|443|3001)\b'`. Port 3001 harus kosong; jika terpakai, ubah `WEB_PORT` di `.env`.
3. **Jalankan stack** (bagian 1-3 di bawah, tanpa `--profile caddy`). Container `web` hanya mendengarkan di `127.0.0.1:3001`
   (tidak terbuka ke internet, tidak perlu mengubah firewall).
4. **Pasang konfigurasi nginx**: salin `nginx/lab.conf.example` ke `/etc/nginx/sites-available/lab.conf`, ganti domain/port, lalu
   ```bash
   sudo ln -s /etc/nginx/sites-available/lab.conf /etc/nginx/sites-enabled/lab.conf
   sudo nginx -t && sudo systemctl reload nginx
   sudo certbot --nginx -d lab.contoh.com        # TLS + redirect otomatis
   ```
   HTTPS wajib: kamera pemindai QR tidak jalan di HTTP biasa.
5. Set `PUBLIC_URL=https://lab.contoh.com` di `.env`.

Catatan penting:
- `client_max_body_size 20m` di blok itu wajib; tanpa itu unggahan dokumen/sertifikat gagal (default nginx 1 MB).
- `X-Forwarded-For` diisi `$remote_addr` (menimpa). Jika ada Cloudflare/proxy lain di depan nginx, `$remote_addr` adalah IP proxy itu,
  sehingga rate limit login dihitung bersama untuk semua pengguna; atur modul `real_ip` nginx agar IP klien asli dipakai.
- Jika nginx Anda berjalan **di dalam Docker** (bukan terpasang di host), `127.0.0.1` di dalam container nginx bukan host:
  hubungkan container nginx ke jaringan `lab_default` dan pakai `proxy_pass http://web:3001;`.
- Aplikasi memilih nama cookie sesi (`__Secure-…`) dari `X-Forwarded-Proto`; di konfigurasi ini nilainya `$scheme` = `https` pada blok TLS.
  Jika TLS justru diterminasi proxy lain di depan nginx (nginx hanya mendengar HTTP), ganti menjadi `$http_x_forwarded_proto`.
- Backup dan restore database: lihat bagian Operasional.

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
`SITE_ADDRESS` hanya dipakai profil Caddy (DNS-nya harus mengarah ke server dan port 80/443 terbuka). Dengan nginx yang sudah ada,
abaikan dan ikuti bagian "VPS dengan nginx". HTTPS wajib di kedua cara: kamera pemindai QR tidak jalan di HTTP biasa.

## 2. Jalankan
```bash
docker compose up -d --build                    # nginx yang sudah ada; tambahkan --profile caddy hanya jika memakai Caddy
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

## Email pengingat kalibrasi (agar tidak spam)
- Default compose: `CALIBRATION_EMAIL_MODE=manual` → tidak ada email otomatis; kirim lewat **Log Email → Kirim email pengingat…** (pilih penerima dulu).
- Penerima ditentukan **aturan penanggung jawab** (menu *Penanggung Jawab*), bukan semua SPV. Seed mengisi aturan awal Ach. Effendi dan
  Muhamad Ridwan Maulana untuk `INV L-ACT 10`. Alat tanpa penanggung jawab tidak dikirimi email ke siapa pun.
- Urutan aman: set `EMAIL_ENABLED=true` + SMTP → uji lewat **Kirim email uji** → lengkapi aturan sampai *cakupan* penuh → coba pengiriman manual
  ke satu orang → baru (opsional) `CALIBRATION_EMAIL_MODE=auto`.
- Migrasi: model baru `equipment_responsibility` ikut migrasi `init`; bila migrasi pertama sudah dibuat, jalankan
  `npm run db:migrate -- --name add_equipment_responsibility` di lokal dan build ulang image.

## Operasional
- **Update**: `git pull && docker compose up -d --build` (migrasi database otomatis saat api start).
- **Ubah PUBLIC_URL**: wajib `docker compose up -d --build web` (nilai QR tertanam saat build).
- **Backup**: `docker compose exec postgres pg_dump -U lab lab_equipment > backup.sql` **dan** volume `lab_uploads`
  (dokumen/sertifikat). Uji restore sebelum dibutuhkan.
- **IoT**: perangkat sensor mengirim ke `/api/v1/monitoring/readings` dengan `X-Sensor-Key`; jika dipakai, buka `ports` api di compose
  lalu aktifkan blok `location` di `nginx/lab.conf.example` (atau blok `handle` di `Caddyfile`); hanya jalur itu yang dibuka.

## Email pengingat: kirim manual agar tidak spam
Compose ini memakai `CALIBRATION_EMAIL_MODE=manual` (default): job harian **tidak** mengirim email. Admin mengirimnya sendiri di
**Log Email → Kirim email pengingat…**: lihat pratinjau per penerima, centang yang akan dikirim (default tidak ada yang terpilih), konfirmasi.
Untuk kembali ke pengiriman otomatis harian 07:00 ke semua penerima, set `CALIBRATION_EMAIL_MODE=auto` di `.env` lalu
`docker compose up -d`. Email nyata hanya keluar jika `EMAIL_ENABLED=true` dan SMTP diisi; selain itu semua email hanya dicatat.

## Batasan yang perlu diketahui
- Satu replika `web` dan satu replika `api`. `api` menjalankan job terjadwal (ganda jika di-scale) dan `web` menyatukan
  refresh token hanya di dalam satu proses; memakai ulang refresh token lama >10 detik mencabut semua sesi pengguna.
  Untuk beberapa replika `web` perlu store bersama (mis. Redis) lebih dulu.
- Rate limit login backend (10 gagal / 15 menit) dihitung per IP klien: `web` meneruskan IP dari `X-Forwarded-For` yang
  diisi Caddy. Jika memakai proxy lain di depan Caddy (mis. Cloudflare), IP yang diteruskan adalah IP proxy itu sampai
  `trusted_proxies` Caddy dikonfigurasi.
- Impor massal dari UI dibatasi 1 MB JSON oleh backend (sekitar 2.000 baris).
