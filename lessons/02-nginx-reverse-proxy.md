# Modul 2 — Nginx & Reverse Proxy

## Ringkasan modul

Modul ini mengubah server yang cuma bisa SSH menjadi server yang melayani aplikasi web: satu pintu masuk HTTPS di depan aplikasi yang hanya mendengarkan di `127.0.0.1`.

Asumsi: Ubuntu 24.04 LTS, Nginx dari repository Ubuntu (1.24.x), aplikasi contoh mendengarkan di `127.0.0.1:3000`, domain `example.com` dan `app.example.com` mengarah ke `203.0.113.10`.
> **Peringatan keamanan (berlaku di seluruh modul):**
> 1. Buka hanya port 80 dan 443 ke internet (`sudo ufw allow 'Nginx Full'` + `sudo ufw allow OpenSSH`). Port backend seperti 3000 **jangan pernah** dibuka ke publik.
> 2. Backend cukup listen di `127.0.0.1:3000`, bukan `0.0.0.0:3000`. Kalau backend terekspos, penyerang bisa melewati Nginx sepenuhnya — TLS, rate limit, dan header keamanan tidak berlaku.
> 3. Jangan pernah `chmod 777` direktori web. Pakai owner `www-data`, mode `755` untuk direktori dan `644` untuk file.
> 4. Jangan taruh kredensial di file config Nginx — isi `/etc/nginx` mudah terbaca.

## Hasil pembelajaran

- Menulis server block yang lolos `nginx -t` dan menerapkan perubahan dengan reload tanpa downtime.
- Menjelaskan urutan prioritas `location` dan memilih jenis pencocokan yang tepat.
- Mengonfigurasi reverse proxy lengkap dengan header `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, dan dukungan WebSocket.
- Mengamankan satu domain dengan HTTPS Let's Encrypt beserta perpanjangan otomatis.
- Mendiagnosis 502, 504, 403, 413, dan kegagalan cert dari log, bukan dari tebakan.
- Menambahkan rate limit, basic auth, dan load balancing sederhana.

---

## Pelajaran 2.1 — HTTP dan cara kerja web server

### Tujuan

Membaca request/response HTTP dan tahu lapisan mana yang gagal saat situs tidak bisa diakses.

### Konsep

Satu permintaan HTTP berisi baris permintaan, header, dan opsional body. Balasannya berisi status code, header, dan body.

Status code yang wajib dikenal:

| Kode | Arti | Sering muncul karena |
| --- | --- | --- |
| 200 | OK | Normal |
| 301/302 | Redirect permanen/sementara | Pindah domain, HTTP→HTTPS |
| 403 | Forbidden | Izin file, `deny`, basic auth ditolak |
| 404 | Not found | Path salah, `root` salah |
| 413 | Payload too large | `client_max_body_size` terlalu kecil |
| 499 | Client closed request | Klien menyerah (khas Nginx) |
| 500 | Internal server error | Bug aplikasi |
| 502 | Bad gateway | Backend mati atau tidak bisa dihubungi |
| 504 | Gateway timeout | Backend lambat atau menggantung |

Header penting: `Host` (domain yang diminta), `X-Real-IP` dan `X-Forwarded-For` (IP asli klien), `X-Forwarded-Proto` (http atau https). Tanpa header ini, aplikasi mengira semua request datang dari `127.0.0.1` dan membuat tautan `http://` di halaman HTTPS.

TLS adalah lapisan enkripsi di bawah HTTP. Port 443 = HTTP di dalam TLS. Sertifikat mengikat nama domain ke kunci publik.

### Praktik

1. Lihat request mentah dengan `curl -v`.

```bash
curl -v http://example.com 2>&1 | head -30
curl -sI https://example.com          # hanya header respons
curl -s -o /dev/null -w 'code=%{http_code} time=%{time_total}s\n' https://example.com
```

2. Kirim header sendiri.

```bash
curl -s -H 'Host: app.example.com' http://203.0.113.10 -I
```

3. Baca urutan lapisan saat debug: DNS → firewall → port → proses → Nginx → aplikasi. Setiap lapisan bisa dibuktikan dengan satu perintah.

```bash
dig +short app.example.com      # DNS
sudo ufw status numbered        # firewall
sudo ss -tulpn | grep -E ':80|:443|:3000'   # proses
sudo nginx -t && systemctl status nginx --no-pager
curl -sI http://127.0.0.1:3000  # aplikasi langsung
```

### Kalau error

| Gejala | Lapisan | Solusi |
| --- | --- | --- |
| `Connection refused` | Tidak ada proses mendengarkan | Cek `ss -tulpn`, jalankan aplikasinya |
| `Connection timed out` | Firewall / security group | Buka 80/443 di `ufw` dan panel VPS |
| `Could not resolve host` | DNS | Perbaiki record A |

### Latihan

1. Tampilkan kode status dan waktu respons `https://example.com` dalam satu perintah.
2. Tentukan lapisan mana yang gagal untuk `curl: (7) Failed to connect to app.example.com port 443`.

**Kriteria selesai:** kamu bisa menyebut perintah pembuktian untuk tiap lapisan tanpa melihat catatan.

### Cek pemahaman

1. Beda 502 dan 504? — 502 backend tidak bisa dihubungi/balasannya tidak valid; 504 backend tidak menjawab dalam batas waktu.
2. Kenapa `X-Forwarded-Proto` penting? — Aplikasi tahu skema asli (https) sehingga tidak membuat tautan http.
3. Apa yang diuji `curl -I`? — Hanya header respons, tanpa mengunduh body.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat tabel 8 status code (200, 301, 403, 404, 413, 499, 502, 504): arti, penyebab paling umum, dan perintah untuk memeriksanya.

**Siapkan:** Jalankan `curl -v` ke domainmu dan simpan seluruh header request/response.

