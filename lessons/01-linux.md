# Modul 1 — Linux Fundamentals

## Ringkasan modul

Modul 0 sudah menyiapkan server dan domain; sekarang saatnya menguasai servernya lewat terminal. Delapan pelajaran di sini adalah fondasi yang dipakai terus sampai modul 5, jadi jangan lompat.

Asumsi lingkungan: VPS **Ubuntu 24.04 LTS**, diakses lewat SSH sebagai user non-root yang punya `sudo`. Ganti `203.0.113.10` dengan IP servermu dan `example.com` dengan domainmu.

## Hasil pembelajaran

- Menjelaskan dan memakai struktur direktori Linux tanpa mencari-cari.
- Mengatur user, grup, dan permission sehingga aplikasi berjalan tanpa hak berlebih.
- Masuk server memakai SSH key, dengan login password dan login root dimatikan.
- Membuat service systemd sendiri dan membaca log-nya lewat `journalctl`.
- Membuka hanya port yang dibutuhkan memakai `ufw`, tanpa mengunci diri sendiri.
- Mengambil kode dari GitHub di server dengan deploy key.

---

## Pelajaran 1.1 — Terminal dan shell

### Tujuan

Membaca prompt, memakai `man`, dan mengatur environment variable tanpa panic.

### Konsep

**Terminal** adalah jendela. **Shell** adalah program di dalamnya yang membaca perintahmu — di Ubuntu defaultnya `bash`.

Prompt khas server:

```
deploy@web-01:~$
```

- `deploy` — user yang sedang login.
- `web-01` — nama host.
- `~` — direktori kerja sekarang (`~` = home user).
- `$` — kamu user biasa. Kalau `#`, kamu root. **Perhatikan tanda ini** sebelum menekan Enter.

Perintah selalu berbentuk `nama-perintah [opsi] [argumen]`. Shell memecahnya berdasarkan spasi, jadi path dengan spasi harus dikutip: `cd "folder saya"`.

`PATH` adalah daftar direktori tempat shell mencari perintah. `echo $PATH` menampilkannya. Kalau muncul `command not found` padahal paketnya terpasang, biasanya letak binary tidak ada di `PATH`.

Environment variable adalah pasangan nama=nilai yang diwariskan ke proses anak. Variabel yang di-`export` terlihat oleh program yang kamu jalankan; variabel tanpa `export` hanya ada di shell saat itu.

### Praktik

1. Kenali diri sendiri dan mesinnya.

```bash
whoami            # user aktif
hostname          # nama server
pwd               # direktori sekarang
echo $SHELL       # shell yang dipakai
cat /etc/os-release | head -3   # versi Ubuntu
```

2. Baca dokumentasi tanpa internet.

```bash
man ls            # keluar dengan q
ls --help | head -20
man -k permission # cari halaman manual berdasarkan kata kunci
```

3. Latihan riwayat dan pelengkapan.

```bash
history | tail -10
!!                # jalankan ulang perintah terakhir
sudo !!           # jalankan ulang perintah terakhir sebagai root
```

Tekan `Tab` setelah mengetik sebagian nama file atau perintah — shell melengkapinya.

4. Variabel environment.

```bash
export APP_ENV=production
echo $APP_ENV
env | grep APP_          # variabel yang diwariskan ke proses anak
bash -c 'echo $APP_ENV'  # proses anak mewarisi nilainya
unset APP_ENV
```

5. Variabel sementara untuk satu perintah saja.

```bash
APP_ENV=staging bash -c 'echo $APP_ENV'   # hanya berlaku untuk perintah itu
```

6. Simpan permanen di `~/.bashrc` (hanya untuk user ini, hanya shell interaktif).

