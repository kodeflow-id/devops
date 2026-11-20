# Modul 3 — Docker

## Ringkasan modul

Sampai sini aplikasi jalan langsung di server: rawan beda versi, susah di-rollback, dan sulit dipindah. Docker mengemas aplikasi beserta dependensinya menjadi image, lalu menjalankannya sebagai container.

Asumsi: Ubuntu 24.04 LTS, Docker Engine dari repository resmi Docker (`docker-ce`), Docker Compose v2 (`docker compose`, bukan `docker-compose`). Aplikasi contoh: API kecil yang listen di `0.0.0.0:3000` di dalam container.

## Hasil pembelajaran

- Menjelaskan perbedaan image, container, dan registry, serta kapan container bukan pilihan tepat.
- Memasang Docker dari repository resmi dan menjelaskan risiko bergabung ke grup `docker`.
- Menulis Dockerfile multi-stage yang menghasilkan image kecil dan berjalan sebagai user non-root.
- Mengelola data dengan volume dan konfigurasi dengan env file tanpa menaruh rahasia di image.
- Menghubungkan container ke database dan ke Nginx di host lewat network dan port publish.
- Menjalankan aplikasi plus database dengan satu perintah `docker compose up -d`.
- Mendiagnosis container yang langsung mati, port bentrok, dan volume yang menutupi data.

---

## Pelajaran 3.1 — Kenapa container

### Tujuan

Memahami masalah yang dipecahkan container, dan kapan sebaiknya tidak memakainya.

### Konsep

| Istilah | Arti |
| --- | --- |
| Image | Paket read-only berisi aplikasi + dependensi + metadata |
| Container | Proses yang berjalan dari image, punya filesystem dan network sendiri |
| Registry | Tempat menyimpan image (Docker Hub, GHCR) |
| Volume | Penyimpanan persisten di luar siklus hidup container |

Container berbagi kernel host, jadi jauh lebih ringan dari VM yang menjalankan sistem operasi penuh. Ini juga berarti isolasi container lebih lemah daripada VM — masalah keamanan kernel berdampak ke semua container.

Masalah "di laptop saya jalan" hilang karena image membawa semua dependensi. Deploy jadi: bangun image, dorong ke registry, tarik di server, jalankan.

Kapan **tidak** memakai Docker: aplikasi tunggal yang sederhana dan sudah stabil dengan systemd; kebutuhan kernel kustom atau akses perangkat khusus; tim kecil yang belum siap merawat registry dan pipeline. Docker menambah lapisan yang harus dipahami, bukan menghilangkannya.

### Praktik

1. Lihat perbedaan ukuran dan isolasi.

```bash
uname -r                                  # kernel yang dibagi semua container
docker run --rm alpine uname -r           # kernel yang sama di dalam container
docker run --rm alpine cat /etc/os-release | head -2   # OS di dalam container beda dari host
```

2. Alur hidup sebuah image.

```bash
docker pull nginx:1.27-alpine
docker images
docker run -d --name web nginx:1.27-alpine
docker ps
docker exec -it web nginx -v
docker rm -f web
docker rmi nginx:1.27-alpine
```

3. Container itu proses, bukan mesin.

```bash
docker run -d --name web nginx:1.27-alpine
ps aux | grep -c nginx        # proses container terlihat di host
docker rm -f web
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Bingung image vs container | Analogi yang salah | Image = cetakan, container = hasil jalan dari cetakan |
| Data hilang setelah container dihapus | Data disimpan di layer container | Pakai volume (pelajaran 3.6) |

### Latihan

1. Jalankan `alpine` dan buktikan kernelnya sama dengan host tetapi OS-nya berbeda.
2. Jelaskan dalam tiga kalimat kenapa container cocok untuk aplikasi web dan kurang cocok untuk menjalankan database besar.

**Kriteria selesai:** kamu bisa menjelaskan beda image, container, dan volume tanpa melihat catatan.

### Cek pemahaman

1. Apa yang tidak dibagi container dengan host? — Filesystem, network namespace, dan proses (isolasinya), meski kernel dibagi.
2. Di mana image disimpan secara default? — Di registry seperti Docker Hub atau GHCR, dan salinannya di cache lokal server.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tulis `container-vs-vm.md`: 5 kalimat perbandingan dan 2 kondisi nyata ketika container bukan pilihan tepat.

**Siapkan:** Jalankan satu container dan bandingkan `uname -r` di host dan di dalam container.

**Bawa bukti:** Isi catatan + output `uname -r` dari keduanya.

Dibahas di awal **Pelajaran 3.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.2 — Instalasi dan verifikasi

### Tujuan

Memasang Docker Engine dari repository resmi dan memahami konsekuensi grup `docker`.

### Konsep

Paket `docker.io` dari Ubuntu sering tertinggal versi. Untuk server, pakai `docker-ce` dari `download.docker.com`.

Grup `docker` memberi hak setara root: anggota grup bisa me-mount seluruh filesystem host ke dalam container. Di server produksi, batasi siapa yang masuk grup ini — atau pakai rootless Docker.

### Praktik

1. Pasang dari repository resmi.

```bash
sudo apt update && sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