**Bawa bukti:** Tabel status code + output `curl -v` dengan dua header yang kamu tandai penting.

Dibahas di awal **Pelajaran 2.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.2 — Instalasi dan struktur konfigurasi Nginx

### Tujuan

Memahami di mana konfigurasi berada dan siklus perubahan yang aman.

### Konsep

```
/etc/nginx/
├── nginx.conf              # konfigurasi utama, berisi http { ... }
├── conf.d/*.conf           # potongan konfigurasi tambahan
├── sites-available/*       # definisi server block (tidak dibaca langsung)
├── sites-enabled/*         # symlink ke sites-available yang aktif
├── snippets/               # potongan yang bisa di-include
└── mime.types
```

Nginx membaca `sites-enabled/*` karena ada baris `include /etc/nginx/sites-enabled/*;` di dalam blok `http` pada `nginx.conf`. Menonaktifkan situs = hapus symlink-nya, bukan hapus file.

**Aturan kerja:** setiap perubahan → `sudo nginx -t` → kalau OK → `sudo systemctl reload nginx`. Reload menerapkan konfigurasi baru tanpa memutus koneksi yang sedang jalan; restart memutus koneksi.

### Praktik

1. Pasang dan periksa.

```bash
sudo apt update
sudo apt install -y nginx
nginx -v
systemctl status nginx --no-pager
ls -l /etc/nginx/
```

2. Halaman default.

```bash
curl -I http://127.0.0.1
ls -l /var/www/html
```

3. Baca konfigurasi utama.

```bash
grep -vE '^\s*(#|$)' /etc/nginx/nginx.conf | head -40
```

4. Siklus perubahan yang benar.

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo nano /etc/nginx/sites-available/demo
sudo ln -s /etc/nginx/sites-available/demo /etc/nginx/sites-enabled/demo
sudo nginx -t
sudo systemctl reload nginx
```

5. Log.

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

6. Nonaktifkan situs default supaya tidak menabrak `server_name`.

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `nginx: [emerg] ... failed` | Sintaks salah, titik koma hilang | Baca baris yang ditunjuk `nginx -t` |
| Perubahan tidak berefek | File tidak ada di `sites-enabled` atau lupa reload | `ls -l /etc/nginx/sites-enabled/`, lalu `systemctl reload nginx` |
| `Address already in use` | Proses lain memakai port 80 | `sudo ss -tulpn \| grep :80` |

### Latihan

1. Buat file `sites-available/demo` minimal, aktifkan dengan symlink, reload, dan akses lewat `curl -H 'Host: demo.local' http://127.0.0.1`.
2. Sengaja hapus satu titik koma, jalankan `nginx -t`, catat pesan errornya, lalu perbaiki.

**Kriteria selesai:** `nginx -t` bersih dan kamu tahu baris mana yang dibaca Nginx saat mencari server block aktif.

### Cek pemahaman

1. Kenapa `sites-available` dan `sites-enabled` dipisah? — Menyimpan konfigurasi tapi hanya mengaktifkan yang diinginkan lewat symlink.
2. Beda reload dan restart? — Reload tanpa memutus koneksi; restart menghentikan lalu menyalakan proses.
3. Urutan yang benar setelah edit konfigurasi? — `nginx -t`, lalu reload.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Pasang Nginx, nonaktifkan situs default, dan buat satu file di `sites-available` yang belum diaktifkan.

**Siapkan:** Baca `nginx.conf` dan temukan baris `include` yang membuat `sites-enabled` terbaca.

**Bawa bukti:** `sudo nginx -t` + `systemctl status nginx` + baris `include` yang kamu temukan.

Dibahas di awal **Pelajaran 2.3** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.3 — Situs statis pertama

### Tujuan

Menyajikan file statis dengan `root`, `index`, dan `try_files`.

### Konsep

`root /var/www/site;` berarti Nginx mencari `/var/www/site` + path URI. Request `/logo.png` → `/var/www/site/logo.png`.

`index index.html;` menentukan file yang dikirim saat URI berakhir dengan `/`.

`try_files $uri $uri/ =404;` adalah pola standar: coba file, coba direktori, kalau tidak ada balas 404 eksplisit. Untuk aplikasi single-page, versi SPA-nya: `try_files $uri /index.html;`.

Nginx berjalan sebagai user `www-data`. File yang tidak bisa dibaca `www-data` menghasilkan 403.

### Praktik

1. Siapkan direktori dan file.

```bash
sudo mkdir -p /var/www/demo
sudo chown -R deploy:deploy /var/www/demo
cat > /var/www/demo/index.html <<'HTML'
<!doctype html>
<meta charset="utf-8">
<title>Demo</title>
<h1>Server ini hidup</h1>
HTML
```

2. Server block.

```bash
sudo nano /etc/nginx/sites-available/demo
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name demo.example.com;

    root /var/www/demo;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

3. Aktifkan dan uji.

```bash
sudo ln -sf /etc/nginx/sites-available/demo /etc/nginx/sites-enabled/demo
sudo nginx -t && sudo systemctl reload nginx
curl -I -H 'Host: demo.example.com' http://127.0.0.1
curl -s -H 'Host: demo.example.com' http://127.0.0.1 | head -3
```

4. Verifikasi kepemilikan dan izin.

```bash
namei -l /var/www/demo/index.html
sudo -u www-data cat /var/www/demo/index.html >/dev/null && echo "www-data bisa baca"
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| 403 Forbidden | Direktori induk tidak bisa dilewati `www-data` | `chmod 755` pada direktori, `namei -l` untuk memeriksa |
| 404 pada file yang ada | `root` salah / `try_files` menunjuk path lain | Bandingkan URI dengan path nyata |
| Halaman default Apache/Nginx muncul | `default_server` masih aktif | `rm /etc/nginx/sites-enabled/default` |

### Latihan