```bash
echo 'export EDITOR=nano' >> ~/.bashrc
source ~/.bashrc
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `command not found` | Paket belum terpasang, atau binary tidak ada di `PATH` | `command -v nama` untuk cek; kalau kosong, pasang paketnya |
| Perintah jalan tapi tidak ada efeknya | Kamu di direktori atau user yang salah | `pwd` dan `whoami` dulu |
| `Permission denied` saat edit file | File milik user lain atau root | Pakai `sudo`, atau perbaiki kepemilikan dengan `chown` |

### Latihan

1. Tampilkan versi kernel, versi Ubuntu, dan waktu server dalam satu blok perintah.
2. Buat variabel `PROJECT=devops-pro`, buktikan proses anak mewarisinya, lalu hapus variabelnya.

**Kriteria selesai:** kamu bisa menjelaskan kenapa `bash -c 'echo $PROJECT'` berhasil sementara shell baru tidak melihatnya.

### Cek pemahaman

1. Apa beda `#` dan `$` di ujung prompt? — `#` berarti shell berjalan sebagai root; `$` user biasa.
2. Di mana shell mencari perintah yang kamu ketik? — Di direktori yang terdaftar pada `PATH`, berurutan.
3. Kenapa `export` penting? — Tanpa `export`, variabel hanya milik shell dan tidak diwariskan ke program anak.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tulis `perintah-harian.md`: 10 perintah yang kamu pakai, satu baris fungsi tiap perintah, dan cara menemukan bantuannya (`man` atau `--help`).

**Siapkan:** Jalankan `echo $PATH`, `echo $SHELL`, dan `history | tail -20`; simpan hasilnya.

**Bawa bukti:** Isi `perintah-harian.md` + 3 baris `history` yang kamu jelaskan sendiri.

Dibahas di awal **Pelajaran 1.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.2 — Filesystem dan navigasi

### Tujuan

Berpindah direktori dan menemukan file dengan cepat, memahami di mana konfigurasi dan log disimpan.

### Konsep

Linux punya satu pohon direktori, dimulai dari `/`. Tidak ada drive `C:`.

| Path | Isi |
| --- | --- |
| `/etc` | Konfigurasi sistem dan service |
| `/var/log` | Log |
| `/var/www` | Konvensi letak situs web |
| `/home/deploy` | Data milik user `deploy` |
| `/opt` | Aplikasi pihak ketiga |
| `/tmp` | Sementara, bisa hilang saat reboot |
| `/usr/bin` | Binary yang dipasang paket |

Path absolut mulai dari `/`. Path relatif dihitung dari direktori sekarang. `.` adalah direktori ini, `..` direktori induk.

Symlink adalah pintu ke path lain. Di Nginx nanti, `sites-enabled` berisi symlink ke `sites-available`.

### Praktik

1. Navigasi dasar.

```bash
cd /var/log && pwd
cd ..              # naik satu level
cd ~               # ke home
cd -               # kembali ke direktori sebelumnya
ls -lah            # detail, ukuran human-readable, file tersembunyi
```

2. Buat struktur proyek.

```bash
mkdir -p ~/apps/demo/{src,logs,config}
touch ~/apps/demo/src/app.py
cp ~/apps/demo/src/app.py ~/apps/demo/src/app.py.bak
mv ~/apps/demo/src/app.py.bak ~/apps/demo/src/app.bak
rm ~/apps/demo/src/app.bak
tree ~/apps          # kalau belum ada: sudo apt install tree
```

3. Symlink.

```bash
ln -s ~/apps/demo/config ~/demo-config
ls -l ~/demo-config   # panah -> menunjukkan target
readlink ~/demo-config
rm ~/demo-config      # hapus symlink, bukan targetnya
```

4. Cari file.

```bash
find /etc -name '*.conf' -type f | head
find ~/apps -type f -size +1M
find /var/log -name '*.log' -mtime -1     # diubah dalam 24 jam terakhir
```

5. Cari isi file.

```bash
grep -rn "server_name" /etc/nginx 2>/dev/null | head
sudo grep -c "ERROR" /var/log/syslog   # /var/log/syslog hanya terbaca root/adm
```

6. Cek ukuran pemakai disk — kebiasaan yang menyelamatkan server.