2. Verifikasi.

```bash
sudo systemctl enable --now docker
docker version
docker info | head -20
sudo docker run --rm hello-world
docker compose version
```

3. Tanpa `sudo` (opsional, sadari risikonya).

```bash
sudo usermod -aG docker deploy
# login ulang agar grup berlaku
id deploy | grep docker
docker ps
```

Peringatan: bergabung ke grup `docker` sama dengan memberi akses root tanpa password kepada user itu.

4. Batasi konsumsi supaya server kecil tidak tumbang.

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```bash
sudo systemctl restart docker
docker info | grep -i "logging driver"
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `permission denied /var/run/docker.sock` | User belum di grup `docker` atau belum login ulang | `usermod -aG docker $USER` lalu login ulang |
| `Cannot connect to the Docker daemon` | Service mati | `sudo systemctl start docker`, cek `journalctl -u docker` |
| `docker compose` tidak dikenal | Hanya plugin compose yang belum terpasang | Pasang `docker-compose-plugin` |

### Latihan

1. Pasang Docker dari repo resmi, buktikan versi server dan versi Compose.
2. Aktifkan log rotation di `daemon.json` dan buktikan nilainya terbaca di `docker info`.

**Kriteria selesai:** `docker run --rm hello-world` sukses dan `docker compose version` menampilkan v2.

### Cek pemahaman

1. Kenapa `docker.io` kurang cocok untuk produksi? — Versinya tertinggal dari rilis resmi Docker.
2. Apa risiko grup `docker`? — Setara root; bisa me-mount host filesystem.
3. Kenapa log rotation penting di VPS kecil? — Log container tanpa batas bisa memenuhi disk dan mematikan semua service.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Pasang Docker dari repo resmi, aktifkan log rotation di `daemon.json`, dan jalankan `hello-world`.

**Siapkan:** Catat versi Docker Engine dan Compose yang terpasang.

**Bawa bukti:** `docker version`, `docker compose version`, `docker info | grep -i logging`, dan isi `daemon.json`.

Dibahas di awal **Pelajaran 3.3** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.3 — Image dan container sehari-hari

### Tujuan

Menjalankan, memeriksa, dan membersihkan container tanpa menebak-nebak.

### Konsep

Opsi `docker run` yang paling sering dipakai:

| Opsi | Arti |
| --- | --- |
| `-d` | Detach, jalan di latar belakang |
| `-p 127.0.0.1:3000:3000` | Publish port, host:container (bind ke localhost saja) |
| `-e KEY=value` | Environment variable |
| `--env-file .env` | Ambil environment dari file |
| `-v nama:/path` | Mount volume |
| `--name app` | Nama tetap, memudahkan perintah lain |
| `--rm` | Hapus otomatis saat berhenti (untuk sekali pakai) |
| `--restart unless-stopped` | Nyalakan ulang saat boot/crash |

Status container penting: `Up`, `Exited (0)` berarti program selesai normal, `Exited (1)` berarti error, `Restarting` berarti crash berulang.

### Praktik

1. Jalankan dan periksa.

```bash
docker run -d --name api -p 127.0.0.1:3000:80 -e TZ=Asia/Jakarta nginx:1.27-alpine
docker ps
docker logs api
docker logs -f --tail 20 api
curl -s http://127.0.0.1:3000 | head -3
```

2. Masuk ke dalam container.

```bash
docker exec -it api sh
# di dalam: ls /etc/nginx, cat /etc/os-release
exit
docker exec api nginx -t
```

3. Periksa detail.

```bash
docker inspect api | head -40
docker inspect -f '{{.State.Status}} {{.NetworkSettings.IPAddress}}' api
docker stats --no-stream
```

4. Hentikan dan hapus dengan benar.

```bash
docker stop api
docker start api
docker restart api
docker rm -f api
```

5. Bersihkan — dengan hati-hati.

```bash
docker system df                  # lihat pemakaian sebelum menghapus
docker container prune            # container berhenti
docker image prune                # image tanpa tag
docker system prune -a --volumes  # PERINGATAN: hapus image tak terpakai DAN volume
```

Peringatan: `docker system prune -a --volumes` bisa menghapus data database yang tidak sedang dipakai container. Di server produksi, jangan dijalankan tanpa membaca `docker system df` dan memastikan semua volume penting sudah dicadangkan.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Container langsung `Exited (0)` | Perintah utama selesai; untuk service butuh program yang tetap jalan | Periksa `CMD` image, jalankan `docker logs` |
| `port is already allocated` | Port host sudah dipakai | `sudo ss -tulpn \| grep :3000`, ganti port host |
| `docker logs` kosong | Aplikasi menulis ke file, bukan stdout | Arahkan log aplikasi ke stdout (kebiasaan container) |

### Latihan

1. Jalankan Nginx dengan port host `8080`, buktikan halaman default muncul, lalu hapus container tanpa menghapus image-nya.
2. Tampilkan status dan IP container dalam satu perintah `docker inspect -f`.

**Kriteria selesai:** kamu bisa menjelaskan kenapa log container sebaiknya ke stdout, bukan ke file di dalam container.

### Cek pemahaman

1. Beda `docker stop` dan `docker kill`? — `stop` mengirim SIGTERM lalu SIGKILL setelah tenggang waktu; `kill` langsung SIGKILL.
2. Kenapa `--rm` cocok untuk percobaan? — Container langsung dibersihkan, tidak menumpuk.
3. Apa arti `-p 127.0.0.1:3000:80`? — Hanya localhost host yang bisa mengakses; port 80 container dipetakan ke 3000.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Jalankan Nginx di port host 8080, inspeksi container (logs, inspect, stats), lalu hapus container tanpa menghapus image.

**Siapkan:** Sebelum membersihkan apa pun, jalankan `docker system df` dan catat angkanya.

**Bawa bukti:** `docker ps -a`, `docker logs`, dan tabel kecil: apa yang hilang saat container dihapus vs saat image dihapus.

Dibahas di awal **Pelajaran 3.4** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.4 — Dockerfile pertama

### Tujuan

Menulis Dockerfile yang bisa di-build, cepat di-cache, dan tidak boros.

### Konsep

Instruksi utama:

| Instruksi | Fungsi |
| --- | --- |
| `FROM` | Image dasar |
| `WORKDIR` | Direktori kerja di dalam image |
| `COPY` | Menyalin file dari konteks build |
| `RUN` | Menjalankan perintah saat build, hasilnya jadi layer |
| `ENV` / `ARG` | Variabel saat runtime / saat build |
| `EXPOSE` | Dokumentasi port (tidak membuka apa pun) |
| `CMD` | Perintah default, bisa ditimpa saat `docker run` |
| `ENTRYPOINT` | Perintah tetap; argumen `docker run` menjadi tambahannya |
| `USER` | User yang menjalankan proses |

Aturan cache: setiap instruksi menghasilkan layer. Kalau sebuah layer berubah, semua layer setelahnya dibangun ulang. Karena itu `COPY package.json` + `RUN npm ci` diletakkan sebelum `COPY . .`.

`.dockerignore` mencegah `node_modules`, `.git`, dan `.env` ikut ke dalam konteks build.

### Praktik

1. Siapkan aplikasi Node.js minimal.

```bash
mkdir -p ~/apps/node-api && cd ~/apps/node-api
cat > package.json <<'JSON'
{
  "name": "node-api",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^4.19.2" }
}
JSON
cat > server.js <<'JS'
const express = require('express');
const app = express();
app.get('/', (req, res) => res.json({ ok: true, host: req.hostname }));
app.get('/health', (req, res) => res.status(200).send('ok'));
app.listen(3000, '0.0.0.0', () => console.log('listen 3000'));
JS
cat > .dockerignore <<'IGN'
node_modules
npm-debug.log
.git
.env
Dockerfile
IGN
```

2. Dockerfile pertama.

```dockerfile
FROM node:22-alpine

