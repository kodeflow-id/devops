# DevOps dari Nol

🙂 **Umur:** 16+
🎓 **Tingkat kesulitan:** Pemula — belum pernah menyentuh server pun tidak masalah
🕐 **Durasi kursus:** 41 pertemuan / 6 modul — pertemuan 1 khusus perkenalan, materi mulai pertemuan 2
⏱ **Durasi kelas:** 90 menit
🛠 **Program untuk dipelajari:** Terminal & SSH, VS Code, Git & GitHub, Docker, Nginx.

## ⭐ Deskripsi

Kursus ini mengajarkan cara membawa aplikasi dari laptop ke server yang bisa diakses publik — tanpa asumsi kamu pernah pegang server sebelumnya.

Peserta mulai dari membeli VPS dan domain, masuk lewat SSH, lalu kerja di terminal: pindah direktori, atur izin file, buat user, jalankan service. Setelah itu memasang Nginx, memahami cara kerja reverse proxy, mengunci situs dengan HTTPS, lalu mengemas aplikasi ke dalam container Docker. Terakhir, proses build dan deploy diotomatiskan dengan GitHub Actions — setiap push ke branch `main` langsung tayang di server.

Setiap modul berakhir dengan praktik di VPS sungguhan. Tidak ada slide teori tanpa perintah yang dijalankan.

## 🎯 Tujuan kursus

Kursus ini membentuk peserta yang bisa berdiri sendiri: menyiapkan, menjalankan, dan merawat satu server Linux yang melayani aplikasi web — mulai dari instalasi, Nginx sebagai reverse proxy, kontainerisasi dengan Docker, sampai deploy otomatis lewat CI/CD. Saat aplikasi mati (502, container berhenti, deploy gagal), peserta belajar memeriksa satu per satu sampai ketemu penyebabnya, bukan menebak-nebak.

---

## ✅ Hasil kursus

Setelah lulus, peserta bisa menyiapkan VPS dari kosong sampai aplikasinya tayang lewat HTTPS, dan setiap rilis berikutnya otomatis — tanpa perlu SSH ke server lagi.

### ✅ Pengetahuan objektif

- Peserta tahu struktur direktori Linux dan fungsi direktori utama (`/etc`, `/var/log`, `/home`, `/opt`).
- Peserta paham model izin Unix (owner/group/other, baca/tulis/eksekusi) dan akibatnya di file maupun direktori.
- Peserta paham cara kerja HTTP request/response, status code, dan peran header di reverse proxy.
- Peserta paham beda image dan container, serta beda volume dan bind mount.
- Peserta tahu komponen GitHub Actions: workflow, event, job, step, runner.

### ✅ Pengetahuan konseptual

- Peserta bisa menjelaskan beda web server dan application server, dan kenapa Nginx dipasang di depan aplikasi.
- Peserta bisa menjelaskan kenapa TLS ditangani reverse proxy, bukan aplikasinya.
- Peserta bisa menjelaskan kenapa data di dalam container hilang saat container dihapus, dan bagaimana volume mencegahnya.
- Peserta bisa menjelaskan beda continuous integration, continuous delivery, dan continuous deployment.
- Peserta bisa menjelaskan kelebihan dan kekurangan container dibanding mesin virtual, termasuk kapan container bukan pilihan yang tepat.

### ✅ Pengetahuan prosedural

- Peserta bisa menyiapkan lingkungan dari nol: instance Ubuntu 24.04, swap, domain dengan record A yang terverifikasi, dan paham firewall-nya ada dua lapis (milik penyedia dan `ufw`).
- Peserta bisa mengamankan SSH: login pakai key, matikan login root, tutup port yang tidak dipakai.
- Peserta bisa memasang Nginx, menulis server block, menguji konfigurasi dengan `sudo nginx -t`, dan reload tanpa downtime.
- Peserta bisa memasang reverse proxy lengkap dengan `proxy_set_header`, dukungan WebSocket, dan HTTPS Let's Encrypt.
- Peserta bisa menulis Dockerfile multi-stage, membangun image, dan menjalankan aplikasi beserta database pakai Docker Compose.
- Peserta bisa menulis workflow GitHub Actions yang menjalankan test, membangun image, mendorongnya ke GHCR, lalu deploy ke VPS.
- Peserta bisa mundur ke versi image sebelumnya kalau deploy bermasalah.

