# Modul 0 — Persiapan: VPS dan Domain

## Ringkasan modul

Inilah titik nol kursus: **belum ada server dan belum ada domain**. Dua pelajaran di modul ini menghasilkan satu hal sederhana tapi wajib — sebuah server yang bisa kamu SSH dan sebuah nama domain yang mengarah ke server itu.

Semua modul berikutnya bergantung pada dua hal ini. Kalau `ssh` belum bisa masuk, atau `dig` belum mengembalikan IP servermu, jangan lanjut dulu.

## Hasil pembelajaran

- Membuat instance VPS Ubuntu 24.04 LTS dan mencatat IP, user awal, serta port SSH.
- Login pertama lewat SSH, memverifikasi CPU/RAM/disk, dan menambah swap pada server ber-RAM kecil.
- Mengenali console/VNC penyedia sebagai jalur darurat saat SSH terkunci.
- Membeli domain dan mengarahkannya ke IP server lewat record A.
- Membuktikan DNS sudah menyebar dengan `dig`, dan tahu kenapa itu syarat sebelum meminta sertifikat HTTPS.

## Pelajaran 0.1 — Membuat VPS dari nol

### Tujuan

Punya satu server Linux yang bisa di-SSH, dari kondisi belum punya apa-apa: memilih penyedia, membuat instance, login pertama, dan memastikan sumber dayanya cukup.

### Konsep

**VPS (Virtual Private Server)** adalah komputer sewaan di pusat data: kamu dapat akses root, IP publik, dan tanggung jawab penuh atas isinya. Berbeda dari shared hosting yang membatasi, di VPS kamu yang mengatur semuanya — termasuk kalau salah konfigurasi sampai server tidak bisa diakses.

Yang perlu diputuskan sebelum membuat instance:

| Pilihan | Rekomendasi untuk kursus ini |
| --- | --- |
| Penyedia | Yang punya region Jakarta/Singapore: IDCloudHost, Biznet Gio, DigitalOcean, Vultr, Hetzner, AWS Lightsail |
| Sistem operasi | Ubuntu 24.04 LTS (LTS = didukung 5 tahun) |
| Spesifikasi | Minimum 1 vCPU / 1 GB RAM / 25 GB SSD; disarankan 2 vCPU / 2 GB |
| Alamat | IPv4 publik (wajib); IPv6 hanya bonus |
| Autentikasi | SSH key bila bisa; password hanya untuk login pertama |
| Backup | Snapshot otomatis bila tersedia dan murah |

Tiga hal yang harus kamu catat setelah instance dibuat: **IP publik**, **user awal** (root, ubuntu, atau admin — tergantung penyedia, biasanya dikirim lewat email), dan **port SSH** (default 22).

Catatan biaya: instance yang dimatikan (`shutdown`) umumnya **tetap ditagih**, karena disk dan IP-nya masih dipesan. Yang menghentikan tagihan adalah menghapus instance (destroy). Snapshot sebelum eksperimen berbahaya jauh lebih murah daripada kehilangan server.

### Praktik

1. Buat akun dan instance di panel penyedia.

- Pilih region terdekat (Jakarta/Singapore) — latensi ke pengguna Indonesia jauh lebih baik.
- Pilih image **Ubuntu 24.04 LTS x86_64**, bukan versi minimal/container.
- Ukuran: mulai dari paket terkecil yang masuk akal; 1 GB RAM cukup untuk Nginx + satu container kecil, 2 GB lebih lega.
- Kalau panel meminta SSH key: tempel public key dari laptop (cara membuatnya ada di pelajaran 1.4). Kalau hanya password, lanjut dengan password yang dikirim ke email.
- Beri nama host yang jelas, misalnya `devops-web-01`.

2. Catat dan amankan data akses.

```text
IP publik   : 203.0.113.10
User awal   : root (atau: ubuntu)
Port SSH    : 22
Kata sandi  : (dari email penyedia)
Panel/console: https://panel.penyedia.example
```

Simpan kata sandi awal di pengelola kata sandi, bukan di catatan biasa.

3. Buka terminal di laptop dan tes jangkauan server.

```bash
# macOS/Linux (Terminal) atau Windows 10+ (PowerShell). Cek dulu klien SSH ada:
ssh -V

# Tes port terbuka sebelum login (Linux/macOS):
nc -zv 203.0.113.10 22

# Login pertama:
ssh root@203.0.113.10
# atau, kalau user awalnya ubuntu:
ssh ubuntu@203.0.113.10
```

Di Windows tanpa klien SSH, pakai PuTTY atau aktifkan OpenSSH Client lewat *Settings → Apps → Optional features*.

4. Kenali mesinnya dan pastikan sumber daya sesuai iklan paket.