WORKDIR /app

# Dependensi dulu supaya layer ini jarang berubah
COPY package.json ./
RUN npm install --omit=dev

# Kode aplikasi paling akhir karena paling sering berubah
COPY server.js ./

ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "server.js"]
```

3. Build dan jalankan.

```bash
docker build -t node-api:1.0.0 .
docker images node-api
docker run -d --name node-api -p 127.0.0.1:3000:3000 node-api:1.0.0
docker logs node-api
curl -s http://127.0.0.1:3000/health
```

4. Buktikan cache bekerja.

```bash
touch server.js              # ubah kode saja
docker build -t node-api:1.0.1 .   # langkah npm install memakai cache
```

5. `CMD` vs `ENTRYPOINT`.

```bash
docker run --rm node-api:1.0.0 node -e "console.log('argumen menimpa CMD')"
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Build sangat lambat setiap kali | Urutan `COPY` salah, kode disalin sebelum dependensi | Salin manifest dulu, baru kode |
| `Cannot find module` | `npm install` tidak dijalankan atau path salah | Cek `WORKDIR` dan `COPY` |
| Image besar tak wajar | `node_modules` host ikut tersalin | Tambahkan `.dockerignore` |
| Container exit 1 | Perintah `CMD` salah atau port bentrok | `docker logs nama`, jalankan `CMD` manual dengan `--entrypoint sh` |