1. Tambahkan `style.css`, pastikan termuat, dan buktikan dari `access.log`.
2. Ubah `try_files` ke mode SPA (`$uri /index.html`) dan uji request `/halaman-tidak-ada` — harus tetap 200.

**Kriteria selesai:** halaman statis tampil lewat `curl` dengan header `Host` yang benar.

### Cek pemahaman

1. Apa arti `root /var/www/demo` untuk URI `/a/b.txt`? — File diambil dari `/var/www/demo/a/b.txt`.
2. Kenapa 403 sering soal izin direktori, bukan file? — Nginx butuh izin `x` pada setiap direktori di sepanjang path.
3. Kapan `try_files $uri /index.html` dipakai? — Untuk aplikasi single-page yang menangani routing di sisi klien.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Sajikan satu situs statis dengan CSS sendiri dan halaman 404 kustom.

**Siapkan:** Siapkan daftar 3 subdomain yang ingin kamu buat (aplikasi, staging, admin).

**Bawa bukti:** `curl -I` untuk 200 dan 404 + output `namei -l` pada file yang disajikan.

Dibahas di awal **Pelajaran 2.4** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.4 — Server block dan domain

### Tujuan

Melayani beberapa domain di satu server dan mengarahkan HTTP ke HTTPS.

### Konsep

`server_name` mencocokkan header `Host`. Nginx memilih server block paling cocok; kalau tidak ada yang cocok, dipakai block dengan `default_server`.

Pola redirect yang benar: satu block `listen 80` untuk redirect, satu block `listen 443 ssl` untuk melayani.

### Praktik

1. Beberapa domain, satu server.

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/company;
    index index.html;
}

server {
    listen 80;
    server_name app.example.com;
    root /var/www/app-public;
    index index.html;
}
```

2. Tangkap semua nama yang tidak dikenal.

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;    # tutup koneksi tanpa balasan
}
```

3. Redirect HTTP ke HTTPS sekaligus menormalkan `www`.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}
```

Jangan membuat blok `listen 80` **kedua** dengan `server_name www.example.com`; Nginx akan mencatat `conflicting server name` dan salah satu blok diabaikan.

4. Kalau `www` tetap ingin dipertahankan tetapi selalu dialihkan ke non-`www` di HTTPS, tambahkan satu blok 443 khusus:

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}
```

Sertifikat harus mencakup `www.example.com` (`-d www.example.com` saat menerbitkan), kalau tidak browser memperingatkan nama tidak cocok.

5. Uji pemilihan block.