```bash
cat /etc/os-release | head -2      # harus Ubuntu 24.04.x LTS
nproc                              # jumlah CPU
free -h                            # RAM dan swap
df -h /                            # ruang disk
curl -s https://ifconfig.me; echo  # IP publik dari sudut pandang internet
```

5. Tambah swap bila RAM hanya 1 GB.

Server kecil tanpa swap bisa membunuh proses sendiri (OOM killer) saat build image Docker. Swap file 2 GB membuat build lebih aman.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# supaya aktif lagi setelah reboot
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

free -h                            # kolom Swap harus terisi
swapon --show
```

6. Amankan akses awal — lanjut ke pelajaran 1.4 untuk detailnya.

```bash
passwd                                   # wajib bila login memakai password
sudo apt update && sudo apt upgrade -y
```

Kalau penyedia menyediakan SSH key saat membuat instance, jangan menonaktifkan autentikasi password sebelum key-mu terbukti bisa login.

7. Pelajari panel penyedia — ini jalur penyelamatanmu.

- **Console / VNC:** masuk ke server lewat browser walau SSH sudah terkunci. Ini satu-satunya cara pulih kalau aturan firewall salah.
- **Firewall penyedia / security group:** beberapa penyedia punya firewall sendiri di depan server. Kalau port 80 tidak bisa diakses, periksa **dua tempat**: `ufw` di dalam server dan firewall di panel.
- **Snapshot/backup:** lakukan sebelum perubahan besar.
- **Reboot/hard reboot:** `reboot` dari SSH dulu; hard reboot hanya kalau server benar-benar menggantung.
- **Resize:** menaikkan RAM/CPU biasanya butuh reboot; menurunkan biasanya butuh rebuild.

8. Siapkan kebiasaan kerja.

- Jangan bekerja sebagai `root` sehari-hari; buat user `deploy` (pelajaran 1.4).
- Jangan pakai IP sebagai satu-satunya catatan; siapkan domain (dipakai mulai modul 2).
- Kalau instance ini hanya untuk latihan, matikan/hapus setelah selesai supaya tagihan tidak jalan terus.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Connection timed out` | Port 22 diblokir firewall penyedia, IP salah, atau instance belum selesai booting | Cek IP di panel, tunggu 1–2 menit, cek firewall/security group penyedia |
| `Permission denied (publickey)` | User awal salah (mis. memakai `root` padahal user `ubuntu`) atau key belum ditempel | Coba user lain sesuai email penyedia; cek console untuk user yang benar |
| `Connection refused` | SSH belum jalan atau port bukan 22 | Buka console di panel, cek `systemctl status ssh` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | IP pernah dipakai server lain, host key tersimpan di laptop | `ssh-keygen -R 203.0.113.10` lalu login ulang, pastikan IP memang milikmu |
| Instance booting tapi tidak bisa login | Image belum selesai provisioning atau cloud-init masih jalan | Tunggu, jalankan `cloud-init status --wait` dari console |
| Tagihan tetap jalan walau server dimatikan | Instance masih ada (disk dan IP masih dipesan) | Snapshot lalu destroy instance bila tidak dipakai |

### Latihan

1. Buat satu VPS Ubuntu 24.04, login pertama, dan catat IP, user awal, serta hasil `nproc`, `free -h`, `df -h /` di `catatan-server.md`.
2. Pasang swap 2 GB (bila RAM ≤ 2 GB), reboot, dan buktikan swap aktif setelah reboot.
3. Buka console/VNC dari panel dan pastikan kamu bisa masuk tanpa SSH. Ini latihan jalur darurat.

**Kriteria selesai:** kamu bisa membuat dan menghapus satu instance VPS sesuai kebutuhan, punya catatan IP + user, dan tahu cara masuk lewat console kalau SSH terkunci.

### Cek pemahaman

1. Kenapa instance yang dimatikan masih ditagih? — Disk dan IP masih dialokasikan; tagihan berhenti saat instance dihapus.
2. Di mana saja firewall harus diperiksa? — Dua lapis: `ufw` di dalam server dan firewall/security group milik penyedia.
3. Kenapa server 1 GB RAM sebaiknya punya swap? — Build atau proses besar bisa memicu OOM killer yang mematikan service; swap memberi ruang cadangan.
4. Apa fungsi console/VNC di panel? — Jalur masuk cadangan saat SSH tidak bisa dipakai, misalnya setelah salah konfigurasi firewall.

---

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat satu VPS Ubuntu 24.04 sesuai paket yang kamu pilih, lalu tulis `catatan-server.md` berisi IP, user awal, port SSH, penyedia, region, dan tanggal perpanjangan.

**Siapkan:** Pasang swap 2 GB bila RAM ≤ 2 GB, reboot, dan pastikan swap masih aktif.