### Latihan

1. Build image, jalankan, dan ubah `server.js`; buktikan langkah `npm install` memakai cache.
2. Bandingkan ukuran image sebelum dan sesudah `node:22` diganti `node:22-alpine`.

**Kriteria selesai:** `docker build` menghasilkan image di bawah 200 MB dan `/health` membalas `ok`.

### Cek pemahaman

1. Kenapa dependensi disalin sebelum kode? — Agar cache layer dependensi tidak invalid setiap kali kode berubah.
2. Apa fungsi `.dockerignore`? — Mengecilkan konteks build dan mencegah file sensitif ikut masuk.
3. Beda `CMD` dan `ENTRYPOINT`? — `CMD` bisa ditimpa penuh; `ENTRYPOINT` tetap dan argumen `run` ditambahkan setelahnya.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tulis Dockerfile untuk aplikasimu + `.dockerignore`, build dua kali, dan tunjukkan layer mana yang memakai cache.

**Siapkan:** Ukur waktu build pertama dan kedua.

**Bawa bukti:** Log build kedua (baris `CACHED`) + selisih waktu build + ukuran image.

Dibahas di awal **Pelajaran 3.5** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.5 — Image kecil dan aman

### Tujuan

Menghasilkan image produksi: multi-stage, non-root, versi ter-pin, tanpa rahasia.

### Konsep

Multi-stage build memakai beberapa `FROM`: satu tahap untuk membangun (compiler, dev dependencies, bundler), satu tahap akhir yang **hanya menyalin hasilnya** memakai `COPY --from=<tahap>`. Tahap build tetap tertinggal di cache dan tidak masuk image akhir, sehingga image lebih kecil dan permukaan serangan menyusut. Tahap build yang tidak pernah di-`COPY --from` adalah tahap mati: ia hanya memboroskan waktu build.

Menjalankan sebagai `root` di dalam container berarti bug di aplikasi bisa berujung pada akses root ke host (terutama bila ada volume yang di-mount). Pakai `USER` non-root.

Rahasia tidak boleh ada di image: layer image bisa dibaca siapa pun yang punya image-nya. Rahasia diberikan saat runtime lewat environment variable atau file yang di-mount.

### Praktik

1. Dockerfile multi-stage untuk aplikasi Node.

```dockerfile
# ---- tahap build: dependensi (dan compiler, bila ada) ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev     # pakai `npm ci` bila package-lock.json sudah di-commit

# ---- tahap runtime: hanya hasil dari tahap build ----
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

COPY --from=build /app/node_modules ./node_modules   # inilah yang membuat tahap build berguna
COPY package.json server.js ./

# Jalankan sebagai user non-root bawaan image node
USER node

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "server.js"]
```