```bash
df -h /            # sisa ruang disk
du -sh /var/log/* 2>/dev/null | sort -h | tail -5
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `No such file or directory` | Path salah atau direktori belum dibuat | `ls` di tiap level path untuk menemukan titik putusnya |
| `Not a directory` | Bagian path ternyata file | `ls -l` komponen path |
| `du` lambat sekali | Menyisir direktori besar | Batasi ke path spesifik |

### Latihan

1. Buat `~/apps/blog/{public,data,tmp}` lalu tampilkan struktur dengan `tree`.
2. Cari semua file `.conf` di `/etc` yang mengandung kata `include`, tampilkan maksimal 5 baris.

**Kriteria selesai:** kamu bisa menyebut path absolut file log sistem dan log Nginx nanti tanpa mencari.

### Cek pemahaman

1. Apa isi `/etc` dan `/var/log`? — Konfigurasi sistem; log.
2. Apa yang terjadi pada symlink jika targetnya dihapus? — Symlink tetap ada tetapi rusak (dangling).
3. Bedanya `find` dan `grep`? — `find` mencari berdasarkan atribut file; `grep` mencari isi teks.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat struktur `~/apps/blog/{public,data,tmp}`, lalu cari 3 file `.conf` di `/etc` yang paling sering muncul dan 3 direktori terbesar di `/var/log`.

**Siapkan:** Biasakan `pwd` dan `ls -lah` sebelum setiap perintah destruktif.

**Bawa bukti:** `tree ~/apps` + daftar 3 path absolut hasil pencarian, masing-masing dengan alasan kenapa kamu memilihnya.

Dibahas di awal **Pelajaran 1.3** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.3 — File dan teks

### Tujuan

Membaca, menyaring, dan memindahkan teks dengan redirect, pipe, dan alat baris perintah.

### Konsep

Setiap proses punya tiga saluran: stdin (0), stdout (1), stderr (2). Redirect mengubah tujuannya:

| Sintaks | Arti |
| --- | --- |
| `>` | stdout ke file, menimpa |
| `>>` | stdout ke file, menambah di akhir |
| `2>` | stderr ke file |
| `2>&1` | stderr ikut ke stdout |
| `\|` | stdout proses kiri jadi stdin proses kanan |

Ini bukan detail kosmetik. Saat men-debug deploy nanti, `docker compose up -d 2>&1 | tee deploy.log` menyimpan bukti yang bisa dibaca ulang.

### Praktik

1. Baca file tanpa membuka editor.

```bash
cat /etc/hostname
less /etc/services        # navigasi panah, keluar dengan q
head -5 /etc/passwd
sudo tail -f /var/log/syslog    # ikuti baris baru, keluar dengan Ctrl+C
```

2. Redirect dan pipe.

```bash
echo "baris pertama" > ~/catatan.txt
echo "baris kedua" >> ~/catatan.txt
cat ~/catatan.txt | wc -l
ls /tidak-ada 2> ~/error.txt; cat ~/error.txt
```

3. Saring dan olah.

```bash
cut -d: -f1 /etc/passwd | sort | uniq | head
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
awk -F: '$3 >= 1000 {print $1}' /etc/passwd   # user non-sistem
```

4. Ganti teks dengan `sed`.

```bash
sed 's/localhost/127.0.0.1/' file.txt          # tampilkan saja
sed -i.bak 's/localhost/127.0.0.1/' file.txt   # ubah, simpan cadangan .bak
```

5. Arsip dan pindahkan.

```bash
tar -czf ~/backup-etc-ssh.tar.gz /etc/ssh 2>/dev/null
tar -tzf ~/backup-etc-ssh.tar.gz | head
tar -xzf ~/backup-etc-ssh.tar.gz -C /tmp
```

6. Uji HTTP dengan `curl` — alat yang akan sering dipakai di modul 2.

```bash
curl -I https://example.com          # header saja
curl -s -o /dev/null -w '%{http_code}\n' https://example.com
curl -s https://api.github.com | head -20
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| File jadi kosong setelah `>` | Menimpa file yang sedang dibaca perintah yang sama | Tulis ke file lain, lalu `mv` |
| `tar: Removing leading '/'` | Path absolut diarsipkan | Normal; gunakan `-C` saat mengekstrak |
| `curl: (7) Failed to connect` | Port tertutup, DNS salah, atau service mati | Cek `ss -tulpn`, `dig`, lalu log service |

### Latihan

1. Simpan 10 baris pertama `/etc/passwd` ke `~/passwd-head.txt`, lalu tampilkan hanya nama user yang punya home di `/home`.
2. Ambil status code `https://example.com` dan tulis hasilnya ke `~/status.txt`.

**Kriteria selesai:** kamu bisa menjelaskan kenapa `2>&1` perlu ditulis sebelum `|` pada beberapa kasus.

### Cek pemahaman

1. Beda `>` dan `>>`? — Menimpa vs menambahkan.
2. Apa fungsi pipe `|`? — Menyambung stdout proses pertama ke stdin proses kedua.
3. Kenapa `tail -f` berguna saat deploy? — Melihat log bertambah real-time tanpa membuka ulang file.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Ambil 50 baris terakhir salah satu log, saring hanya baris yang mengandung `error` (tanpa peduli huruf besar/kecil), lalu simpan ke `~/latihan-log.txt`.