### ✅ Pengetahuan meta-kognitif

- Peserta bisa memecah masalah "situs tidak bisa diakses" menjadi lapisan pemeriksaan: DNS, firewall, port, proses, Nginx, aplikasi, log.
- Peserta bisa membaca log (`journalctl`, `docker logs`, `/var/log/nginx/error.log`) dan mengambil informasi yang relevan sebelum mengubah konfigurasi.
- Peserta bisa menilai kesiapan rilis dengan checklist go-live, dan tahu tindakan mana yang berbahaya (`rm -rf`, `docker system prune`, `ufw enable` saat masih terhubung SSH).
- Peserta terbiasa mendokumentasikan perubahan konfigurasi dan mengembalikannya bila gagal.

---

## 🦾 Keterampilan yang diperoleh siswa selama kursus

### Hard-skills

**Linux**

- Terminal, shell, PATH, environment variable
- Struktur filesystem dan navigasi
- Redirect, pipe, pencarian teks, arsip
- User, grup, `sudo`, permission, `umask`, SSH key
- Proses, sinyal, `systemctl`, unit file systemd, `journalctl`
- `apt`, update keamanan, `ufw`, `ss`, DNS dasar

**Nginx**

- Struktur konfigurasi dan siklus `sudo nginx -t` → reload
- Server block, `server_name`, static site, `try_files`
- `location`, prioritas pencocokan, `root` vs `alias`, `rewrite`
- Reverse proxy: `proxy_pass`, header `Host`/`X-Forwarded-*`, WebSocket, timeout
- HTTPS Let's Encrypt, perpanjangan otomatis, HSTS
- `upstream`, load balancing, `limit_req`, basic auth, security header
- Diagnosis 502/504/403/413/404 dari log

**Docker**

- Image, container, registry, layer cache
- Dockerfile: `FROM`/`COPY`/`RUN`/`CMD`/`ENTRYPOINT`/`USER`
- Multi-stage build, image kecil, non-root
- Volume, bind mount, env file
- Network bridge, DNS antar-service, publish vs expose
- Docker Compose: service, healthcheck, `depends_on`, restart policy

**GitHub Actions**

- Workflow YAML, `on`, job, step, matrix, cache
- Secret, variable, environment, `GITHUB_TOKEN` dan `permissions`
- Build & push image ke GHCR dengan buildx
- Deploy via SSH, health check, rollback, concurrency

### Soft-skills

**Kemampuan berkomunikasi**

- Menulis laporan insiden singkat: gejala, dugaan, tindakan, hasil
- Menjelaskan keputusan teknis ke orang non-teknis

**Kerja tim**

- Membagi pekerjaan lewat branch dan pull request
- Review perubahan konfigurasi sebelum diterapkan ke server bersama

**Berpikir kritis**

- Membedakan gejala dan penyebab
- Menguji satu perubahan pada satu waktu, biar ketahuan mana yang bikin gagal

**Manajemen waktu dan proyek**

- Memecah tugas deploy menjadi langkah yang bisa diverifikasi
- Menyusun checklist rilis dan rollback sebelum eksekusi

**Kemandirian**

- Membaca dokumentasi resmi dan `man` page
- Mencari jawaban dari log, bukan dari tebakan

---

### Pertemuan 1: Perkenalan (belum masuk materi)

**Hasil:** Peserta saling kenal, paham gambaran besar kursus, dan pulang bawa akun GitHub aktif + sudah tahu mau beli VPS dan domain yang mana.