2. Verifikasi non-root dan ukuran.

```bash
docker build -t node-api:2.0.0 .
docker run -d --name node-api2 -p 127.0.0.1:3001:3000 node-api:2.0.0
docker exec node-api2 whoami          # harus: node
docker images node-api --format '{{.Repository}}:{{.Tag}} {{.Size}}'
docker inspect -f '{{.State.Health.Status}}' node-api2
```

3. Pin versi, jangan `latest`.

```dockerfile
FROM node:22.11-alpine3.20
```

4. Jangan pernah begini:

```dockerfile
# SALAH: rahasia ikut tersimpan di layer image
ENV DB_PASSWORD=rahasia123
RUN curl -H "Authorization: Bearer token-rahasia" https://internal/api
```

Benar: lewat `--env-file` atau secret Compose saat runtime (pelajaran 3.6 dan 3.8).

5. Scan kerentanan dasar (butuh akses jaringan; opsional).

```bash
docker scout quickview node-api:2.0.0 2>/dev/null || echo "scout tidak tersedia, lanjut"
# PERINGATAN: mount docker.sock memberi container itu akses setara root ke host.
# Pakai hanya untuk pemindaian sekali pakai, jangan di server produksi.
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL node-api:2.0.0 2>/dev/null | tail -20
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `permission denied` di runtime | `USER` non-root tidak punya izin tulis direktori kerja | Siapkan direktori dengan `--chown=node:node` saat `COPY` |
| Healthcheck selalu `unhealthy` | Alamat/port salah dari dalam container | Cek dengan `docker exec ... wget -qO- http://127.0.0.1:3000/health` |
| Image tetap besar | Tahap build ikut jadi image akhir | Pastikan `COPY --from=build` hanya menyalin artefak |
| `exec format error` | Image beda arsitektur (mis. Apple Silicon) | Build dengan `--platform linux/amd64` |

### Latihan

1. Ubah image ke multi-stage dan catat selisih ukurannya.
2. Tambahkan `HEALTHCHECK` dan buktikan statusnya `healthy` setelah 30 detik.

**Kriteria selesai:** `docker exec node-api2 whoami` menghasilkan `node`, bukan `root`.

### Cek pemahaman

1. Kenapa rahasia di `ENV`/`RUN` berbahaya? — Tersimpan permanen di layer image dan bisa dibaca siapa pun yang memegang image.
2. Apa manfaat `HEALTHCHECK`? — Memberi Docker cara menilai kesiapan aplikasi, dipakai Compose dan orchestrator.
3. Kenapa `latest` dihindari? — Tidak deterministik; deploy berikutnya bisa mengambil versi berbeda tanpa perubahan kode.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Ubah image ke multi-stage, jalankan sebagai user non-root, dan tambahkan `HEALTHCHECK`.

**Siapkan:** Bandingkan ukuran image sebelum dan sesudah.

**Bawa bukti:** `docker exec ... whoami` (harus non-root) + status health `healthy` + tabel ukuran sebelum/sesudah.

Dibahas di awal **Pelajaran 3.6** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.6 — Data dan konfigurasi

### Tujuan

Menyimpan data di luar container dan memisahkan konfigurasi dari image.

### Konsep

Filesystem container bersifat sementara: `docker rm` menghapus semuanya. Data harus berada di luar:

| Cara | Kapan dipakai |
| --- | --- |
| Named volume | Data yang dikelola Docker (database, upload). Direkomendasikan untuk produksi |
| Bind mount | File/direktori host yang ingin kamu lihat atau edit langsung (config, kode saat development) |

Konfigurasi: `--env-file` membaca variabel dari file. File `.env` **tidak** di-commit ke Git.

### Praktik

1. Bandingkan data di dalam vs di volume.

```bash
docker volume create appdata
docker run --rm -v appdata:/data alpine sh -c 'echo "tersimpan" > /data/isi.txt'
docker run --rm -v appdata:/data alpine cat /data/isi.txt   # tetap ada
```

2. Volume vs bind mount.