```bash
curl -sI -H 'Host: app.example.com' http://127.0.0.1 | head -1
curl -sI -H 'Host: tidak-dikenal.local' http://127.0.0.1 | head -1
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `conflicting server name` di error log | `server_name` sama di dua block | Hapus/ubah salah satu |
| Loop redirect | HTTPS block juga me-redirect ke HTTPS | Pastikan redirect hanya di block port 80 |
| Domain lain ikut melayani | Semua masuk `default_server` | Tambahkan block `server_name` yang tepat |

### Latihan

1. Buat dua domain berbeda dilayani satu Nginx, buktikan dengan `curl` memakai header `Host`.
2. Tambahkan redirect permanen dari `www` ke non-`www`, dan buktikan kode 301.

**Kriteria selesai:** request ke nama yang tidak terdaftar tidak membocorkan situs mana pun.

### Cek pemahaman

1. Apa fungsi `default_server`? — Block yang dipakai saat tidak ada `server_name` yang cocok.
2. Kenapa `return 301` lebih baik daripada `rewrite` untuk redirect domain? — Lebih sederhana, lebih cepat, dan tidak melibatkan evaluasi regex.
3. Di block mana redirect HTTP→HTTPS harus diletakkan? — Di block `listen 80`, bukan di block SSL.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat dua server block berbeda + satu `default_server` yang membalas 444 + redirect `www` ke non-`www`.

**Siapkan:** Sebelum menguji, tulis tebakanmu: blok mana yang menangani tiap nama?

**Bawa bukti:** Tiga output `curl -sI -H 'Host: ...'` dan bandingkan dengan tebakanmu.

Dibahas di awal **Pelajaran 2.5** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.5 — `location` dan routing

### Tujuan

Mengarahkan URI ke tujuan yang tepat, dan tahu urutan pencocokan Nginx.

### Konsep

Urutan pencocokan `location`:

1. `location = /path` — cocok persis, prioritas tertinggi.
2. `location ^~ /path` — prefix yang menghentikan pencarian regex.
3. `location ~ pola` / `~* pola` — regex (case-sensitive / tidak), urut kemunculan.
4. `location /path` — prefix terpanjang menang.

`root` menempelkan path ke URI; `alias` menggantinya. Kesalahan klasik: memakai `alias` dengan URI yang berakhir `/` tanpa menutup `alias` dengan `/`.

### Praktik

1. Demonstrasi urutan.

```nginx
server {
    listen 80;
    server_name route.example.com;
    root /var/www/route;

    location = /health { return 200 "ok\n"; }
    location ^~ /static/ { }
    location ~* \.(jpg|png|css|js)$ { expires 7d; }
    location / { try_files $uri $uri/ =404; }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s -H 'Host: route.example.com' http://127.0.0.1/health
```

2. `root` vs `alias`.

```nginx
location /files/ { root /srv/data; }    # /files/a.txt -> /srv/data/files/a.txt
location /media/ { alias /srv/photos/; } # /media/a.jpg -> /srv/photos/a.jpg
```

3. `return` dan `rewrite`.

```nginx
location = /lama { return 301 /baru; }
rewrite ^/artikel/(\d+)$ /post/$1 permanent;
```

4. Halaman error sendiri.

```bash
sudo mkdir -p /var/www/demo/errors && echo "<h1>404</h1>" | sudo tee /var/www/demo/errors/404.html
```

```nginx
error_page 404 /errors/404.html;
location = /errors/404.html { internal; }
```

5. Uji semuanya dengan `curl`.

```bash
for p in /health /static/x.css /a.jpg /lama /tidak-ada; do
  printf '%s -> ' "$p"; curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: route.example.com' "http://127.0.0.1$p"
done
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| 404 padahal file ada | `alias` kurang/tambah `/` | Samakan trailing slash antara `location` dan `alias` |
| Semua request kena 404 | `root` salah level | `namei -l` path hasil akhir |
| 403 pada direktori | `autoindex` mati dan tidak ada `index` | Tambahkan `index` atau `autoindex on;` (hati-hati) |

### Latihan

1. Buat `location = /health` yang membalas `200` tanpa menyentuh disk.
2. Buat `location /media/` memakai `alias` ke direktori luar `root` dan buktikan file termuat.

**Kriteria selesai:** kamu bisa menebak lokasi mana yang menangani sebuah URI, lalu membuktikannya dengan `curl`.

### Cek pemahaman

1. Mana yang menang: `= /a` atau `~ \.php$` untuk URI `/a`? — `= /a`, karena pencocokan persis paling tinggi.
2. Beda `root` dan `alias`? — `root` menambahkan URI ke path; `alias` menggantikan bagian yang cocok.
3. Fungsi `internal;`? — Melarang akses langsung dari klien; hanya boleh dipakai lewat `error_page`/redirect internal.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat 5 `location` dengan jenis pencocokan berbeda (persis, `^~`, regex, prefix, alias), lalu uji URI yang sama ke semuanya.

**Siapkan:** Baca dokumentasi `ngx_http_proxy_module` bagian `proxy_pass` sebagai persiapan.

**Bawa bukti:** Tabel tebakan vs hasil (kode status) untuk 5 URI + penjelasan satu kasus `root` vs `alias`.

Dibahas di awal **Pelajaran 2.6** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.6 — Reverse proxy

### Tujuan

Meneruskan request ke aplikasi di `127.0.0.1:3000` dengan header yang benar, termasuk WebSocket.

### Konsep

Reverse proxy berdiri di depan aplikasi: klien bicara ke Nginx, Nginx bicara ke aplikasi. Keuntungan: TLS di satu tempat, aplikasi tidak perlu port publik, ganti aplikasi tanpa mengubah DNS, log terpusat.

`proxy_pass http://127.0.0.1:3000;` mengirim request ke backend. Tanpa `proxy_set_header`, aplikasi kehilangan informasi penting:

| Header | Gunanya |
| --- | --- |
| `Host` | Aplikasi tahu domain yang diminta (penting untuk cookie, redirect) |
| `X-Real-IP` | IP klien asli |
| `X-Forwarded-For` | Rantai IP klien |
| `X-Forwarded-Proto` | Skema asli (http/https) |

WebSocket butuh header `Upgrade` dan `Connection` yang di-set dari request klien. Karena `Connection` harus bernilai `upgrade` hanya saat ada `Upgrade`, dipakai `map`.

### Praktik

1. Siapkan backend sederhana untuk latihan.

```bash
mkdir -p ~/apps/api && cat > ~/apps/api/server.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer
import json, os

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        body = json.dumps({
            "path": self.path,
            "host": self.headers.get("Host"),
            "real_ip": self.headers.get("X-Real-IP"),
            "proto": self.headers.get("X-Forwarded-Proto"),
        }).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(body)

HTTPServer(("127.0.0.1", 3000), H).serve_forever()
PY
python3 ~/apps/api/server.py &   # untuk latihan; di produksi pakai systemd
curl -s http://127.0.0.1:3000/cek
```

2. Konfigurasi reverse proxy.

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      '';      # string kosong: header Connection tidak diubah, keepalive upstream tetap jalan
}

upstream app_backend {
    server 127.0.0.1:3000;
    keepalive 16;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
        proxy_buffering       off;   # untuk SSE/streaming
    }
}
```

Catatan: `map` harus berada di dalam blok `http`, bukan di dalam `server`. Letakkan di `/etc/nginx/conf.d/proxy-map.conf`.

3. Terapkan dan uji.

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s -H 'Host: app.example.com' http://127.0.0.1/ | python3 -m json.tool
curl -s -H 'Host: app.example.com' -H 'X-Forwarded-Proto: https' http://127.0.0.1/cek
```

4. Pastikan backend **tidak** terbuka ke internet.

```bash
sudo ss -tulpn | grep 3000        # harus 127.0.0.1:3000, bukan 0.0.0.0:3000
```

5. Ganti backend tanpa menyentuh klien: ubah `server` di `upstream`, `nginx -t`, reload.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| 502 Bad Gateway | Backend mati atau bukan HTTP | `curl -sI http://127.0.0.1:3000`, `journalctl -u app -n 50` |
| 504 Gateway Timeout | Backend lambat/menggantung | Naikkan `proxy_read_timeout`, perbaiki aplikasinya |
| Redirect balik ke `http://` | `X-Forwarded-Proto` tidak di-set | Tambahkan `proxy_set_header X-Forwarded-Proto $scheme;` |
| Cookie hilang | Cookie di-set untuk domain backend | Set `proxy_cookie_domain`, atau perbaiki konfigurasi aplikasi |
| WebSocket gagal (`426`) | Header `Upgrade`/`Connection` tidak diteruskan | Pakai `map` seperti contoh |

### Latihan

1. Tambahkan endpoint `/health` di backend dan `location = /health` yang mem-proxy langsung ke sana dengan `proxy_pass http://127.0.0.1:3000/health;`.
2. Buktikan aplikasi menerima `X-Forwarded-Proto: https` padahal kamu mengakses lewat HTTP ke localhost.

**Kriteria selesai:** `curl -H 'Host: app.example.com' http://127.0.0.1/` mengembalikan JSON dengan `host` dan `proto` yang benar.