| Menit | Kegiatan |
| --- | --- |
| 0–15 | Kenalan: pengajar, peserta, satu-satu cerita latar belakang |
| 15–35 | Gambaran kursus: tujuan, 6 modul, alur pertemuan, sistem tugas & bukti, cara penilaian |
| 35–55 | Demo singkat: pengajar buka aplikasi yang sudah tayang di domain dengan HTTPS, lalu tunjukkan pipeline deploy jalan sendiri saat `git push` |
| 55–75 | Setup akun: buat akun GitHub, aktifkan 2FA, install VS Code; pengantar terminal bila ada waktu |
| 75–90 | Bahas kebutuhan teknis: pilih penyedia VPS, perkiraan biaya VPS + domain, kapan harus sudah beli (sebelum pertemuan 2 / Modul 0.1) |

**Tugas untuk pertemuan 2:** punya akun GitHub aktif (sudah diaktifkan 2FA) dan sudah putuskan mau beli VPS + domain yang mana. Pembeliannya sendiri dilakukan di Pelajaran 0.1 bareng pengajar.

## 📚 Rencana Kursus

### Alur tiap pertemuan (90 menit)

| Menit | Bagian |
| --- | --- |
| 0–10 | **Bahas tugas** pertemuan sebelumnya: tiap peserta menunjukkan bukti, dua kasus tersulit dibedah bersama |
| 10–70 | Materi + demo langsung oleh pengajar |
| 70–85 | Praktik terbimbing di VPS masing-masing |
| 85–90 | **Tugas baru** dijelaskan, termasuk bukti yang harus dibawa minggu depan |

Setiap pelajaran di modul 0–5 punya bagian **“Tugas untuk pertemuan berikutnya”** dengan tiga hal: apa yang dikerjakan, apa yang disiapkan, dan bukti apa yang dibawa. Tugas selalu pakai server peserta sendiri, bukan studi kasus fiktif.



### Modul 0: Persiapan — VPS dan Domain (Pelajaran 0.1–0.2)

**Hasil:** Peserta punya satu VPS Ubuntu 24.04 yang bisa di-SSH dan satu domain yang sudah mengarah ke IP server itu — syarat mutlak sebelum masuk modul berikutnya.

- **Pelajaran 0.1.** Membuat VPS dari nol. Pilih penyedia dan paket, buat instance Ubuntu 24.04, login pertama, cek CPU/RAM/disk, pasang swap, kenali console panel sebagai jalur darurat.
- **Pelajaran 0.2.** Domain dan DNS. Beli domain, pilih tempat mengelola DNS, tambahkan record A untuk subdomain aplikasi, verifikasi dengan `dig` dari beberapa resolver, dan pahami kenapa proxy CDN harus dimatikan saat menerbitkan sertifikat.

### Modul 1: Dasar Linux di Server (Pelajaran 1–8)

**Hasil:** Peserta menguasai server lewat terminal: navigasi dan izin file, user deploy, akses SSH yang dikunci ke key, service systemd sendiri, dan pengambilan kode dari GitHub.

- **Pelajaran 1.** Terminal dan shell. Prompt, PATH, `man`, history, variabel environment.
- **Pelajaran 2.** Filesystem. Hierarki direktori, path absolut dan relatif, `find`, symlink.
- **Pelajaran 3.** File dan teks. Redirect, pipe, `grep`, `sed`, `tar`, `curl`.
- **Pelajaran 4.** User, grup, dan permission. `sudo`, `chmod`, `chown`, `umask`, SSH key, hardening SSH.
- **Pelajaran 5.** Proses dan service. `ps`, sinyal, `systemctl`, unit file sendiri, `journalctl`.
- **Pelajaran 6.** Manajemen paket. `apt`, repository, update keamanan, `unattended-upgrades`.
- **Pelajaran 7.** Jaringan dan firewall. `ip`, `ss`, DNS, `dig`, `ufw` dengan aman.
- **Pelajaran 8.** Git di server. Clone, pull, `.gitignore`, SSH key GitHub, kenapa tidak mengedit langsung di server.

### Modul 2: Satu Pintu Masuk — Nginx dan Reverse Proxy (Pelajaran 9–16)

**Hasil:** Peserta mempublikasikan aplikasi lokal di `127.0.0.1:3000` menjadi `https://app.example.com` dengan redirect, header benar, dan rate limit.

