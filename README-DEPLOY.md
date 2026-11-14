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
# isi: PUBLIC_URL, SITE_ADDRESS, POSTGRES_PASSWORD, JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, AUTH_SECRET
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

## 3. Admin pertama
**Jangan** jalankan `db:seed` di produksi (menghapus semua data + akun demo ber-password publik).
```bash
read -rsp "Password admin: " ADMIN_PASSWORD; echo
docker compose run --rm -e ADMIN_EMAIL=admin@contoh.com -e ADMIN_NAME="Administrator" \
  -e ADMIN_PASSWORD="$ADMIN_PASSWORD" -v "$PWD/create-admin.mjs:/app/create-admin.mjs:ro" api node create-admin.mjs
```
Pengguna lain dibuat admin lewat menu Pengguna.

## 4. Impor inventaris.csv (sekali)
Api tidak dipublikasikan, jadi jalankan skrip di jaringan Docker yang sama:
```bash
docker run --rm --network lab_default -v "$PWD/../lab-management:/work" -w /work \
  -e API_URL=http://api:3000/api/v1 -e MIGRATE_EMAIL=admin@contoh.com -e MIGRATE_PASSWORD="$ADMIN_PASSWORD" \
  node:20-slim node scripts/migrate-inventory.mjs            # dry run
# jika sesuai, tambahkan --commit di akhir perintah
```

## Operasional
- **Update**: `git pull && docker compose up -d --build` (migrasi database otomatis saat api start).
- **Ubah PUBLIC_URL**: wajib `docker compose up -d --build web` (nilai QR tertanam saat build).
- **Backup**: `docker compose exec postgres pg_dump -U lab lab_equipment > backup.sql` **dan** volume `lab_uploads`
  (dokumen/sertifikat). Uji restore sebelum dibutuhkan.
- **IoT**: perangkat sensor mengirim ke `/api/v1/monitoring/readings` dengan `X-Sensor-Key`; jika dipakai, buka `ports` api di compose
  lalu aktifkan blok `location` di `nginx/lab.conf.example` (atau blok `handle` di `Caddyfile`); hanya jalur itu yang dibuka.

## Batasan yang perlu diketahui
- Satu replika `web` dan satu replika `api`. `api` menjalankan job terjadwal (ganda jika di-scale) dan `web` menyatukan
  refresh token hanya di dalam satu proses; memakai ulang refresh token lama >10 detik mencabut semua sesi pengguna.
  Untuk beberapa replika `web` perlu store bersama (mis. Redis) lebih dulu.
- Rate limit login backend (10 gagal / 15 menit) dihitung per IP klien: `web` meneruskan IP dari `X-Forwarded-For` yang
  diisi Caddy. Jika memakai proxy lain di depan Caddy (mis. Cloudflare), IP yang diteruskan adalah IP proxy itu sampai
  `trusted_proxies` Caddy dikonfigurasi.
- Impor massal dari UI dibatasi 1 MB JSON oleh backend (sekitar 2.000 baris).
