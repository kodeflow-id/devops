# Modul 5 — Proyek Akhir & Praktik Terpadu

## Ringkasan modul

Enam pelajaran terakhir menyatukan semuanya menjadi satu alur rilis nyata: dari repo GitHub ke `https://app.example.com`, otomatis, dengan rollback.

Yang dinilai bukan hafalan perintah, melainkan kemampuan mengambil keputusan saat ada yang gagal: membaca log, mempersempit lapisan masalah, dan memperbaiki tanpa merusak yang sudah jalan.

## Hasil pembelajaran

- Menyiapkan VPS produksi: user, akses, firewall, Docker, Nginx, HTTPS.
- Menerapkan aplikasi terkontainerisasi di balik reverse proxy dengan port backend tertutup dari internet.
- Membangun pipeline CI/CD yang men-deploy otomatis dan bisa dibalik.
- Menjalankan checklist go-live dan keamanan sebelum menyatakan rilis selesai.
- Mendiagnosis dan memperbaiki lima insiden umum: 502, restart loop, deploy "sukses" yang tidak berubah, cert kedaluwarsa, disk penuh.

---

## Pelajaran 5.1 — Menyiapkan server dan DNS

### Tujuan

Menyiapkan server dari kondisi kosong sampai siap menerima deploy.

### Langkah

0. Buat instance VPS Ubuntu 24.04 dan login pertama (lihat Modul 0). Domain juga sudah diarahkan ke IP server di Modul 0. Catat IP publik, user awal, dan pastikan console panel bisa diakses sebagai jalur darurat.

1. Update dan paket dasar.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git ufw nginx ca-certificates
```

2. User deploy, SSH key, hardening.

```bash
sudo adduser deploy
sudo mkdir -p /home/deploy/.ssh && sudo chmod 700 /home/deploy/.ssh
# tempel public key ke /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'CONF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
CONF
sudo sshd -t
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication)'   # bukti nilai efektif
sudo systemctl restart ssh           # Ubuntu 24.04 socket activation: sudo systemctl restart ssh.socket
```

3. Firewall.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status numbered
```

4. Docker dari repo resmi (lihat pelajaran 3.2).

5. DNS: buat record `A` untuk `app.example.com` ke IP server, dan `www` bila perlu.

```bash
dig +short app.example.com
```

6. Siapkan direktori aplikasi.

```bash
sudo mkdir -p /home/deploy/apps/app && sudo chown -R deploy:deploy /home/deploy/apps
su - deploy
mkdir -p ~/apps/app && cd ~/apps/app
```

### Verifikasi

```bash
ssh deploy@203.0.113.10 'whoami; sudo ufw status | head -8; docker --version; systemctl is-active nginx'
dig +short app.example.com
```

Semua harus sesuai: `deploy`, aturan firewall hanya 22/80/443, Docker versi terbaru, Nginx aktif, DNS menunjuk IP server.

### Kalau gagal

| Gejala | Periksa |
| --- | --- |
| Tidak bisa SSH setelah hardening | Console VPS; cek `journalctl -u ssh`, izin `~/.ssh` 700 dan key 600 |
| DNS belum mengarah | Panel DNS, TTL, `dig @1.1.1.1 app.example.com` |
| `docker: permission denied` | Grup `docker`, login ulang |

### Latihan

Dokumentasikan langkah 1–6 dalam `runbook-server.md` supaya server kedua bisa disiapkan tanpa berpikir ulang.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Siapkan server produksi dari nol: user deploy, SSH key-only, ufw 22/80/443, Docker, Nginx, dan DNS untuk domain proyek.

**Siapkan:** Isi checklist 5.1 dan kumpulkan bukti tiap baris.

**Bawa bukti:** Checklist terisi + output verifikasi (`ssh`, `ufw status`, `docker --version`, `dig`).

Dibahas di awal **Pelajaran 5.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 5.2 — Menyiapkan aplikasi dan image

### Tujuan

Menghasilkan image yang bisa dijalankan konsisten di server.

### Langkah

1. Repo berisi minimal: `Dockerfile`, `compose.yaml`, `.env.example`, `.dockerignore`, `.gitignore`, endpoint `/health`.

2. Build lokal dan uji sebelum menyentuh server.

```bash
docker build -t app:local .
docker run --rm -p 127.0.0.1:3000:3000 --env-file .env app:local
curl -s http://127.0.0.1:3000/health
```

3. Pastikan image tidak memuat rahasia.

```bash
docker history --no-trunc app:local | head -20
docker run --rm --entrypoint env app:local | grep -iE 'pass|secret|token' || echo "bersih"
```

4. `compose.yaml` untuk server (memakai image dari registry, bukan `build`).