**Siapkan:** Uji `curl -I` ke domainmu sendiri dan simpan output header-nya.

**Bawa bukti:** Isi `~/latihan-log.txt` + penjelasan singkat kapan memakai `>` dan kapan `>>`.

Dibahas di awal **Pelajaran 1.4** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.4 — User, grup, permission, dan SSH

### Tujuan

Membuat user deploy, mengatur izin file, dan mengunci akses SSH ke key saja.

### Konsep

Setiap file punya owner, group, dan tiga set izin: owner, group, other. `rwx` untuk file berarti baca, tulis, eksekusi. Untuk direktori, `x` berarti "boleh masuk", `r` berarti "boleh daftar isi", `w` berarti "boleh membuat/menghapus entri".

Angka oktal: `r=4`, `w=2`, `x=1`. `750` = owner `rwx`, group `r-x`, other `---`.

`umask` menentukan izin default file baru. `umask 022` menghasilkan file `644` dan direktori `755`.

Prinsip yang dipakai sepanjang kursus: **aplikasi tidak berjalan sebagai root**, dan **SSH tidak menerima password**.

### Praktik

1. Buat user deploy.

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy     # hanya jika perlu sudo penuh
id deploy
```

Untuk server produksi, alternatif lebih ketat: user tanpa sudo, hanya untuk menjalankan aplikasi.

2. Permission.

```bash
mkdir -p ~/apps/demo && cd ~/apps/demo
echo "PORT=3000" > .env
chmod 600 .env            # hanya owner bisa baca/tulis
ls -l .env
mkdir -p public data          # buat dulu; chmod pada direktori yang belum ada akan gagal
chmod 755 public
chmod -R 750 data
```

Perintah berbahaya: `chmod -R 777` membuka semua file untuk semua user. Jangan dipakai sebagai solusi "permission denied" — perbaiki kepemilikan:

```bash
sudo chown -R deploy:deploy ~/apps/demo
```

3. Buat SSH key di laptop (bukan di server).

```bash
ssh-keygen -t ed25519 -C "deploy@laptop"
cat ~/.ssh/id_ed25519.pub
```

4. Pasang public key di server.

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys    # tempel isi id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
```

5. Uji login tanpa password dari terminal baru **sebelum** mengubah konfigurasi SSH.

```bash
ssh deploy@203.0.113.10
```

6. Hardening SSH — hanya setelah langkah 5 berhasil.

Jangan mengedit `/etc/ssh/sshd_config` langsung. Di Ubuntu, baris `Include /etc/ssh/sshd_config.d/*.conf` ada di **atas** file dan sshd memakai nilai **pertama** yang ditemukan. Artinya drop-in selalu menang — termasuk `50-cloud-init.conf` bawaan image cloud yang sering berisi `PasswordAuthentication yes`. Karena itu buat drop-in bernomor paling akhir:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'CONF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
CONF