### Cek pemahaman

1. Kenapa aplikasi sebaiknya listen di `127.0.0.1`? — Hanya terjangkau dari server itu; satu-satunya pintu masuk adalah Nginx.
2. Apa akibat `Host` tidak diteruskan? — Aplikasi memakai `localhost:3000` sebagai host, cookie dan redirect jadi salah.
3. Kenapa `Connection` di-set lewat `map`? — Karena harus bernilai `upgrade` hanya jika klien mengirim `Upgrade`.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Arahkan `app.example.com` ke aplikasi di `127.0.0.1:3000` dengan empat header wajib dan dukungan WebSocket.

**Siapkan:** Pastikan aplikasi menerima `X-Forwarded-Proto` dan `X-Real-IP` yang benar.

**Bawa bukti:** Respons JSON dari aplikasi yang menampilkan `host`, `real_ip`, dan `proto` + `ss` yang membuktikan backend hanya di `127.0.0.1`.

Dibahas di awal **Pelajaran 2.7** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.7 — HTTPS dengan Let's Encrypt

### Tujuan

Menerbitkan sertifikat, memasangnya di Nginx, dan memastikan perpanjangan otomatis berjalan.

### Konsep

Let's Encrypt memvalidasi kepemilikan domain lewat HTTP-01: server harus menjawab `http://domain/.well-known/acme-challenge/...` pada port 80. Karena itu port 80 wajib terbuka dan DNS harus sudah benar **sebelum** meminta sertifikat.

Certbot dengan plugin Nginx mengubah konfigurasi secara otomatis, lalu memasang timer systemd untuk perpanjangan.

### Praktik

1. Prasyarat.

```bash
dig +short app.example.com        # harus IP server
sudo ufw status | grep -E '80|443'
sudo systemctl is-active nginx
```

2. Pasang certbot.

```bash
sudo apt install -y certbot python3-certbot-nginx
```

3. Terbitkan sertifikat (pastikan `server_name` sudah benar).

```bash
sudo certbot --nginx -d app.example.com --redirect --agree-tos -m admin@example.com --no-eff-email
# tambahkan `-d www.app.example.com` HANYA jika server_name untuk www memang sudah ada
# di Nginx; domain tanpa server block membuat validasi gagal.
```

4. Verifikasi hasil.

```bash
curl -sI https://app.example.com | head -3
curl -s -o /dev/null -w '%{http_code}\n' http://app.example.com    # harus 301
echo | openssl s_client -connect app.example.com:443 -servername app.example.com 2>/dev/null | openssl x509 -noout -subject -dates
```

5. Perpanjangan otomatis.

```bash
systemctl list-timers | grep certbot
sudo certbot renew --dry-run
```

6. HSTS dan header keamanan tambahan.

```bash
sudo nano /etc/nginx/snippets/security-headers.conf
```

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

```nginx
# di dalam server block 443
include /etc/nginx/snippets/security-headers.conf;
```

Peringatan: `includeSubDomains` pada HSTS berarti semua subdomain wajib HTTPS. Jangan aktifkan sebelum semua subdomain siap.

7. Perbarui sertifikat manual (kalau perlu setelah DNS berubah).

```bash
sudo certbot renew --force-renewal
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Timeout during connect` | Port 80 tertutup atau DNS belum mengarah ke server | Buka 80 di `ufw`, cek `dig`, cek panel VPS |
| `Invalid response ... 404` | Nginx tidak melayani domain itu | Pastikan `server_name` benar dan block aktif |
| `too many failed authorizations` | Percobaan berulang gagal | Tunggu (rate limit 1 jam), perbaiki penyebab dulu |
| Browser protes sertifikat | Sertifikat untuk domain lain / belum terpasang di block SSL | Cek `openssl s_client`, pastikan `ssl_certificate` menunjuk domain itu |

### Latihan

1. Terbitkan sertifikat untuk satu domain, buktikan redirect 301 dari HTTP dan `ssl_certificate` ada di file konfigurasi.
2. Jalankan `certbot renew --dry-run` dan sebutkan timer yang bertanggung jawab atas perpanjangan.

**Kriteria selesai:** `https://app.example.com` hijau di browser, dan konfigurasi lolos `nginx -t`.

### Cek pemahaman

1. Kenapa port 80 harus terbuka walau situs memakai HTTPS? — Untuk validasi HTTP-01 dan redirect ke HTTPS.
2. Apa yang dilakukan `certbot --nginx`? — Menerbitkan sertifikat, menambahkan direktif SSL ke server block, dan menambah redirect.
3. Apa risikonya HSTS `includeSubDomains`? — Subdomain yang belum punya sertifikat menjadi tidak bisa diakses.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Terbitkan sertifikat untuk satu domain, aktifkan redirect HTTP→HTTPS dan HSTS, lalu jalankan `certbot renew --dry-run`.

**Siapkan:** Catat tanggal kedaluwarsa sertifikat dan nama timer perpanjangan.

**Bawa bukti:** Output `openssl s_client ... -dates` + kode 301 dari HTTP + hasil dry-run.

Dibahas di awal **Pelajaran 2.8** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 2.8 — Load balancing, proteksi, dan tuning produksi

### Tujuan

Membagi beban ke beberapa backend dan melindungi aplikasi dari lonjakan serta pengunjung kasar.

### Konsep

`upstream` adalah daftar backend. Metode: round-robin (default), `least_conn` (ke koneksi paling sedikit), `ip_hash` (klien sama ke backend sama, untuk sesi tanpa shared state).

`limit_req_zone` membatasi laju request per IP. Batas ini didefinisikan di blok `http` dan dipakai di `location` dengan `limit_req`.

`client_max_body_size` mengatur ukuran body maksimum; default 1 MB adalah penyebab tersembunyi error 413 saat upload.

### Praktik

1. Upstream dan metode.