**Bawa bukti:** `free -h`, `nproc`, `df -h /` dari server + isi `catatan-server.md`. Sebutkan satu kesulitan yang kamu temui di panel penyedia.

Dibahas di awal **Pelajaran 0.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 0.2 — Domain dan DNS

### Tujuan

Membeli satu domain, mengarahkannya ke IP server, dan membuktikan DNS-nya sudah menyebar sebelum dipakai HTTPS.

### Konsep

**Domain** adalah nama sewaan tahunan (mis. `example.com`). **DNS** adalah buku telepon yang menerjemahkan nama itu jadi alamat IP. Kamu tidak "memasang" domain di server — kamu menambahkan catatan (record) di DNS yang menunjuk ke IP server.

Record yang dipakai kursus ini:

| Record | Fungsi | Contoh |
| --- | --- | --- |
| `A` | Nama → IPv4 | `app.example.com` → `203.0.113.10` |
| `AAAA` | Nama → IPv6 | opsional |
| `CNAME` | Nama → nama lain | `www` → `example.com` |
| `NS` | Server nama yang berwenang | dari penyedia DNS |
| `TXT` | Verifikasi & email (SPF/DKIM) | tidak dipakai di sini |

Istilah penting:

- **Registrar** — tempat membeli domain (Namecheap, Cloudflare Registrar, Rumahweb, Niagahoster, Porkbun).
- **Nameserver (NS)** — server yang menyimpan record-mu. Bisa di registrar, bisa dipindah ke penyedia DNS lain (Cloudflare, Route 53).
- **TTL** — berapa lama record di-cache. TTL 300 detik mempercepat perubahan saat menyiapkan; TTL 3600 lebih hemat query.
- **Subdomain** — `app.example.com` untuk aplikasi, `example.com` untuk halaman utama.

Kalau kamu memakai proxy CDN di depan domain (mis. awan oranye Cloudflare), validasi HTTP-01 Let's Encrypt bisa gagal karena challenge dijawab oleh proxy. Untuk kursus ini, matikan proxy (awan abu-abu / DNS only) sampai sertifikat terbit.

Menunggu DNS menyebar tidak bisa dipercepat dengan me-restart server. Yang bisa dilakukan: menurunkan TTL **sebelum** perubahan, dan memverifikasi lewat beberapa resolver.

### Praktik

1. Beli domain.

- Pilih nama pendek yang mudah diingat; TLD `.com`/`.id`/`.dev` semuanya bisa.
- Aktifkan auto-renew dan proteksi privasi (WHOIS privacy) bila tersedia.
- **Jangan** beli hosting dari registrar kalau kamu sudah punya VPS — cukup domainnya.

2. Tentukan di mana DNS dikelola.

- Kalau memakai NS registrar, langsung edit record di panel registrar.
- Kalau memindahkan NS ke Cloudflare: tambahkan domain di Cloudflare, salin dua NS yang diberikan, tempel di panel registrar, lalu tunggu sampai status domain "Active" di Cloudflare.

3. Tambahkan record A.

```text
Type : A
Name : app            (atau @ untuk example.com)
Value: 203.0.113.10   (IP VPS-mu)
TTL  : 300
Proxy: DNS only (awan abu-abu) bila memakai Cloudflare
```

Tambahkan juga `www` (CNAME ke `example.com`) dan `@` (A ke IP yang sama) bila ingin halaman utama ikut jalan.

4. Verifikasi dari laptop.

```bash
dig +short app.example.com          # harus IP server
dig NS example.com +short           # nameserver yang aktif
dig @1.1.1.1 +short app.example.com # tanya resolver publik
dig @8.8.8.8 +short app.example.com
nslookup app.example.com            # alternatif di Windows
```

Kalau `dig` kosong, belum ada jawaban. Kalau mengembalikan IP lama, cache TTL masih berlaku.

5. Cek dari sisi server juga.

```bash
getent hosts app.example.com
curl -sI http://app.example.com | head -3    # 502/404 masih normal: Nginx belum diatur
```

Koneksi berhasil (walau balasannya error HTTP) berarti DNS + jaringan sudah benar. `Could not resolve host` berarti DNS belum siap.

6. Catat semuanya.