sudo sshd -t                 # validasi konfigurasi
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication)'
sudo systemctl restart ssh           # Ubuntu 24.04 socket activation: sudo systemctl restart ssh.socket
```

`sshd -T` menampilkan nilai **efektif** setelah semua include digabung — inilah bukti hardening benar-benar berlaku, bukan sekadar tertulis di file yang kalah prioritas.

Peringatan: kalau key belum terbukti bekerja, menonaktifkan password akan mengunci kamu dari server. Siapkan akses console/VNC dari panel VPS sebagai jalur darurat.

7. `sudo` yang selektif.

```bash
sudo -l                      # apa yang boleh kamu jalankan
sudo visudo                  # SELALU edit sudoers lewat visudo
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Permission denied (publickey)` | Key belum terpasang, izin `~/.ssh` salah | `chmod 700 ~/.ssh`, `chmod 600 authorized_keys`, cek `journalctl -u ssh` |
| `sudo: command not found` | User tidak di grup sudo | `usermod -aG sudo user` lalu login ulang |
| Aplikasi tidak bisa tulis file | Owner direktori salah | `chown -R user:group path`, bukan `chmod 777` |

### Latihan

1. Buat user `deploy`, pasang key, login tanpa password, lalu matikan `PasswordAuthentication` dan buktikan login masih bisa.
2. Buat file `.env` berisi satu baris, atur agar hanya owner yang bisa membacanya, dan buktikan dengan `ls -l`.

**Kriteria selesai:** `ssh deploy@IP` masuk tanpa diminta password, dan `sudo sshd -t` tidak menghasilkan error.

### Cek pemahaman

1. Arti `640`? — Owner baca/tulis, group baca, other tidak punya akses.
2. Kenapa `chmod 777` buruk? — Semua user bisa menulis dan mengeksekusi; memperbesar permukaan serangan dan menyembunyikan masalah kepemilikan.
3. Kenapa `sshd -t` dijalankan sebelum reload? — Menolak restart dengan konfigurasi rusak yang bisa mengunci akses.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat user `deploy`, pasang SSH key, lalu kunci akses dengan drop-in `99-hardening.conf`.

**Siapkan:** Pastikan kamu masih bisa login dari terminal baru **sebelum** menutup sesi lama.

**Bawa bukti:** Output `sudo sshd -T | grep -Ei 'passwordauth|permitrootlogin'` + bukti login tanpa password.

Dibahas di awal **Pelajaran 1.5** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.5 — Proses, service, dan systemd

### Tujuan

Melihat apa yang berjalan, menghentikannya dengan benar, dan membuat service sendiri.

### Konsep

Proses adalah program yang sedang berjalan, punya PID. Proses anak dibuat dari proses induk. `systemd` (PID 1) mengelola service: menyalakan saat boot, me-restart kalau mati, mengumpulkan log ke journal.

`systemctl` mengendalikan service. `journalctl` membaca log-nya. Ini pengganti `nohup app.py &` yang rapuh.

### Praktik

1. Lihat proses dan port.

```bash
ps aux | head
ps aux --sort=-%mem | head -5
top -b -n1 | head -12
ss -tulpn | grep LISTEN
```

2. Sinyal.

```bash
sleep 300 &
jobs
kill %1            # SIGTERM: minta berhenti baik-baik
kill -9 PID        # SIGKILL: paksa, hanya kalau SIGTERM gagal
```

3. Kelola service bawaan.

```bash
systemctl status ssh --no-pager
systemctl is-enabled ssh
sudo systemctl restart ssh
journalctl -u ssh -n 20 --no-pager
journalctl -u ssh --since "10 min ago"
```

4. Buat service sendiri. Siapkan skrip:

```bash
mkdir -p ~/apps/hello
cat > ~/apps/hello/server.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(b"hello from systemd\n")

HTTPServer(("127.0.0.1", 9000), H).serve_forever()
PY
```

```bash
sudo nano /etc/systemd/system/hello.service
```

```ini
[Unit]
Description=Hello HTTP server
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/apps/hello
ExecStart=/usr/bin/python3 /home/deploy/apps/hello/server.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hello
systemctl status hello --no-pager
curl -s http://127.0.0.1:9000
journalctl -u hello -n 20 --no-pager
```

5. Uji ketahanan.

```bash
sudo systemctl stop hello
sudo systemctl start hello
sudo reboot       # setelah server hidup lagi: systemctl is-active hello
```

6. Timer sederhana (pengganti cron).

```bash
sudo systemctl list-timers --all | head
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `status=203/EXEC` | Path `ExecStart` salah atau tidak executable | Cek path absolut, `chmod +x` bila skrip |
| Service langsung `inactive (dead)` | `Type` tidak cocok atau program keluar | Jalankan `ExecStart` manual sebagai user yang sama |
| Perubahan unit tidak berlaku | Lupa `daemon-reload` | `sudo systemctl daemon-reload` lalu restart |

### Latihan

1. Jalankan service `hello`, matikan prosesnya paksa dengan `kill -9`, dan buktikan systemd menyalakannya kembali.
2. Ubah port ke `9001`, reload, dan verifikasi lewat `curl`.

**Kriteria selesai:** `systemctl is-enabled hello` menghasilkan `enabled` dan service hidup setelah reboot.

### Cek pemahaman

1. Beda `enable` dan `start`? — `enable` mengaktifkan saat boot; `start` menjalankan sekarang.
2. Kenapa `Restart=on-failure` penting untuk aplikasi? — Memulihkan otomatis dari crash tanpa campur tangan manual.
3. Di mana log service systemd? — Journal, dibaca dengan `journalctl -u nama`.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat service `hello` dengan unit file sendiri yang melayani `127.0.0.1:9000`, lalu paksa mati dengan `kill -9` dan biarkan systemd menyalakannya kembali.