```nginx
upstream app_backend {
    least_conn;
    server 127.0.0.1:3000 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:3001 max_fails=3 fail_timeout=10s backup;
    keepalive 16;
}
```

2. Rate limit.

```bash
sudo nano /etc/nginx/conf.d/limits.conf
```

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
```

```nginx
location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
    limit_conn conn_limit 20;
    proxy_pass http://app_backend;
}
```

3. Basic auth (misalnya untuk halaman admin).

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
sudo chmod 640 /etc/nginx/.htpasswd && sudo chown root:www-data /etc/nginx/.htpasswd
```

```nginx
location /admin/ {
    auth_basic "Area terbatas";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://app_backend;
}
```

4. Tuning penting.

```nginx
client_max_body_size 25m;        # upload file
client_body_timeout 30s;
gzip on;                          # di blok http
gzip_types text/plain text/css application/json application/javascript;
```

5. Diagnosis cepat error umum.

```bash
sudo tail -50 /var/log/nginx/error.log
sudo ss -tulpn | grep -E ':3000|:3001'
curl -sI http://127.0.0.1:3000
journalctl -u app -n 50 --no-pager
```

| Kode | Langkah pertama |
| --- | --- |
| 502 | Cek backend hidup: `curl -sI http://127.0.0.1:3000`, lalu `journalctl -u app` |
| 504 | Cek `proxy_read_timeout` dan waktu respons backend |
| 403 | Cek izin path (`namei -l`) dan `auth_basic` |
| 413 | Naikkan `client_max_body_size` |
| 499 | Klien membatalkan; biasanya backend terlalu lambat |

6. Struktur file yang rapi untuk banyak situs.

```
/etc/nginx/sites-available/app.example.com
/etc/nginx/snippets/security-headers.conf
/etc/nginx/conf.d/limits.conf
/etc/nginx/conf.d/proxy-map.conf
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `limit_req_zone` di tempat salah | Ditaruh di dalam `server` | Pindahkan ke blok `http` |
| `upstream` tidak dikenal | Didefinisikan setelah dipakai / di file lain | Definisikan sekali di blok `http` |
| 429 terlalu sering untuk pengguna sah | `rate`/`burst` terlalu ketat | Naikkan `burst`, atau kecualikan IP tertentu |

### Latihan

1. Tambahkan backend kedua di port berbeda, buktikan round-robin dengan melihat log akses dua backend.
2. Pasang rate limit 5r/s dengan `burst=10` dan buktikan dengan `for i in $(seq 1 30); do curl -s -o /dev/null -w '%{http_code} ' http://127.0.0.1/api/; done`.

**Kriteria selesai:** error 413 pada upload hilang setelah `client_max_body_size` dinaikkan, dan kamu bisa menjelaskan tiap baris pada blok `location /api/`.

### Cek pemahaman

1. Kapan `ip_hash` diperlukan? — Saat sesi pengguna disimpan di memori satu backend, tanpa penyimpanan bersama.
2. Apa fungsi `burst` pada `limit_req`? — Mengizinkan lonjakan sesaat di atas laju rata-rata sebelum menolak.
3. Kenapa `keepalive` pada `upstream` menghemat sumber daya? — Koneksi ke backend dipakai ulang, tidak membuat koneksi TCP baru tiap request.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tambahkan `upstream` dua backend, `limit_req` 10r/s dengan `burst=20`, dan basic auth untuk `/admin/`.

**Siapkan:** Uji dengan 30 request beruntun dan simpan log distribusinya.

**Bawa bukti:** Bukti distribusi request ke dua backend + respons 429 saat batas terlampaui + `curl` 401 pada `/admin/`.

Dibahas di awal **Pelajaran 3.1** — 10 menit pertama, sebelum materi baru.

---

## Proyek mini modul

Publikasikan satu aplikasi di `127.0.0.1:3000` sebagai `https://app.example.com`:

1. Jalankan aplikasi (boleh skrip latihan pelajaran 2.6) lewat systemd, listen di `127.0.0.1:3000`.
2. Buat server block `app.example.com` dengan reverse proxy lengkap: `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, dukungan WebSocket.
3. Tambahkan redirect HTTP→HTTPS dan `location = /health` yang membalas `200 ok`.
4. Terbitkan sertifikat Let's Encrypt, buktikan `certbot renew --dry-run` sukses.
5. Tambahkan `limit_req` 10r/s `burst=20`, `client_max_body_size 25m`, dan header keamanan.
6. Buktikan port 3000 tidak bisa diakses dari luar server (`nc -zv IP 3000` dari laptop harus gagal).
7. Tulis `catatan-nginx.md`: file konfigurasi yang dipakai, perintah reload, dan cara membaca log.

**Kriteria selesai:** `curl -sI https://app.example.com` mengembalikan 200, `curl -s -o /dev/null -w '%{http_code}' http://app.example.com` mengembalikan 301, dan `sudo nginx -t` bersih.

## Config contoh lengkap

Satu file `/etc/nginx/nginx.conf` utuh yang siap pakai untuk skenario modul ini (statis `example.com` + reverse proxy `app.example.com`). Blok-blok terpisah di pelajaran sebelumnya adalah potongan dari file ini. Sudah diuji dengan `nginx -t`.