- **Pelajaran 9.** HTTP dan web server. Request/response, status code, header, TLS sekilas.
- **Pelajaran 10.** Instalasi dan struktur Nginx. `sudo nginx -t`, sites-available/enabled, log.
- **Pelajaran 11.** Situs statis pertama. `root`, `index`, `try_files`, uji dengan `curl -I`.
- **Pelajaran 12.** Server block dan domain. `server_name`, default server, redirect www dan HTTP→HTTPS.
- **Pelajaran 13.** `location` dan routing. Prioritas, `root` vs `alias`, `rewrite`, halaman error kustom.
- **Pelajaran 14.** Reverse proxy. `proxy_pass`, `proxy_set_header`, WebSocket, timeout, buffering.
- **Pelajaran 15.** HTTPS Let's Encrypt. Certbot, renew otomatis, HSTS, diagnosis kegagalan cert.
- **Pelajaran 16.** Load balancing dan tuning. `upstream`, `limit_req`, basic auth, `client_max_body_size`, diagnosis 502/504/413.

### Modul 3: Bawa Aplikasi ke Mana Saja — Docker (Pelajaran 17–24)

**Hasil:** Peserta mengemas aplikasi menjadi image multi-stage non-root dan menjalankannya bersama database memakai Docker Compose.

- **Pelajaran 17.** Kenapa container. Image vs container vs registry, VM vs container, kapan tidak dipakai.
- **Pelajaran 18.** Instalasi dan verifikasi. docker-ce, `hello-world`, grup docker dan risikonya.
- **Pelajaran 19.** Image dan container. Opsi `run`, `logs`, `exec`, `inspect`, `stats`, pembersihan yang aman.
- **Pelajaran 20.** Dockerfile pertama. Layer cache, `.dockerignore`, `CMD` vs `ENTRYPOINT`.
- **Pelajaran 21.** Image kecil dan aman. Multi-stage, `USER` non-root, pin tag, secret tidak masuk image.
- **Pelajaran 22.** Data dan konfigurasi. Volume vs bind mount, env file, kenapa `.env` tidak di-commit.
- **Pelajaran 23.** Network. Custom bridge, DNS antar-container, publish vs expose, sambungan ke Nginx di host.
- **Pelajaran 24.** Docker Compose. App + Postgres bervolume, healthcheck, `depends_on`, restart policy.

### Modul 4: Rilis Tanpa Tangan — GitHub Actions (Pelajaran 25–32)

**Hasil:** Peserta punya pipeline yang menjalankan test, membangun image, mendorong ke GHCR, dan men-deploy ke VPS setiap push ke `main`.

- **Pelajaran 25.** CI/CD dan komponen Actions. Workflow, event, job, step, runner.
- **Pelajaran 26.** Workflow pertama. YAML, `on`, `runs-on`, `uses` vs `run`, membaca log.
- **Pelajaran 27.** Test dan lint otomatis. Cache dependency, matrix, status check di pull request.
- **Pelajaran 28.** Secret, variable, environment. `secrets` vs `vars`, `GITHUB_TOKEN`, masking, required reviewer.
- **Pelajaran 29.** Build dan push image. buildx, login GHCR, strategi tag, cache registry.
- **Pelajaran 30.** Deploy lewat SSH. Key khusus deploy, known_hosts, `compose pull && up -d`, health check.
- **Pelajaran 31.** Deploy aman. Rollback tag, smoke test, concurrency, hanya dari `main`.
- **Pelajaran 32.** Menaikkan level. Reusable workflow, composite action, `needs`, badge, notifikasi gagal, least privilege.

### Modul 5: Proyek Akhir — Deploy Nyata (Pelajaran 33–38)

**Hasil:** Peserta men-deploy satu aplikasi dari repo GitHub ke VPS dengan seluruh rantai: Linux, Docker, Nginx, HTTPS, dan CI/CD otomatis.