**Siapkan:** Baca `journalctl -u hello` dan temukan baris yang menandai crash.

**Bawa bukti:** `systemctl status hello` + potongan log crash + penjelasan arti `Restart=on-failure`.

Dibahas di awal **Pelajaran 1.6** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.6 — Manajemen paket dan update keamanan

### Tujuan

Memasang perangkat lunak dari repository resmi dan menjaga patch keamanan tetap masuk.

### Konsep

`apt` mengambil paket dari repository yang terdaftar di `/etc/apt/sources.list.d/`. `apt update` menyegarkan daftar paket; `apt upgrade` memasang versi baru.

Server yang tidak pernah di-update adalah celah keamanan yang menunggu waktu. Tapi upgrade juga bisa mengubah perilaku service — lakukan di jendela yang terkontrol dan baca outputnya.

### Praktik

1. Dasar.

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade -y
sudo apt install -y nginx curl git ufw
apt show nginx | head -15
dpkg -l | grep nginx
```

2. Hapus dan bersihkan.

```bash
sudo apt remove --purge paket-name
sudo apt autoremove -y
sudo apt clean
```

3. Update keamanan otomatis.

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades   # pilih Yes
systemctl status unattended-upgrades --no-pager
cat /etc/apt/apt.conf.d/20auto-upgrades
```

4. Cek kapan update terakhir dan apakah reboot tertunda.

```bash
ls -l /var/lib/apt/periodic/
cat /var/run/reboot-required 2>/dev/null || echo "tidak perlu reboot"
```

5. Repository pihak ketiga (pola yang dipakai Docker di modul 3): key di `/etc/apt/keyrings/`, sumber di `/etc/apt/sources.list.d/`. Jangan menambah repo acak.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Could not get lock` | Proses apt lain berjalan | Tunggu, cek `ps aux \| grep apt` |
| `404 Not Found` saat update | Repo usang/tidak cocok versi | Periksa file di `/etc/apt/sources.list.d/` |
| Upgrade mengubah konfigurasi | Prompt konfigurasi paket | Baca pilihan dengan teliti; simpan versi lama bila ragu |

### Latihan

1. Pasang `htop` dan `tree`, lalu buktikan keduanya ada di `PATH`.
2. Aktifkan `unattended-upgrades` dan tunjukkan file konfigurasi yang mengaktifkannya.

**Kriteria selesai:** `apt list --upgradable` bisa kamu baca dan jelaskan artinya.

### Cek pemahaman

1. Beda `apt update` dan `apt upgrade`? — Menyegarkan indeks vs memasang versi baru.
2. Kenapa `autoremove` perlu hati-hati? — Bisa menghapus dependensi yang masih dibutuhkan paket lain.
3. Sinyal apa yang menunjukkan server butuh reboot setelah patch? — Adanya `/var/run/reboot-required`.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Aktifkan `unattended-upgrades`, lalu catat 3 paket yang punya pembaruan beserta alasan kenapa itu penting.

**Siapkan:** Periksa `/var/run/reboot-required` dan catat apakah servermu butuh reboot.

**Bawa bukti:** `apt list --upgradable` + isi `/etc/apt/apt.conf.d/20auto-upgrades`.

Dibahas di awal **Pelajaran 1.7** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.7 — Jaringan dan firewall

### Tujuan

Membaca alamat IP, port yang terbuka, dan mengaktifkan firewall tanpa mengunci SSH.

### Konsep

Server punya interface jaringan dengan alamat IP. Aplikasi "mendengarkan" pada port. Port `22` SSH, `80` HTTP, `443` HTTPS. Aplikasi yang listen di `127.0.0.1` hanya bisa diakses dari server itu sendiri — ini pola aman untuk backend di belakang Nginx.

DNS menerjemahkan nama ke IP lewat record `A` (IPv4) dan `AAAA` (IPv6). `ufw` adalah frontend iptables: default tolak masuk, izinkan yang eksplisit.

### Praktik

1. Lihat IP dan route.

```bash
ip -brief address
ip route
curl -s https://ifconfig.me; echo    # IP publik dari sudut pandang internet
```

2. Port yang terbuka.

```bash
ss -tulpn
sudo ss -tulpn | grep -E ':22|:80|:443'
```

3. DNS.

```bash
dig example.com +short
dig A app.example.com +short
cat /etc/resolv.conf
getent hosts example.com
```

4. Uji konektivitas.

```bash
ping -c 3 1.1.1.1
curl -sS -o /dev/null -w '%{http_code}\n' http://example.com
```

5. Firewall dengan urutan aman.

```bash
sudo ufw status verbose
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH          # WAJIB sebelum enable
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable                 # konfirmasi: ketik y
sudo ufw status numbered
```

Peringatan: `ufw enable` tanpa aturan `allow OpenSSH` akan memutus koneksi SSH-mu seketika. Kalau terjadi, pakai console dari panel VPS.

Peringatan kedua, khusus Docker: port container yang di-*publish* (`-p 3000:3000`) **tidak** tunduk pada `ufw`. Docker menyisipkan aturan sendiri di tabel `nat` yang diproses sebelum aturan firewall host, sehingga port itu tetap terbuka dari internet walau `ufw` menolaknya. Karena itu backend selalu di-publish ke localhost (`-p 127.0.0.1:3000:3000`). Cara memeriksa dari luar: `nc -zv <IP> 3000` dari laptop — harus gagal.

6. Cek dari luar.

```bash
# dari laptop
nc -zv 203.0.113.10 22
nc -zv 203.0.113.10 80
curl -I http://203.0.113.10
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| SSH terputus setelah `ufw enable` | Port 22 tidak diizinkan | Akses console VPS, `ufw allow OpenSSH` |
| Site jalan di localhost tapi tidak dari luar | Aplikasi bind ke `127.0.0.1` atau port tertutup | Bind ke `0.0.0.0` (bila memang publik) atau pasang reverse proxy |
| `dig` kosong | Record belum ada atau propagasi | Cek di panel DNS, tunggu TTL |