```bash
mkdir -p ~/apps/node-api/config
echo "LOG_LEVEL=info" > ~/apps/node-api/config/app.env

docker run -d --name cfg-demo \
  -v appdata:/data \
  -v ~/apps/node-api/config:/config:ro \
  alpine sleep 300
docker exec cfg-demo cat /config/app.env
docker exec cfg-demo sh -c 'echo baru >> /config/app.env'   # gagal: read-only
```

3. Inspeksi dan hapus volume.

```bash
docker volume ls
docker volume inspect appdata | head -15
docker volume rm appdata        # hanya bisa kalau tidak dipakai container
```

4. Env file.

```bash
cd ~/apps/node-api
cat > .env <<'ENV'
NODE_ENV=production
PORT=3000
DB_HOST=db
DB_PASSWORD=ganti-dengan-rahasia-kuat
ENV
chmod 600 .env
cat > .gitignore <<'IGN'
.env
node_modules/
IGN
```

```bash
docker run -d --name node-env --env-file .env -p 127.0.0.1:3002:3000 node-api:2.0.0
docker exec node-env env | grep -E 'NODE_ENV|DB_HOST'
```

5. Perbedaan `-v` (volume) dan `--mount` (lebih eksplisit):

```bash
docker run --rm --mount type=volume,src=appdata,dst=/data alpine ls /data
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Data "hilang" setelah `docker compose up` baru | Volume anonymous baru dibuat, bukan yang lama | Tentukan nama volume eksplisit |
| Bind mount kosong menimpa isi image | Mount ke direktori yang berisi kode di image | Mount hanya direktori data, bukan direktori kode |
| `Permission denied` menulis volume | UID di container beda dengan pemilik direktori host | Samakan UID/GID atau perbaiki `--chown` |
| `docker volume rm` gagal | Volume masih dipakai container | `docker ps -a \| grep volume`, hapus container dulu |

### Latihan

1. Buat volume `dbdata`, tulis file di dalamnya, hapus container, lalu buktikan file masih ada.
2. Jalankan container dengan `--env-file .env`, tampilkan variabelnya dari dalam container, dan pastikan `.env` ada di `.gitignore`.

**Kriteria selesai:** kamu bisa menjelaskan kapan memakai volume dan kapan bind mount.

### Cek pemahaman

1. Kenapa `.env` tidak boleh di-commit? — Berisi rahasia; riwayat Git sulit dibersihkan.
2. Apa yang terjadi jika bind mount menunjuk direktori kosong di host? — Direktori di container tampak kosong karena host yang menang.
3. Di mana data named volume disimpan? — Di bawah `/var/lib/docker/volumes/`, dikelola Docker.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat named volume untuk data, bind mount read-only untuk konfigurasi, dan jalankan container dengan `--env-file`.

**Siapkan:** Hapus container lalu jalankan lagi; pastikan data di volume tetap ada.

**Bawa bukti:** Isi file di volume setelah container dibuat ulang + `ls -l .env` (mode 600) + isi `.gitignore`.

Dibahas di awal **Pelajaran 3.7** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.7 — Network dan koneksi antar-container

### Tujuan

Menghubungkan container aplikasi ke database dan ke Nginx di host dengan konfigurasi minimal yang aman.

### Konsep

Mode network:

| Mode | Arti |
| --- | --- |
| `bridge` (default) | Container dapat IP internal; harus di-publish untuk diakses dari host |
| `host` | Container memakai network host langsung; port tidak perlu di-publish |
| `none` | Tanpa jaringan |

Docker menyediakan DNS internal: di network yang sama, container bisa saling memanggil lewat **nama service/container**, bukan IP. Ini alasan nama service di Compose penting.

Pola aman untuk server tunggal: container aplikasi hanya di-publish ke `127.0.0.1`, Nginx di host yang meneruskan. Database **tidak** di-publish sama sekali.

Alasan pola ini bukan sekadar kerapian: `ufw` **tidak** mengatur port yang di-publish Docker. Docker menulis aturan DNAT-nya sendiri sebelum rantai `INPUT`, jadi `-p 3000:3000` terbuka ke internet meski `ufw` bilang tertutup. `-p 127.0.0.1:3000:3000` menutup celah itu karena aturan DNAT-nya hanya berlaku untuk trafik loopback.

### Praktik

1. Network dan DNS internal.

```bash
docker network create appnet
docker run -d --name db --network appnet -e POSTGRES_PASSWORD=secret postgres:16-alpine
docker run --rm --network appnet alpine sh -c 'apk add --no-cache postgresql-client >/dev/null && pg_isready -h db -U postgres'
```

2. Buktikan isolasi network.

```bash
docker run --rm alpine getent hosts db || echo "benar: db tidak terlihat dari network lain"
```

3. Publish port ke localhost saja.

```bash
docker run -d --name api --network appnet -p 127.0.0.1:3000:3000 node-api:2.0.0
sudo ss -tulpn | grep 3000       # harus 127.0.0.1:3000
```

4. Bersihkan.

```bash
docker rm -f db api
docker network rm appnet
docker network ls
```

5. Kalau container memang perlu mengakses service di host (misalnya Nginx di host), jangan pakai IP host yang berubah — gunakan gateway atau `host.docker.internal` (Docker Desktop) / `--add-host`.

```bash
docker run -d --name api-host --add-host host.docker.internal:host-gateway -p 127.0.0.1:3003:3000 node-api:2.0.0
docker exec api-host getent hosts host.docker.internal
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `could not translate host name "db"` | Container tidak berada di network yang sama | Tambahkan `--network appnet` ke keduanya |
| Port tidak bisa diakses dari luar | Publish ke `127.0.0.1` | Sengaja; gunakan reverse proxy Nginx |
| `bind: address already in use` | Port host dipakai proses lain | `sudo ss -tulpn \| grep :PORT` |