```yaml
services:
  api:
    image: ${IMAGE:-ghcr.io/USER/REPO:latest}
    restart: unless-stopped
    env_file: .env
    ports:
      - "127.0.0.1:3000:3000"
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 3s
      retries: 3
```

5. Buat `.env` di server (mode 600), jangan di repo.

### Verifikasi

```bash
docker compose config --quiet && echo "compose valid"
docker compose up -d && sleep 5 && docker compose ps
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/health
```

### Kalau gagal

| Gejala | Periksa |
| --- | --- |
| Container exit 1 | `docker compose logs api`; jalankan perintah yang sama manual |
| `/health` 404 | Endpoint belum ada di aplikasi, atau path berbeda |
| Image besar | Multi-stage, base slim/alpine, `.dockerignore` |

### Latihan

Ukur ukuran image dan waktu start. Target: image < 250 MB, sehat dalam 10 detik.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Siapkan image aplikasi proyek dengan endpoint `/health`, lalu jalankan lewat `compose.yaml` versi server (image dari registry).

**Siapkan:** Uji image di lokal: pastikan tidak ada rahasia yang ikut ke dalamnya.

**Bawa bukti:** `curl` 200 pada `/health` + ukuran image + output pemeriksaan rahasia.

Dibahas di awal **Pelajaran 5.3** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 5.3 — Nginx reverse proxy dan HTTPS

### Tujuan

Membuka aplikasi ke internet lewat satu pintu dengan TLS.

### Langkah

1. Server block HTTP untuk challenge + redirect.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;
    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 301 https://$host$request_uri; }
}
```

2. Terbitkan sertifikat.

```bash
sudo certbot --nginx -d app.example.com --redirect --agree-tos -m admin@example.com --no-eff-email
# `-d www.app.example.com` hanya jika vhost www sudah ada dan DNS-nya mengarah ke server ini
```

3. Tambahkan reverse proxy ke `127.0.0.1:3000` beserta header dan WebSocket (lihat pelajaran 2.6), plus `location = /health` yang dialihkan tanpa log.

4. Header keamanan dan ukuran upload.

```nginx
include /etc/nginx/snippets/security-headers.conf;
client_max_body_size 25m;
```

5. Terapkan dan uji.

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -sI https://app.example.com | head -3
curl -s -o /dev/null -w '%{http_code}\n' http://app.example.com     # 301
curl -s https://app.example.com/health
```

### Verifikasi

- `https://app.example.com` membalas 200 dengan sertifikat valid.
- `http://` membalas 301 ke HTTPS.
- `nc -zv 203.0.113.10 3000` dari laptop **gagal** (backend tertutup).
- `sudo certbot renew --dry-run` sukses.

### Kalau gagal

| Gejala | Periksa |
| --- | --- |
| 502 di browser | `curl -sI http://127.0.0.1:3000/health`, `docker compose ps`, `tail /var/log/nginx/error.log` |
| Cert gagal | Port 80 terbuka? DNS benar? `server_name` cocok? Rate limit Let's Encrypt? |
| Redirect loop | Redirect HTTPS→HTTPS tertinggal di block SSL |

### Latihan

Tambahkan rate limit 10r/s `burst=20` untuk `/api/` dan buktikan dengan pengujian beruntun.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Pasang reverse proxy + HTTPS untuk domain proyek, lalu tutup port backend dari internet.

**Siapkan:** Uji dari luar: HTTPS 200, HTTP 301, port 3000 gagal.

**Bawa bukti:** Tiga hasil uji tersebut + `certbot renew --dry-run` + `nginx -t` bersih.

Dibahas di awal **Pelajaran 5.4** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 5.4 — Pipeline CI/CD

### Tujuan

Membuat deploy berjalan otomatis dari `git push`.

### Langkah