### Latihan

1. Aktifkan `ufw` dengan hanya 22, 80, 443 terbuka. Buktikan port 3000 tertutup dari luar.
2. Buat `app.example.com` mengarah ke IP server, verifikasi dengan `dig`, lalu jelaskan kenapa belum bisa diakses lewat browser.

**Kriteria selesai:** `sudo ufw status numbered` menunjukkan tiga aturan itu dan SSH masih hidup.

### Cek pemahaman

1. Apa arti listen di `127.0.0.1:3000`? — Hanya dapat diakses dari dalam server, tidak dari jaringan luar.
2. Kenapa `allow OpenSSH` sebelum `enable`? — Mencegah terkunci di luar server.
3. Beda record A dan AAAA? — A untuk IPv4, AAAA untuk IPv6.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Aktifkan `ufw` hanya dengan 22/80/443, lalu buktikan dari laptop bahwa port 3000 tertutup.

**Siapkan:** Catat hasil `dig` untuk domain yang akan dipakai di modul 2.

**Bawa bukti:** `sudo ufw status numbered` + output `nc -zv <IP> 3000` dari laptop yang harus gagal.

Dibahas di awal **Pelajaran 1.8** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 1.8 — Git di server

### Tujuan

Mengambil kode dari GitHub di server dengan aman, dan tahu kapan tidak boleh mengedit langsung di server.

### Konsep

Server produksi menerima kode dalam bentuk artefak: hasil build, image container, atau clone yang di-`pull`. Yang **tidak** dilakukan: mengedit file langsung di server lalu lupa menyalinnya ke repo.

Deploy key adalah SSH key khusus satu repo, read-only. Lebih aman daripada memakai key pribadi di server.

### Praktik

1. Identitas git dan clone.

```bash
git config --global user.name "Deploy Bot"
git config --global user.email "deploy@example.com"
git clone git@github.com:USER/REPO.git ~/apps/app
cd ~/apps/app && git log --oneline -3
```

2. Deploy key read-only.

```bash
ssh-keygen -t ed25519 -C "deploy-key-app" -f ~/.ssh/id_ed25519_github -N ""
cat ~/.ssh/id_ed25519_github.pub
```

Tempel public key di repo GitHub → Settings → Deploy keys → Add deploy key → **jangan centang** "Allow write access".

```bash
cat >> ~/.ssh/config <<'CFG'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
CFG
chmod 600 ~/.ssh/config
ssh -T git@github.com    # harus: "successfully authenticated"
```