### Latihan

1. Jalankan Postgres dan satu container klien di network yang sama; buktikan koneksi lewat nama `db`.
2. Buktikan database **tidak** bisa diakses dari host (`nc -zv 127.0.0.1 5432` harus gagal).

**Kriteria selesai:** kamu bisa menjelaskan kenapa nama service dipakai, bukan IP, dan kenapa database tidak dipublish.

### Cek pemahaman

1. Kenapa IP container tidak boleh di-hardcode? — Berubah setiap container dibuat ulang.
2. Apa fungsi `host-gateway`? — Memberi alias agar container bisa menghubungi IP host.
3. Kenapa `EXPOSE` saja tidak cukup? — `EXPOSE` hanya dokumentasi; akses dari host butuh `-p`.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat network `appnet`, jalankan Postgres dan satu container klien di dalamnya, lalu buktikan Postgres tidak bisa diakses dari host.

**Siapkan:** Catat nama service yang kamu pakai untuk koneksi, bukan IP.

**Bawa bukti:** Output `getent hosts db` dari container klien + `nc` dari host yang harus gagal + `docker network inspect`.

Dibahas di awal **Pelajaran 3.8** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 3.8 — Docker Compose

### Tujuan

Menjalankan aplikasi, database, dan konfigurasinya dengan satu file dan satu perintah.

### Konsep

Compose mendeskripsikan beberapa service dalam satu `compose.yaml`. Setiap service bisa punya image, port, volume, environment, dan `healthcheck`. `depends_on` dengan `condition: service_healthy` menunggu database siap sebelum aplikasi mulai.

`restart: unless-stopped` membuat container hidup lagi setelah reboot, kecuali kamu sengaja menghentikannya.

### Praktik

1. File Compose lengkap.

```bash
cd ~/apps/node-api
cat > compose.yaml <<'YAML'
services:
  api:
    build: .
    image: node-api:2.0.0
    restart: unless-stopped
    env_file: .env
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: appdb
      DB_USER: appuser
    ports:
      - "127.0.0.1:3000:3000"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 3s
      retries: 3
    networks: [appnet]

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD wajib diisi}
    volumes:
      - dbdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [appnet]

volumes:
  dbdata:

networks:
  appnet:
    driver: bridge
YAML
```

Catatan: pastikan `DB_PASSWORD` ada di `.env`, dan `DB_USER`/`DB_NAME` di `.env` konsisten dengan nilai di atas.

2. Jalankan dan periksa.

```bash
docker compose config --quiet && echo "YAML valid"
docker compose up -d
docker compose ps
docker compose logs -f --tail 30 api
curl -s http://127.0.0.1:3000/health
```

3. Perintah harian.

```bash
docker compose exec api sh
docker compose restart api
docker compose build --no-cache api
docker compose pull
docker compose down              # container+network dihapus, volume tetap
docker compose down -v           # PERINGATAN: volume ikut dihapus (data hilang)
```