1. `.github/workflows/ci.yml` — test + lint, berjalan pada PR dan push ke `main`.
2. `.github/workflows/release.yml` — build & push image ke GHCR dengan tag `sha-<commit>` dan `latest`.
3. `.github/workflows/deploy.yml` — lewat `workflow_run` dari release, memanggil `deploy.sh` di server lewat SSH.
4. Secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY`, `GHCR_TOKEN` (bila package private), opsional `DISCORD_WEBHOOK`.
5. Environment `production` dengan required reviewer bila ingin gerbang manual.
6. `~/apps/app/deploy.sh` dengan health check dan rollback otomatis (lihat pelajaran 4.7).

### Verifikasi

```bash
git commit --allow-empty -m "chore: uji pipeline" && git push
# di tab Actions: tiga workflow hijau berurutan
ssh deploy@203.0.113.10 'cd ~/apps/app && cat .env && docker compose ps'
curl -s https://app.example.com/health
```

Tag image di server harus sama dengan commit yang baru saja di-push.

### Kalau gagal

| Gejala | Periksa |
| --- | --- |
| Workflow tidak jalan | Path `.github/workflows/`, branch, apakah workflow ada di default branch |
| `Host key verification failed` | `fingerprint`/`known_hosts` pada action SSH |
| Pull image `denied` | Scope token `read:packages`, atau package private tanpa login |
| Deploy jalan tapi versi lama | `.env` tidak diperbarui, atau `docker compose up -d` tidak dijalankan |

### Latihan

Picu deploy yang gagal (mis. image tag tidak ada), tunjukkan rollback otomatis di log, lalu perbaiki.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Rangkai tiga workflow (CI, release, deploy) dan picu satu deploy lewat commit kosong.

**Siapkan:** Cocokkan tag image yang berjalan di server dengan commit yang dipush.

**Bawa bukti:** Tiga run hijau + `docker inspect` tag image + `/health` publik 200.

Dibahas di awal **Pelajaran 5.5** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 5.5 — Go-live dan observability

### Tujuan

Menjalankan rilis dengan bukti, bukan dengan harapan.

### Checklist go-live

- [ ] DNS mengarah ke server, `dig +short` sesuai.
- [ ] Nginx lolos `nginx -t`, service aktif dan `enabled`.
- [ ] Sertifikat valid > 30 hari, `certbot renew --dry-run` sukses, timer aktif.
- [ ] Firewall hanya membuka 22/80/443; port backend tertutup dari internet — dibuktikan dengan `nc -zv <IP> 3000` dari luar, bukan hanya dari `ufw status` (Docker melewati `ufw`).
- [ ] `docker compose ps` menunjukkan semua service `running`/`healthy`.
- [ ] Health check lokal dan publik membalas 200.
- [ ] Log akses dan error Nginx terisi; `docker compose logs` bersih dari error berulang.
- [ ] `.env` mode 600, tidak ada rahasia di repo maupun image.
- [ ] Backup database terjadwal dan **sudah pernah dipulihkan** sekali.
- [ ] Rollback terbukti berjalan (diuji di staging atau saat latihan).
- [ ] Deploy hanya bisa dipicu dari branch `main`.

### Checklist keamanan dasar

- [ ] SSH key-only, `PermitRootLogin no`, user deploy tanpa sudo penuh bila memungkinkan.
- [ ] Tidak ada service yang listen di `0.0.0.0` selain 22/80/443.
- [ ] Image memakai tag versi, bukan `latest` saja, dan berjalan sebagai non-root.
- [ ] Secret hanya di GitHub Secrets dan `.env` server.
- [ ] Login GHCR memakai token scope minimum.
- [ ] Log rotation Docker aktif (`/etc/docker/daemon.json`) dan ada pemantauan disk.

### Observability minimum

```bash
# ringkasan cepat
docker compose ps
docker stats --no-stream
df -h /; du -sh /var/lib/docker /var/log
sudo tail -20 /var/log/nginx/error.log
journalctl -u nginx --since "1 hour ago" | tail -20
```

Pemeriksaan otomatis sederhana (systemd timer atau cron harian):

```bash
#!/usr/bin/env bash
set -euo pipefail
code=$(curl -s -o /dev/null -w '%{http_code}' https://app.example.com/health || echo 000)
[ "$code" = "200" ] || echo "ALERT: health $code $(date -Is)" >> ~/alerts.log
disk=$(df --output=pcent / | tail -1 | tr -dc '0-9')
[ "$disk" -lt 85 ] || echo "ALERT: disk ${disk}% $(date -Is)" >> ~/alerts.log
```

### Verifikasi

Tunjukkan keluaran nyata untuk setiap item checklist, bukan centang tanpa bukti.

### Kalau gagal

| Gejala | Periksa |
| --- | --- |
| Disk penuh | `du -sh /var/lib/docker/*`, `docker system df`, log rotation |
| Cert mendekati kedaluwarsa | `systemctl list-timers \| grep certbot`, `certbot renew --dry-run` |
| Backup belum pernah diuji | Pulihkan ke database sementara dan bandingkan jumlah baris |

### Latihan

Buat satu skrip pemeriksaan kesehatan + disk yang berjalan tiap 15 menit lewat systemd timer.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Jalankan checklist go-live, buat backup database dan **uji restore**-nya, lalu pasang pemeriksaan kesehatan + disk berkala.

**Siapkan:** Siapkan bahan demo: arsitektur, alur deploy, dan rencana rollback.

**Bawa bukti:** Hasil restore (jumlah baris sebelum/sesudah) + log skrip pemeriksaan + checklist keamanan terisi.

Dibahas di awal **Pelajaran 5.6** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 5.6 — Simulasi insiden dan demo

### Tujuan

Berlatih diagnosis di bawah tekanan, dengan alur yang jelas.

### Alur diagnosis umum

```
1. Apa yang dilihat pengguna? (kode, pesan, sejak kapan)
2. Lapisan mana?        DNS -> firewall -> port -> proses -> Nginx -> aplikasi -> database
3. Buktikan per lapisan dengan satu perintah, jangan menebak
4. Ubah satu hal, uji lagi, catat hasilnya
5. Setelah pulih: tulis penyebab dan pencegahan
```

### Insiden 1 — 502 Bad Gateway

```bash
curl -sI http://127.0.0.1:3000/health        # aplikasi hidup?
docker compose ps                            # container running?
docker compose logs --tail 50 api            # error aplikasi?
sudo tail -50 /var/log/nginx/error.log        # pesan Nginx: connect() failed / upstream
```

Penyebab umum: container mati, port berubah, aplikasi crash saat start, `upstream` menunjuk port salah.

### Insiden 2 — Container restart loop

```bash
docker compose ps                 # Restarting?
docker inspect -f '{{.State.ExitCode}} {{.State.Error}}' $(docker compose ps -q api)
docker compose logs --tail 100 api
docker compose config | grep -A5 environment
```

Penyebab umum: env wajib kosong, migrasi database gagal, port bentrok, izin volume.

### Insiden 3 — Deploy sukses tapi aplikasi tidak berubah

```bash
ssh deploy@IP 'cd ~/apps/app && cat .env && docker compose ps'
docker inspect -f '{{.Config.Image}}' $(docker compose ps -q api)
docker compose pull && docker compose up -d --force-recreate
```

Penyebab umum: tag `latest` yang di-cache, `.env` tidak diperbarui, `up -d` tidak dijalankan setelah pull.

### Insiden 4 — Sertifikat kedaluwarsa

```bash
echo | openssl s_client -connect app.example.com:443 -servername app.example.com 2>/dev/null | openssl x509 -noout -dates
systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot renew --force-renewal && sudo systemctl reload nginx
```

Penyebab umum: timer mati, port 80 tertutup (validasi HTTP-01 gagal), DNS berubah.

### Insiden 5 — Disk penuh

```bash
df -h /
du -sh /var/lib/docker /var/log /home/* 2>/dev/null | sort -h
docker system df
journalctl --disk-usage
sudo journalctl --vacuum-time=7d
docker image prune -f          # aman: hanya image tanpa tag
```

Peringatan: jangan langsung `docker system prune -a --volumes`; itu bisa menghapus data database.

### Verifikasi (demo akhir)

Presentasikan 8 menit:

1. Arsitektur: repo → Actions → GHCR → VPS → Docker → Nginx → pengguna.
2. Demo: satu commit → deploy otomatis → situs diperbarui.
3. Demo: rollback ke versi sebelumnya.
4. Demo: satu insiden disimulasikan (matikan container atau rusak health check) dan didiagnosis di depan penguji, sambil menyebut perintah dan alasannya.

### Ide proyek alternatif

| Proyek | Lingkup | Kriteria penilaian |
| --- | --- | --- |
| API + Postgres di balik Nginx | Satu API, satu database bervolume, migrasi saat deploy | Backup/restore terbukti, health check menyentuh database, rollback jalan |
| Situs statis dengan pipeline build | Build situs, unggah artefak, Nginx menyajikan hasil build | Cache CI efektif, deploy < 2 menit, HTTPS + redirect, tanpa downtime |
| Multi-container dengan worker | Web + worker antrean + Redis + Postgres | Worker hidup setelah reboot, log terpisah, scaling worker terbukti, tanpa port internal terbuka |

### Cek pemahaman

1. Apa tiga perintah pertama saat menghadapi 502? — `curl` ke backend langsung, `docker compose ps`, `docker compose logs`; lanjut `/var/log/nginx/error.log`.
2. Kenapa `docker system prune -a --volumes` berbahaya di produksi? — Menghapus volume yang tidak sedang dipakai, termasuk data database.
3. Bagaimana membuktikan deploy benar-benar menerapkan versi baru? — Bandingkan tag image di `.env`/`docker inspect` dengan commit yang di-push, lalu uji `/health`.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Siapkan demo 8 menit: arsitektur, deploy otomatis, rollback, dan satu insiden yang kamu diagnosis sendiri.

**Siapkan:** Tulis laporan insiden satu halaman: gejala, dugaan, tindakan, hasil, pencegahan.

**Bawa bukti:** Demo berjalan + laporan insiden + proyek akhir dapat diakses lewat HTTPS.

Dibawa ke **sesi penutup / demo akhir** dan menjadi bagian penilaian proyek akhir.

## Referensi

- Nginx reverse proxy — https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- Docker Compose production — https://docs.docker.com/compose/how-tos/production/
- GitHub Actions deployments — https://docs.github.com/en/actions/deployment
- Certbot — https://eff-certbot.readthedocs.io/
- Twelve-Factor App (konfigurasi & rilis) — https://12factor.net/