```text
Domain      : example.com
Subdomain   : app.example.com
Registrar   : (nama)
DNS dikelola: (registrar / Cloudflare)
Record A    : app -> 203.0.113.10
Tanggal beli: 2026-11-11
Perpanjangan: 2027-11-11
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `dig` tidak mengembalikan apa pun | Record belum dibuat, salah tipe, atau salah nama (`app` vs `app.example.com`) | Isi `Name` hanya label-nya (`app`), kecuali panel meminta nama penuh |
| Masih IP lama setelah diubah | Cache TTL | Tunggu sesuai TTL; uji dengan `dig @1.1.1.1`, jangan hanya resolver lokal |
| Domain "Active" tapi record tidak jalan | NS registrar belum benar-benar diganti | `dig NS example.com +short`, bandingkan dengan NS dari penyedia DNS |
| Situs balas halaman penyedia/proxy | Proxy CDN aktif | Set ke DNS only selama menyiapkan dan menerbitkan sertifikat |
| `www` tidak jalan | Belum ada record untuk `www` | Tambahkan CNAME `www` → `example.com`, atau A ke IP yang sama |
| Certbot gagal padahal domain benar | Port 80 tertutup atau proxy aktif | Buka 80, matikan proxy, coba lagi setelah rate limit reda |

### Latihan

1. Beli satu domain, tambahkan record A untuk `app` ke IP VPS-mu, dan buktikan dengan `dig @1.1.1.1 +short app.example.com`.
2. Tambahkan subdomain kedua (mis. `staging`) dan pastikan keduanya mengarah ke IP yang sama.
3. Tunjukkan bahwa domain **belum** melayani situs dengan benar sebelum Nginx diatur: `curl -sI http://app.example.com` boleh gagal, tapi DNS harus sudah benar.

**Kriteria selesai:** `dig +short app.example.com` konsisten mengembalikan IP server dari dua resolver berbeda.

### Cek pemahaman

1. Apa beda registrar dan penyedia DNS? — Registrar tempat membeli/memperpanjang domain; penyedia DNS tempat record disimpan (bisa sama, bisa dipindah lewat NS).
2. Kenapa TTL diturunkan sebelum perubahan besar? — Perubahan tersebar lebih cepat karena cache lama berumur pendek.
3. Kenapa proxy CDN sebaiknya dimatikan saat menerbitkan sertifikat? — Validasi HTTP-01 dijawab proxy, bukan server, sehingga Let's Encrypt tidak menemukan challenge.
4. Bagaimana membedakan masalah DNS dan masalah Nginx? — `Could not resolve host` = DNS; balasan HTTP (502/404) berarti DNS sudah benar dan masalahnya di server.

---

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Beli satu domain, tambahkan record A untuk `app` dan `www` ke IP VPS-mu, lalu catat di mana DNS dikelola.

**Siapkan:** Uji `dig` dari dua resolver berbeda (`1.1.1.1` dan `8.8.8.8`).

**Bawa bukti:** Output `dig +short app.example.com` dari dua resolver + tangkapan layar panel DNS. Jelaskan kenapa `curl` masih gagal sebelum Nginx diatur.

Dibahas di awal **Pelajaran 1.1** — 10 menit pertama, sebelum materi baru.

---

## Proyek mini modul

Sebelum masuk Modul 1, pastikan semuanya beres:

1. Satu VPS Ubuntu 24.04 LTS hidup, bisa di-SSH, dan kamu tahu user awalnya.
2. Swap aktif (bila RAM ≤ 2 GB) dan bertahan setelah reboot.
3. Console/VNC panel penyedia sudah pernah kamu buka — jalur darurat teruji.
4. Satu domain mengarah ke IP server: `dig +short example.com` mengembalikan IP itu.
5. `catatan-server.md` berisi: IP, user awal, port SSH, penyedia + region, tanggal dibuat, dan tanggal perpanjangan/akhir sewa.

**Kriteria selesai:** dari laptop, `ssh user@IP` berhasil dan `dig +short example.com` menunjuk ke IP yang sama.

## Perintah penting

| Kebutuhan | Perintah |
| --- | --- |
| Cek klien SSH | `ssh -V` |
| Tes port SSH | `nc -zv 203.0.113.10 22` |
| Login | `ssh root@203.0.113.10` atau `ssh ubuntu@203.0.113.10` |
| Lihat resource | `nproc`, `free -h`, `df -h /` |
| Pasang swap | `fallocate` → `chmod 600` → `mkswap` → `swapon` |
| Cek DNS | `dig +short example.com`, `dig NS example.com` |
| Cek propagasi publik | `dig @1.1.1.1 +short example.com` |
| Hapus host key lama | `ssh-keygen -R 203.0.113.10` |

## Referensi

- Ubuntu Server documentation — https://documentation.ubuntu.com/server/
- DigitalOcean: initial server setup with Ubuntu — https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu
- Cloudflare DNS records — https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- Let's Encrypt challenge types (HTTP-01 butuh port 80 & DNS benar) — https://letsencrypt.org/docs/challenge-types/
- ICANN: memilih dan mengelola domain — https://www.icann.org/resources/pages/domain-name-basics-2013-05-24-en