```nginx
# /etc/nginx/nginx.conf
# Konfigurasi Nginx utuh untuk Ubuntu 24.04 LTS (Nginx 1.24.x dari repo Ubuntu).
# Skenario Modul 2: example.com (statis) + app.example.com (reverse proxy) di 203.0.113.10.
# Semua direktif di bawah ada di dokumentasi resmi Nginx (lihat bagian Referensi).

user  www-data;                          # worker berjalan sebagai user tanpa hak root
worker_processes auto;                   # satu worker per core CPU
pid /run/nginx.pid;                      # lokasi file PID master process
error_log /var/log/nginx/error.log warn; # log error global, level warn

include /etc/nginx/modules-enabled/*.conf;   # modul dinamis dari paket distro

events {
    worker_connections 1024;             # koneksi simultan per worker
    multi_accept on;                     # terima beberapa koneksi sekaligus
}

http {
    ## --- Dasar ---
    sendfile on;                         # kirim file langsung dari kernel
    tcp_nopush on;                       # gabungkan header dengan awal body
    tcp_nodelay on;                      # jangan tunda paket kecil saat keepalive
    keepalive_timeout 65;                # lama koneksi client dibiarkan terbuka
    types_hash_max_size 2048;
    server_tokens off;                   # sembunyikan versi Nginx di header Server

    include /etc/nginx/mime.types;       # peta ekstensi -> MIME type
    default_type application/octet-stream;

    ## --- Logging ---
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    ## --- TLS untuk semua server block 443 ---
    ssl_protocols TLSv1.2 TLSv1.3;       # buang TLS 1.0/1.1 yang sudah tidak aman
    ssl_prefer_server_ciphers off;       # biarkan client memilih cipher tercepat
    ssl_session_cache shared:SSL:10m;    # cache sesi TLS antar worker
    ssl_session_timeout 1d;              # masa berlaku cache sesi

    ## --- Kompresi ---
    gzip on;                             # aktifkan gzip
    gzip_vary on;                        # kirim Vary: Accept-Encoding
    gzip_proxied any;                    # tetap kompres response dari backend
    gzip_comp_level 5;                   # keseimbangan CPU vs rasio kompresi
    gzip_min_length 1000;                # response < 1 KB tidak sepadan dikompres
    gzip_types text/plain text/css text/xml application/json
               application/javascript application/xml application/xml+rss
               image/svg+xml;            # text/html selalu dikompres, tidak perlu didaftar

    ## --- Rate limit (zone wajib di konteks http) ---
    limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_ip:10m rate=1r/s;
    limit_req_status 429;                # status saat ditolak (default 503)
    limit_req_log_level warn;

    ## --- Helper WebSocket ---
    map $http_upgrade $connection_upgrade {
        default upgrade;                 # ada header Upgrade -> teruskan
        ''      '';                      # tidak ada -> biarkan header apa adanya (keepalive utuh)
    }

    ## --- Grup backend aplikasi ---
    upstream app_nodes {
        least_conn;                      # pilih node dengan koneksi aktif tersedikit
        server 127.0.0.1:3000 max_fails=3 fail_timeout=15s;
        server 127.0.0.1:3001 max_fails=3 fail_timeout=15s;
        keepalive 16;                    # pakai ulang koneksi TCP ke backend
    }

    ## --- Penangkap request tak dikenal ---
    server {
        listen 80 default_server;        # blok default untuk IPv4
        listen [::]:80 default_server;   # dan IPv6
        server_name _;                   # nama dummy, bukan domain nyata
        return 444;                      # tutup koneksi tanpa balasan
    }

    ## --- Redirect HTTP -> HTTPS untuk semua domain ---
    server {
        listen 80;
        listen [::]:80;
        server_name example.com www.example.com app.example.com;
        location /.well-known/acme-challenge/ {
            root /var/www/html;          # biarkan certbot menaruh file challenge
        }
        location / {
            return 301 https://$host$request_uri;
        }
    }

    ## --- example.com: situs statis ---
    server {
        listen 443 ssl http2;            # Nginx 1.24: HTTP/2 lewat parameter listen.
        listen [::]:443 ssl http2;       # Di 1.25.1+ tulis `listen 443 ssl;` + `http2 on;`
        server_name example.com;

        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        root /var/www/static.example.com;
        index index.html;

        access_log /var/log/nginx/static.example.com.access.log main;
        error_log  /var/log/nginx/static.example.com.error.log warn;

        location / {
            try_files $uri $uri/ =404;   # coba file, coba direktori, kalau gagal 404
        }

        location = /health {
            default_type text/plain;
            return 200 "ok\n";           # health check murah, tidak menyentuh disk
        }

        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /404.html { internal; }    # hanya lewat error_page
        location = /50x.html { internal; }
    }

    ## --- www -> non-www (kanonik: example.com) ---
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name www.example.com;

        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

        return 301 https://example.com$request_uri;
    }

    ## --- app.example.com: reverse proxy ke 127.0.0.1:3000/3001 ---
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name app.example.com;

        ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

        client_max_body_size 25m;        # batas body request; default hanya 1m
        server_tokens off;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        access_log /var/log/nginx/app.example.com.access.log main;
        error_log  /var/log/nginx/app.example.com.error.log warn;

        # Aset statis dilayani Nginx, tidak mengganggu aplikasi
        location ^~ /static/ {
            root /var/www/app.example.com;
            expires 30d;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        location = /favicon.ico { return 204; }

        # Admin: basic auth + rate limit. Hanya layak di atas HTTPS.
        location /admin/ {
            auth_basic           "Area admin";
            auth_basic_user_file /etc/nginx/.htpasswd;
            limit_req zone=per_ip burst=10 nodelay;

            proxy_pass http://app_nodes;
            proxy_http_version 1.1;
            proxy_set_header Connection        "";   # kosong agar keepalive upstream aktif
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Login: rate limit ketat untuk menahan brute force
        location = /login {
            limit_req zone=login_ip burst=3 nodelay;

            proxy_pass http://app_nodes;
            proxy_http_version 1.1;
            proxy_set_header Connection        "";
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # WebSocket: Upgrade/Connection adalah header hop-by-hop, harus di-set manual
        location /ws/ {
            proxy_pass http://app_nodes;
            proxy_http_version 1.1;
            proxy_set_header Upgrade           $http_upgrade;
            proxy_set_header Connection        $connection_upgrade;
            proxy_set_header Host              $host;
            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }

        # Upload: body besar, diteruskan streaming tanpa ditulis ke disk dulu
        location /upload {
            client_max_body_size 100m;
            proxy_pass http://app_nodes;
            proxy_http_version 1.1;
            proxy_request_buffering off;
            proxy_read_timeout 300s;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Sisanya ke aplikasi
        location / {
            limit_req zone=per_ip burst=20 nodelay;

            proxy_pass http://app_nodes;         # tanpa URI -> path dikirim apa adanya
            proxy_http_version 1.1;              # default 1.0 di 1.24; wajib 1.1 untuk keepalive
            proxy_set_header Connection        "";
            proxy_set_header Host              $host;                          # domain asli, bukan upstream
            proxy_set_header X-Real-IP         $remote_addr;                   # IP client asli
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;     # rantai proxy
            proxy_set_header X-Forwarded-Proto $scheme;                        # https, bukan http

            proxy_connect_timeout 5s;            # gagal sambung cepat -> 502
            proxy_send_timeout    60s;
            proxy_read_timeout    60s;           # backend terlalu lama -> 504

            proxy_buffering on;                  # serap response backend ke buffer
            proxy_buffers 16 16k;
            proxy_buffer_size 16k;
        }

        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /404.html { internal; root /var/www/app.example.com; }
        location = /50x.html { internal; root /var/www/app.example.com; }
    }
}
```