3. Alur yang benar di server.

```bash
cd ~/apps/app
git fetch --all
git status
git pull --ff-only        # gagal kalau ada perubahan lokal -> itu fitur, bukan bug
```

4. Lihat apa yang berubah sebelum menerapkan.

```bash
git log --oneline HEAD..origin/main
git diff --stat HEAD origin/main
```

5. `.gitignore` minimal.

```gitignore
.env
node_modules/
__pycache__/
*.log
```

6. Aturan praktis: kalau butuh memperbaiki sesuatu di server, perbaiki di repo, commit, lalu tarik ulang. Kecuali darurat — dan setelah darurat, segera selaraskan repo.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Permission denied (publickey)` | Deploy key belum terpasang atau `ssh-agent` memakai key lain | `ssh -vT git@github.com` untuk melihat key mana yang dipakai |
| `diverged` saat pull | Ada commit lokal di server | `git status`, pindahkan perubahan ke branch, reset ke `origin/main` |
| `dubious ownership` | Repo dimiliki user lain | `git config --global --add safe.directory /path/repo` |

### Latihan

1. Clone satu repo publik ke `~/apps/`, tampilkan 3 commit terakhir, lalu buat satu perubahan lokal dan buktikan `git pull --ff-only` menolaknya.
2. Pasang deploy key pada satu repo dan verifikasi dengan `ssh -T git@github.com`.

**Kriteria selesai:** kamu bisa menjelaskan kenapa deploy key read-only lebih aman daripada key akun pribadi.

### Cek pemahaman

1. Kenapa tidak mengedit langsung di server? — Perubahan hilang saat redeploy dan tidak tercatat di repo.
2. Fungsi `--ff-only`? — Menolak merge otomatis, memaksa kamu melihat konflik.
3. Apa izin minimum deploy key? — Read-only ke satu repo.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Pasang deploy key read-only pada satu repo dan clone ke `~/apps/`.

**Siapkan:** Periksa isi repo: apakah sudah ada `Dockerfile`, `compose.yaml`, dan endpoint `/health`? Catat yang belum ada.

**Bawa bukti:** `ssh -T git@github.com` + `git log --oneline -3` + daftar berkas yang belum ada di repo.

Dibahas di awal **Pelajaran 2.1** — 10 menit pertama, sebelum materi baru.

---

## Proyek mini modul

Siapkan satu VPS dari nol, tanpa melihat catatan:

1. Update paket, pasang `nginx ufw git curl`.
2. Buat user `deploy`, pasang SSH key, matikan login password dan login root. Buktikan masih bisa masuk.
3. Aktifkan `ufw` dengan 22/80/443.
4. Buat service systemd `hello` yang melayani `127.0.0.1:9000`, `enable` dan uji setelah reboot.
5. Clone satu repo dengan deploy key read-only.
6. Tulis `catatan-server.md` berisi: IP, user, port terbuka, service aktif, dan perintah untuk memeriksa log.

**Kriteria selesai:** semua langkah bisa diulang dari VPS kosong dalam 30 menit, dan `catatan-server.md` cukup untuk membuat orang lain melanjutkan pekerjaanmu.

## Perintah penting

| Kebutuhan | Perintah |
| --- | --- |
| Cek user & path | `whoami`, `pwd`, `id` |
| Lihat port terbuka | `sudo ss -tulpn` |
| Status service | `systemctl status nama --no-pager` |
| Log service | `journalctl -u nama -n 50 --no-pager` |
| Ikuti log | `journalctl -u nama -f` |
| Izin file | `chmod 640 file`, `chown user:group path` |
| Firewall | `sudo ufw status numbered`, `sudo ufw allow 80/tcp` |
| Cari file | `find /path -name '*.conf'` |
| Cari isi | `grep -rn "kata" /path` |
| Uji HTTP | `curl -I https://domain`, `curl -s -o /dev/null -w '%{http_code}'` |
| Sisa disk | `df -h`, `du -sh /var/log/* \| sort -h \| tail` |

## Referensi

- Ubuntu Server documentation — https://documentation.ubuntu.com/server/
- `man` pages: `man 7 file-hierarchy`, `man 5 systemd.service`, `man 8 ufw`
- OpenSSH manual — https://www.openssh.com/manual.html
- systemd — https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
- Git documentation — https://git-scm.com/doc