4. Update aplikasi tanpa menyentuh database.

```bash
docker compose build api
docker compose up -d api        # recreate hanya service api
docker compose ps
```

5. Sambungkan ke Nginx di host (jembatan ke modul 2).

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

6. Autostart setelah reboot.

```bash
sudo systemctl is-enabled docker
sudo reboot
# setelah server hidup:
docker compose ps
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `variable is not set` / `.env` tidak terbaca | Compose membaca `.env` di direktori yang sama dengan `compose.yaml` | `cd` ke direktori itu, atau gunakan `--env-file` |
| API crash: `ECONNREFUSED db` | Aplikasi start sebelum database siap | Pakai `depends_on: condition: service_healthy` + retry di aplikasi |
| `dbdata` kosong setelah `down -v` | Volume sengaja dihapus | Pulihkan dari backup; `down -v` hanya untuk data yang boleh hilang |
| Perubahan kode tidak berefek | Image lama dipakai, `build` tidak dijalankan | `docker compose build api && docker compose up -d api` |

### Latihan

1. Jalankan stack, hentikan dengan `docker compose down`, jalankan lagi, dan buktikan data Postgres masih ada.
2. Tambahkan service ketiga (Redis atau Nginx) ke `compose.yaml` dan hubungkan ke network yang sama.

**Kriteria selesai:** `docker compose ps` menunjukkan `db` sebagai `healthy`, dan API bisa diakses via `127.0.0.1:3000` sementara database tidak terbuka ke host.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tulis `compose.yaml` berisi aplikasi + Postgres bervolume dengan healthcheck dan `restart: unless-stopped`.

**Siapkan:** Uji siklus `up -d` → `down` → `up -d` dan pastikan data database bertahan.

**Bawa bukti:** `docker compose ps` (semua sehat) + jumlah baris tabel sebelum dan sesudah siklus + log startup aplikasi.

Dibahas di awal **Pelajaran 4.1** — 10 menit pertama, sebelum materi baru.

---

## Proyek mini modul

Ubah latihan modul ini menjadi stack yang layak produksi:

1. Dockerfile multi-stage, user `node`, `HEALTHCHECK`, tag versi ter-pin.
2. `compose.yaml` berisi `api` + `db` (Postgres bervolume) + `redis` opsional, dengan healthcheck dan `restart: unless-stopped`.
3. Semua rahasia di `.env` (mode 600, ada di `.gitignore`), image tidak memuat rahasia apa pun.
4. `api` hanya publish ke `127.0.0.1:3000`; `db` tidak publish.
5. Tulis `catatan-docker.md`: cara build, cara up/down, cara melihat log, cara mencadangkan volume database (`docker compose exec db pg_dump ...`).

**Kriteria selesai:** server baru bisa menjalankan stack ini hanya dengan `git clone`, `docker compose up -d`, dan tidak ada port database yang terbuka dari luar.

## Cheat sheet

| Kebutuhan | Perintah |
| --- | --- |
| Build image | `docker build -t nama:tag .` |
| Jalankan container | `docker run -d --name x -p 127.0.0.1:3000:3000 --env-file .env nama:tag` |
| Lihat log | `docker logs -f --tail 50 nama` |
| Masuk container | `docker exec -it nama sh` |
| Status & resource | `docker ps -a`, `docker stats --no-stream` |
| Detail container | `docker inspect -f '{{.State.Status}}' nama` |
| Compose naik/turun | `docker compose up -d`, `docker compose down` |
| Compose log | `docker compose logs -f --tail 50` |
| Bersihkan aman | `docker system df` lalu `docker image prune` |
| Ukuran image | `docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}'` |

## Referensi

- Docker Engine di Ubuntu — https://docs.docker.com/engine/install/ubuntu/
- Dockerfile reference — https://docs.docker.com/reference/dockerfile/
- Docker Compose file reference — https://docs.docker.com/reference/compose-file/
- Docker volumes — https://docs.docker.com/engine/storage/volumes/
- Docker networking — https://docs.docker.com/engine/network/
- Node.js image resmi — https://hub.docker.com/_/node
- Postgres image resmi — https://hub.docker.com/_/postgres