- **Pelajaran 33.** Menyiapkan server dan DNS. User deploy, firewall, A record, verifikasi akses.
- **Pelajaran 34.** Menyiapkan aplikasi dan Dockerfile. Build lokal, uji container, publish port ke localhost.
- **Pelajaran 35.** Nginx reverse proxy dan HTTPS. Server block, certbot, uji dari luar.
- **Pelajaran 36.** Pipeline CI/CD. Test, build, push GHCR, deploy, smoke test.
- **Pelajaran 37.** Go-live dan observability. Checklist rilis, log, disk, backup database, alert sederhana.
- **Pelajaran 38.** Simulasi insiden dan demo. Diagnosis 502, container restart loop, cert expired, disk penuh; presentasi proyek.

---

Kursus ini **bukan** kursus lanjutan. Tidak ada asumsi pengalaman server, Docker, atau CI/CD sebelumnya; istilah teknis dijelaskan saat pertama dipakai.

## 🧐 Target audiens kursus

Untuk pemula yang mau kerja di bidang infrastruktur, backend, atau platform, serta developer yang selama ini cuma menjalankan aplikasi di laptop dan ingin bisa deploy sendiri.

Kursus ini cocok untuk:

- **Mahasiswa dan lulusan baru** yang butuh portofolio deploy nyata, bukan hanya repo kode.
- **Developer junior** yang ingin memahami apa yang terjadi setelah `git push`.
- **Freelancer** yang harus memasang dan merawat server klien sendiri.
- **Peserta dari jalur Python/JavaScript** yang sudah bisa menulis aplikasi tetapi belum pernah menyentuh server Linux.

Kursus ini tidak mengajarkan pemrograman dari nol. Aplikasi contoh sudah disiapkan; fokusnya cara menjalankannya dengan benar di server.

## 🤓 Pengetahuan yang diperlukan untuk memasuki kursus

- Lancar pakai mouse dan keyboard.
- Bisa membuka tautan, mengatur tab browser, dan copy-paste perintah.
- Paham konsep file dan folder di OS mana pun.
- Pernah memakai Git dasar (`clone`, `commit`, `push`) — atau ikuti pengantar di modul 1.
- Tidak perlu pengalaman server, jaringan, atau Docker.

## 🖥 Persyaratan teknis

- Laptop atau PC dengan keyboard; Windows, macOS, atau Linux.
- Browser modern (Chrome, Firefox, atau Edge).
- Akun GitHub aktif.
- Akun penyedia VPS (IDCloudHost, Biznet Gio, DigitalOcean, Vultr, Hetzner, atau AWS Lightsail) dengan metode pembayaran aktif.
- VPS Ubuntu 24.04 LTS. Minimum 1 vCPU / 1 GB RAM; disarankan 2 vCPU / 2 GB RAM. Bila RAM ≤ 2 GB, siapkan swap 2 GB.
- Domain (opsional untuk latihan awal; bisa memakai IP publik) dan akses ke pengaturan DNS.
- Koneksi internet stabil untuk SSH dan `docker pull`.

## 📽 Contoh proyek

1. **API + database di balik reverse proxy.** Satu container API, satu container Postgres bervolume, Nginx menangani HTTPS, deploy otomatis dari GitHub Actions.
2. **Situs statis dengan pipeline rilis.** Build situs, unggah artefak, Nginx menyajikan hasil build, setiap push ke `main` tayang dalam hitungan menit.
3. **Aplikasi multi-container dengan worker.** Web, worker antrean, cache Redis, dan database — dijalankan dengan satu `docker compose up -d` dan di-deploy lewat pipeline yang sama.

## 📦 Isi paket

| Berkas | Isi |
| --- | --- |
| `curriculum/syllabus.md` | Dokumen ini |
| `lessons/00-persiapan.md` | Modul 0 — VPS dan domain (persiapan) |
| `lessons/01-linux.md` | Modul 1 — dasar Linux di server |
| `lessons/02-nginx-reverse-proxy.md` | Modul 2 — Nginx dan reverse proxy |
| `lessons/03-docker.md` | Modul 3 — Docker |
| `lessons/04-github-actions.md` | Modul 4 — GitHub Actions |
| `lessons/05-capstone.md` | Modul 5 — Proyek akhir |
| `assessment/bank-soal.md` | Bank soal dan rubrik penilaian |