Cara memakai:

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak   # selalu backup dulu
sudo nginx -t                                             # validasi sintaks
sudo systemctl reload nginx                               # terapkan tanpa downtime
```

Kalau lebih suka struktur per-situs, potong tiap `server {}` ke `/etc/nginx/sites-available/<domain>` lalu `ln -s` ke `sites-enabled`. Blok `map`, `limit_req_zone`, dan `upstream` boleh tetap ditulis di file situs, karena berkas di `sites-enabled` di-`include` **di dalam** blok `http`.

## Cheat sheet

| Kebutuhan | Perintah / direktif |
| --- | --- |
| Uji konfigurasi | `sudo nginx -t` |
| Terapkan tanpa downtime | `sudo systemctl reload nginx` |
| Lihat versi & modul | `nginx -v`, `nginx -V` |
| Log akses/error | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` |
| Cek port | `sudo ss -tulpn \| grep -E ':80\|:443'` |
| Uji tanpa DNS | `curl -sI -H 'Host: domain' http://127.0.0.1` |
| Sertifikat | `sudo certbot --nginx -d domain`, `sudo certbot renew --dry-run` |
| Lihat masa berlaku cert | `echo \| openssl s_client -connect domain:443 -servername domain 2>/dev/null \| openssl x509 -noout -dates` |
| Rate limit | `limit_req_zone` (http) + `limit_req` (location) |
| Ukuran upload | `client_max_body_size 25m;` |

## Referensi

Dokumentasi resmi yang dipakai untuk memverifikasi setiap direktif di modul ini:

- Nginx — `ngx_http_core_module` (`listen`, `server_name`, `root`, `alias`, `location`, `try_files`, `error_page`, `client_max_body_size`): https://nginx.org/en/docs/http/ngx_http_core_module.html
- Nginx — `ngx_http_proxy_module` (`proxy_pass`, `proxy_set_header`, `proxy_http_version`, timeout, buffering): https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- Nginx — `ngx_http_upstream_module` (`upstream`, `least_conn`, `ip_hash`, `keepalive`, `max_fails`, `fail_timeout`): https://nginx.org/en/docs/http/ngx_http_upstream_module.html
- Nginx — `ngx_http_limit_req_module` (`limit_req_zone`, `limit_req`, `limit_req_status`): https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
- Nginx — `ngx_http_headers_module` (`add_header`, `expires`): https://nginx.org/en/docs/http/ngx_http_headers_module.html
- Nginx — `ngx_http_gzip_module` (`gzip`, `gzip_types`, `gzip_vary`, `gzip_min_length`): https://nginx.org/en/docs/http/ngx_http_gzip_module.html
- Nginx — `ngx_http_rewrite_module` (`rewrite`, `return`, `if`): https://nginx.org/en/docs/http/ngx_http_rewrite_module.html
- Nginx — `ngx_http_ssl_module` (`ssl_certificate`, `ssl_protocols`, `ssl_session_cache`): https://nginx.org/en/docs/http/ngx_http_ssl_module.html
- Nginx — `ngx_http_auth_basic_module` (`auth_basic`, `auth_basic_user_file`): https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html
- Nginx — `ngx_http_v2_module` (direktif `http2`, tersedia sejak 1.25.1): https://nginx.org/en/docs/http/ngx_http_v2_module.html
- Nginx — WebSocket proxying (`Upgrade`, `Connection`): https://nginx.org/en/docs/http/websocket.html
- Nginx — cara memilih virtual server dari header `Host`: https://nginx.org/en/docs/http/request_processing.html
- Nginx — parameter command line (`-t`, `-T`, `-v`, `-V`, `-s`): https://nginx.org/en/docs/switches.html
- Certbot — instruksi resmi Nginx (snap, `certbot --nginx`, `certbot renew --dry-run`): https://certbot.eff.org/instructions?ws=nginx&os=snap
- Certbot — pemakaian plugin nginx dan flag `--redirect`/`--dry-run`: https://eff-certbot.readthedocs.io/en/stable/using.html
- Let's Encrypt — challenge HTTP-01 lewat port 80: https://letsencrypt.org/docs/challenge-types/
- MDN — header `Strict-Transport-Security` (HSTS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security
- Ubuntu 24.04 (noble) — paket `nginx` 1.24.0: https://packages.ubuntu.com/noble/nginx
- Ubuntu 24.04 (noble) — isi paket `nginx-common` (`nginx.conf`, `sites-available`, `snippets`, `proxy_params`, profil ufw): https://packages.ubuntu.com/noble/all/nginx-common/filelist
