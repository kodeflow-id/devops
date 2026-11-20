# Modul 4 — GitHub Actions (CI/CD)

## Ringkasan modul

Sampai modul 3, deploy masih manual: SSH ke server, pull, build. Modul ini memindahkan seluruh proses itu ke runner GitHub: setiap push ke `main` menjalankan test, membangun image, mendorong ke registry, lalu men-deploy ke VPS.

Asumsi: repo GitHub berisi aplikasi dari modul 3 (Dockerfile + `compose.yaml`), server Ubuntu 24.04 dengan Docker dan Nginx, domain `app.example.com`, user `deploy`, dan registry GHCR (`ghcr.io`).

## Hasil pembelajaran

- Menjelaskan perbedaan CI, continuous delivery, dan continuous deployment, serta komponen workflow Actions.
- Menulis workflow YAML yang valid, membaca log-nya, dan memperbaiki kegagalan umum.
- Menjalankan test/lint otomatis dengan cache dan matrix build di pull request.
- Memakai secret dan `GITHUB_TOKEN` dengan izin minimum tanpa membocorkan nilainya.
- Membangun image dan mendorongnya ke GHCR dengan tag yang bisa di-rollback.
- Men-deploy ke VPS lewat SSH dengan health check dan mekanisme rollback.
- Mendiagnosis deploy gagal: host key, permission packages, secret kosong, tag tidak ditemukan.

---

## Pelajaran 4.1 — Konsep CI/CD dan komponen Actions

### Tujuan

Memahami alur kerja Actions sebelum menulis YAML.

### Konsep

| Istilah | Arti |
| --- | --- |
| CI (continuous integration) | Setiap perubahan otomatis diuji dan dibangun |
| Continuous delivery | Setiap build siap dirilis, rilis dipicu manual |
| Continuous deployment | Setiap perubahan yang lulus langsung dirilis |

Komponen:

- **Workflow** — satu file YAML di `.github/workflows/`.
- **Event** (`on`) — pemicu: `push`, `pull_request`, `schedule`, `workflow_dispatch`.
- **Job** — kumpulan step yang berjalan di satu runner; job default paralel.
- **Step** — satu langkah: `uses` (memakai action) atau `run` (perintah shell).
- **Runner** — mesin yang menjalankan job (`ubuntu-24.04`, `ubuntu-latest`, self-hosted).
- **Action** — komponen siap pakai dari Marketplace, dipanggil dengan `uses: owner/repo@versi`.

Runner GitHub bersifat sementara: setiap job mulai dari VM bersih. Karena itu jangan mengandalkan file yang tersisa dari job sebelumnya.

### Praktik

1. Lihat paket gratis dan menit yang terpakai: repo → Settings → Billing (atau halaman Actions usage). Repo publik tidak memakai kuota.

2. Buat workflow pertama yang bisa dipicu manual.

```bash
mkdir -p .github/workflows
cat > .github/workflows/hello.yml <<'YAML'
name: Hello

on:
  workflow_dispatch:

jobs:
  info:
    runs-on: ubuntu-24.04
    steps:
      - name: Tampilkan informasi runner
        run: |
          uname -a
          docker --version
          echo "repo=$GITHUB_REPOSITORY ref=$GITHUB_REF"
YAML
git add .github/workflows/hello.yml && git commit -m "ci: workflow hello" && git push
```

3. Jalankan dari tab **Actions** → pilih workflow → **Run workflow**. Baca log tiap step.

4. Kenali variabel bawaan yang sering dipakai: `GITHUB_SHA`, `GITHUB_REF_NAME`, `GITHUB_REPOSITORY`, `GITHUB_ACTOR`, `RUNNER_OS`.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Workflow tidak muncul di tab Actions | File bukan di `.github/workflows/`, ekstensi bukan `.yml`/`.yaml`, atau belum di-push ke default branch | Periksa path dan branch |
| `workflow_dispatch` tidak bisa diklik | Workflow belum ada di default branch | Push ke `main` dulu |
| `unable to resolve action owner/repo@vN` | Versi major action berubah / tag dihapus | Cek versi terbaru di GitHub Marketplace, ganti `@vN` sesuai |

### Latihan

1. Buat workflow `workflow_dispatch` yang mencetak `GITHUB_SHA` dan versi Docker runner.
2. Sebutkan tiga perbedaan runner GitHub dengan server produksimu.

**Kriteria selesai:** kamu bisa menjelaskan mengapa job tidak boleh bergantung pada file dari job lain tanpa artifact.

### Cek pemahaman

1. Beda CI dan continuous deployment? — CI menguji tiap perubahan; continuous deployment otomatis merilis yang lulus.
2. Kenapa runner dianggap tidak menyimpan state? — VM-nya baru setiap job.
3. Apa fungsi `uses`? — Memanggil action siap pakai, bukan perintah shell.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat workflow `workflow_dispatch` yang mencetak `GITHUB_SHA`, `GITHUB_REF_NAME`, dan versi Docker di runner.

**Siapkan:** Baca log run tersebut sampai selesai dan catat nama tiap step.

**Bawa bukti:** Tangkapan layar run hijau + daftar step + jawaban: kenapa job tidak boleh bergantung pada file dari job lain?

Dibahas di awal **Pelajaran 4.2** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.2 — Workflow pertama

### Tujuan

Menulis YAML yang benar dan tahu cara membacanya saat gagal.

### Konsep

Struktur minimum:

```yaml
name: Nama
on: [push]
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - run: echo "halo"
```

YAML sensitif terhadap indentasi spasi (bukan Tab). Kesalahan paling sering: indentasi salah, titik dua hilang, atau `on:` ditulis `On:`.

`actions/checkout` wajib dipanggil lebih dulu kalau step butuh isi repo. Tanpa itu, direktori kerja kosong.

### Praktik

1. Workflow CI dasar.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Pasang Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm

      - name: Pasang dependensi
        run: npm ci

      - name: Jalankan lint
        run: npm run lint --if-present

      - name: Jalankan test
        run: npm test --if-present
```

2. Validasi YAML sebelum push (tanpa Actions sekalipun).

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/ci.yml')); print('YAML valid')" 2>/dev/null || \
  node -e "const fs=require('fs');console.log('cek manual: indentasi konsisten')"
```

3. Melihat log: tab **Actions** → pilih run → klik job → klik step. Baris merah menandai perintah yang gagal; exit code tercatat.

4. Debug lokal sebelum push.

```bash
npm ci
npm test
docker build -t node-api:local .
```

Memperbaiki di lokal jauh lebih cepat daripada bolak-balik lewat runner.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Invalid workflow file` | YAML tidak valid | Cek indentasi; gunakan editor dengan YAML linter |
| `npm ci` gagal | Tidak ada `package-lock.json` atau tidak sinkron | Jalankan `npm install` lokal, commit lockfile |
| Step hijau tapi tidak melakukan apa pun | `--if-present` atau perintah tak menghasilkan error | Pastikan script benar-benar ada di `package.json` |
| Job gagal di langkah checkout | Repo private tanpa izin, atau submodule | Tambahkan `token`/`submodules` pada `checkout` |

### Latihan

1. Buat workflow yang berjalan pada `push` dan `pull_request`, menjalankan `npm ci` dan `npm test`.
2. Sengaja buat satu test gagal, lihat log-nya, lalu perbaiki.

**Kriteria selesai:** kamu bisa menunjukkan step mana yang gagal dan kenapa dari log Actions tanpa menebak.

### Cek pemahaman

1. Kenapa `checkout` hampir selalu jadi step pertama? — Agar isi repo tersedia di runner.
2. Kenapa Tab tidak boleh dipakai di YAML? — Spesifikasi YAML melarang Tab untuk indentasi.
3. Apa yang harus dijalankan lokal sebelum push? — Perintah yang sama dengan step CI (install, lint, test, build).

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat workflow CI yang berjalan pada `push` ke `main` dan semua `pull_request`: checkout, setup, install, test.

**Siapkan:** Sengaja buat satu test gagal di branch terpisah, lalu perbaiki.

**Bawa bukti:** Dua tautan run: satu merah (gagal) dan satu hijau, plus step mana yang gagal pada run merah.

Dibahas di awal **Pelajaran 4.3** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.3 — Test dan lint otomatis

### Tujuan

Menjalankan pengujian di beberapa versi sekaligus dan menahan perubahan yang gagal.

### Konsep

**Matrix** menjalankan job yang sama dengan kombinasi variabel berbeda. `fail-fast: false` membuat kombinasi lain tetap jalan walau satu gagal — berguna untuk melihat gambaran lengkap.

Branch protection memakai hasil job sebagai syarat merge: pull request tidak bisa digabung sebelum status check hijau.

### Praktik

1. Matrix build.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      matrix:
        node: ['20', '22']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test --if-present
```

2. Job terpisah untuk lint dan test (paralel, lebih cepat terlihat).

```yaml
jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npx --yes eslint . || true   # jangan `|| true` di produksi sungguhan

  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npm test
```

Catatan: `|| true` membuat langkah selalu hijau. Hanya dipakai saat memperkenalkan lint ke kode lama, dan harus dihapus setelah bersih.

3. Jadikan syarat merge: repo → Settings → Branches → Add branch protection rule untuk `main` → aktifkan **Require status checks to pass** → pilih nama job (`test`).

4. Untuk Python (sesuaikan bila proyekmu Python).

```yaml
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: pip install -r requirements.txt
      - run: pytest -q
```

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Build matrix memakan waktu lama | Cache tidak aktif | Aktifkan `cache:` pada setup action |
| `npm ci` gagal hanya di satu versi Node | Ketidakcocokan dependensi | Pin `engines` di `package.json` atau perbaiki kode |
| Status check tidak muncul di PR | Nama job berubah, atau workflow belum berjalan di branch itu | Samakan nama job dengan yang didaftarkan di branch protection |

### Latihan

1. Tambahkan matrix dua versi Node dan buktikan keduanya berjalan.
2. Aktifkan branch protection pada `main` dan buktikan PR tidak bisa di-merge saat test merah.

**Kriteria selesai:** pull request menampilkan status check dari workflow-mu.

### Cek pemahaman

1. Apa fungsi `cache: npm`? — Menyimpan cache dependensi antar-run agar `npm ci` lebih cepat.
2. Kenapa `fail-fast: false` berguna? — Melihat semua kombinasi yang gagal, bukan hanya yang pertama.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Tambahkan matrix dua versi bahasa, aktifkan cache dependensi, dan pasang branch protection yang mewajibkan status check hijau.

**Siapkan:** Buat pull request yang sengaja gagal dan buktikan tombol merge terkunci.

**Bawa bukti:** Tangkapan layar status check di PR + perbandingan waktu run dengan dan tanpa cache.

Dibahas di awal **Pelajaran 4.4** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.4 — Secret, variable, dan environment

### Tujuan

Menyimpan kredensial dengan benar dan memberi token izin seminimal mungkin.

### Konsep

| Konsep | Sifat |
| --- | --- |
| `secrets.*` | Terenkripsi, tidak bisa dibaca setelah disimpan, otomatis di-mask di log |
| `vars.*` | Tidak rahasia, terlihat sebagai teks biasa |
| `GITHUB_TOKEN` | Token otomatis per-run, izinnya diatur blok `permissions` |
| Environment | Kumpulan secret/variable dengan aturan proteksi (reviewer, branch terbatas) |

`GITHUB_TOKEN` adalah cara paling aman mengakses registry dan API GitHub: tidak perlu Personal Access Token, dan masa berlakunya hanya selama run.

Jangan pernah mencetak secret ke log. Masking bekerja untuk nilai yang persis, tetapi nilai yang di-encode atau dipotong bisa lolos — jadi jangan dicetak sama sekali.

### Praktik

1. Buat secret di repo: Settings → Secrets and variables → Actions → New repository secret. Contoh nama: `SSH_HOST`, `SSH_USER`, `SSH_KEY`, `DEPLOY_PATH`. Nilai rahasia hanya diisi di GitHub, tidak di repo.

2. Pakai di workflow.

```yaml
    steps:
      - name: Tampilkan konfigurasi non-rahasia
        run: |
          echo "host=${{ vars.DEPLOY_HOST }}"
          echo "user=${{ secrets.SSH_USER }}"        # hindari contoh seperti ini di kode sungguhan
```

3. Environment dengan reviewer (untuk produksi).

```yaml
jobs:
  deploy:
    runs-on: ubuntu-24.04
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: echo "deploy ke production"
```

Buat environment `production` di Settings → Environments, tambahkan **Required reviewers**, dan batasi deployment branch ke `main`.

4. `permissions` minimum per workflow.

```yaml
permissions:
  contents: read
```

Naikkan hanya di job yang butuh, misalnya untuk mendorong image:

```yaml
jobs:
  publish:
    permissions:
      contents: read
      packages: write
```

5. Cek kebocoran: cari di log Actions kata kunci seperti `password`, `token`, `PRIVATE KEY`. Kalau ada yang tercetak, rotasi nilainya segera.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Secret terlihat kosong (`echo ${{ secrets.X }}` menghasilkan baris kosong) | Nama salah atau secret belum dibuat di repo/environment yang tepat | Periksa ejaan nama dan scope secret |
| Secret bocor di log | Dicetak lewat `echo` atau dipakai di perintah yang menampilkannya | Hapus step itu, rotasi secret |
| Fork PR tidak bisa mengakses secret | Desain keamanan GitHub | Gunakan `pull_request_target` dengan sangat hati-hati, atau jalankan deploy hanya dari branch utama |

### Latihan

1. Buat satu secret dan satu variable, tampilkan hanya variable-nya di log.
2. Tambahkan `permissions: contents: read` pada workflow dan jelaskan mengapa itu aman.

**Kriteria selesai:** tidak ada nilai rahasia yang tercetak di log manapun.

### Cek pemahaman

1. Kenapa `GITHUB_TOKEN` lebih baik daripada PAT? — Izin terbatas per-run, otomatis kedaluwarsa, tidak perlu disimpan.
2. Apa gunanya required reviewers? — Menahan deploy sampai ada persetujuan manusia.
3. Kenapa secret di PR dari fork tidak tersedia? — Mencegah pencurian kredensial lewat pull request.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat satu secret, satu variable, dan satu environment `production` dengan required reviewer.

**Siapkan:** Audit log run: pastikan tidak ada nilai rahasia yang tercetak.

**Bawa bukti:** Daftar **nama** secret/variable (tanpa nilainya) + bukti deploy menunggu persetujuan + hasil audit log.

Dibahas di awal **Pelajaran 4.5** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.5 — Build dan push image ke GHCR

### Tujuan

Menghasilkan image ber-tag yang bisa dilacak dan di-rollback.

### Konsep

GHCR (`ghcr.io`) menerima login memakai `GITHUB_TOKEN` sebagai username `${{ github.actor }}`. Butuh `packages: write`.

Strategi tag yang sehat:

| Tag | Gunanya |
| --- | --- |
| `sha-<commit>` | Presisi untuk rollback |
| `latest` | Kenyamanan; **tidak** dipakai sebagai satu-satunya acuan deploy |
| `<git tag>` | Untuk rilis resmi |

Cache build memakai `type=gha` sehingga build berikutnya jauh lebih cepat.

### Praktik

1. Workflow build & push.

```yaml
name: Build and push

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Login ke GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Siapkan metadata tag
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=sha,prefix=sha-,format=long   # 40 hex, sama dengan workflow_run.head_sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build dan push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

2. Verifikasi hasil: repo → tab **Packages** → pilih image → lihat tag dan ukuran.

3. Uji tarik image di server.

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u USERNAME --password-stdin
# tag sha panjang (40 karakter) bila metadata memakai format=long
docker pull "ghcr.io/USER/REPO:sha-$(git rev-parse HEAD)"
```

Catatan: bila package diset private, server butuh kredensial untuk menariknya. Untuk repo publik, image juga bisa dibuat publik.

4. Rollback dengan tag sha: identifikasi tag sebelumnya di tab Packages.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `denied: installation not allowed to Write organization package` | `packages: write` tidak ada | Tambahkan blok `permissions` di job |
| `unauthorized` saat login | Memakai password akun, bukan token | Pakai `${{ secrets.GITHUB_TOKEN }}` |
| Build selalu dari nol | Cache tidak dikonfigurasi | Tambahkan `cache-from`/`cache-to: type=gha` |
| Tag image memakai garis bawah (`_`) | Nama repo dengan huruf besar | Tag GHCR memakai huruf kecil; metadata action menanganinya |

### Latihan

1. Push ke `main`, buktikan image muncul di Packages dengan tag `sha-...`.
2. Bandingkan waktu build pertama dan kedua; tunjukkan efek cache.

**Kriteria selesai:** `docker pull ghcr.io/USER/REPO:sha-<commit>` berhasil di server.

### Cek pemahaman

1. Kenapa tag sha lebih penting daripada `latest`? — Menunjuk image persis, memungkinkan rollback deterministik.
2. Kenapa `permissions` diletakkan di level job, bukan workflow? — Least privilege: hanya job yang butuh yang mendapat izin tulis.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Build dan push image ke GHCR dengan tag `sha` panjang + `latest`, memakai cache `type=gha`.

**Siapkan:** Tarik image itu dari server memakai kredensial yang benar.

**Bawa bukti:** Tautan tab Packages + `docker pull` sukses di server + waktu build pertama vs kedua.

Dibahas di awal **Pelajaran 4.6** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.6 — Deploy ke VPS lewat SSH

### Tujuan

Menerapkan image baru di server tanpa interaksi manual.

### Konsep

Alur deploy yang dapat diulang:

1. Runner menarik image baru dengan tag spesifik.
2. Mengganti versi image di server (lewat image tag di `.env`).
3. Recreate container.
4. Menjalankan health check.
5. Gagal → kembali ke tag sebelumnya.

Server **tidak** membangun image. Server hanya menarik dan menjalankan — build di runner, jalankan di server.

Kunci SSH khusus deploy: satu key pair, hanya untuk pipeline, tanpa passphrase, disimpan sebagai secret. Public key-nya ditaruh di `~/.ssh/authorized_keys` milik user `deploy`.

### Praktik

1. Siapkan key khusus deploy.

```bash
ssh-keygen -t ed25519 -C "gha-deploy" -f ~/.ssh/gha_deploy -N ""
cat ~/.ssh/gha_deploy.pub    # tempel ke server
ssh-copy-id -i ~/.ssh/gha_deploy.pub deploy@203.0.113.10
```

Simpan isi `~/.ssh/gha_deploy` (private key) sebagai secret `SSH_KEY` di GitHub, dan hapus salinan lokalnya kalau tidak dipakai lagi.

2. Di server, siapkan direktori dan file Compose yang memakai image dari registry.

```bash
mkdir -p ~/apps/app && cd ~/apps/app
cat > compose.yaml <<'YAML'
services:
  api:
    image: ${IMAGE:-ghcr.io/USER/REPO:latest}
    restart: unless-stopped
    env_file: .env
    ports:
      - "127.0.0.1:3000:3000"
YAML
cat > .env <<'ENV'
IMAGE=ghcr.io/USER/REPO:latest
ENV
chmod 600 .env
```

`IMAGE` adalah satu-satunya variabel yang ditulis ulang oleh pipeline. Variabel rahasia lain (mis. `DB_PASSWORD`) juga tinggal di `.env` yang sama — karena itu pipeline mengubah **satu baris** memakai `sed`, bukan menimpa seluruh file.

3. Workflow deploy.

```yaml
name: Deploy

on:
  workflow_run:
    workflows: ["Build and push"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-24.04
    environment: production
    concurrency:
      group: deploy-production
      cancel-in-progress: false
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        env:
          GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}   # diteruskan ke server lewat `envs`, tidak ditempel ke skrip
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          envs: GHCR_TOKEN
          script: |
            set -e
            cd ~/apps/app
            echo "$GHCR_TOKEN" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            export IMAGE=ghcr.io/${{ github.repository }}:sha-${{ github.event.workflow_run.head_sha }}
            # Ubah hanya baris IMAGE supaya variabel lain di .env tidak terhapus
            if grep -q '^IMAGE=' .env; then sed -i "s|^IMAGE=.*|IMAGE=$IMAGE|" .env; else echo "IMAGE=$IMAGE" >> .env; fi
            docker compose pull
            docker compose up -d --remove-orphans
            for i in $(seq 1 10); do
              curl -fsS http://127.0.0.1:3000/health && break
              sleep 3
            done
            docker image prune -f
```

Keterangan penting:

- `set -e` di awal skrip menghentikan eksekusi pada error pertama. Jangan bergantung pada input `script_stop` — input itu tidak tersedia di semua versi action; `set -e` selalu bekerja.
- Tag harus konsisten dengan pelajaran 4.5. `type=sha` default menghasilkan 7 karakter, sedangkan `workflow_run.head_sha` 40 karakter. Karena itu metadata memakai `format=long` dan deploy memakai `sha-${{ ...head_sha }}` — keduanya 40 karakter.
- `envs: GHCR_TOKEN` mengirim token sebagai environment variable, sehingga token tidak ikut tercetak di skrip yang dijalankan di server.

4. Alternatif tanpa action pihak ketiga: pakai `ssh-agent` + `ssh` dari runner. Lebih banyak kode, lebih sedikit dependensi.

5. Login GHCR di server: buat token dengan scope `read:packages` dan simpan sebagai secret `GHCR_TOKEN`. Untuk package publik, langkah login bisa dihapus.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `Host key verification failed` | known_hosts belum ada | Tambahkan `fingerprint` atau `ssh-keyscan` pada action |
| `Permission denied (publickey)` | Public key salah tempat, atau secret berisi key yang salah | Bandingkan `ssh-keygen -y -f key` dengan isi `authorized_keys` |
| Deploy sukses tapi app lama | Tag image tidak berubah | Pastikan `.env` benar-benar diperbarui dan `up -d` dijalankan |
| `docker: command not found` di server | Docker tidak ada di `PATH` user non-login | Pakai path absolut atau pastikan user di grup `docker` |
| `denied` saat pull di server | Token tidak punya `read:packages` | Buat token baru dengan scope itu |

### Latihan

1. Deploy satu kali lewat workflow dan tunjukkan `docker compose ps` di server menampilkan image dengan tag sha baru.
2. Jalankan workflow dengan secret SSH yang salah, catat pesan errornya, lalu perbaiki.

**Kriteria selesai:** push ke `main` berakhir dengan versi baru berjalan di `https://app.example.com` tanpa kamu SSH manual.

### Cek pemahaman

1. Kenapa server tidak membangun image sendiri? — Membebani server, dan hasil build tidak deterministik antar mesin.
2. Kenapa deploy memakai key terpisah, bukan key pribadi? — Bisa dicabut tanpa mengganggu akses lain, dan hanya dipakai pipeline.
3. Fungsi health check setelah `up -d`? — Memastikan versi baru benar-benar melayani sebelum dianggap sukses.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat workflow deploy lewat SSH memakai key khusus, token dikirim lewat `envs:`, dan health check setelah `up -d`.

**Siapkan:** Pastikan `.env` di server tidak kehilangan variabel lain saat tag diganti.

**Bawa bukti:** `docker compose ps` di server dengan tag commit terbaru + isi `.env` (nilai rahasia disensor) + log deploy.

Dibahas di awal **Pelajaran 4.7** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.7 — Deploy aman dan rollback

### Tujuan

Membuat deploy bisa gagal dengan aman, dan bisa dibalik cepat.

### Konsep

- Deploy hanya dari `main` (atau tag rilis), tidak dari PR.
- `concurrency` mencegah dua deploy berjalan bersamaan.
- Health check + smoke test menentukan sukses/gagal.
- Rollback = deploy ulang tag sebelumnya, bukan memperbaiki di server.

### Praktik

1. Skrip deploy dengan rollback.

```bash
cat > ~/apps/app/deploy.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

NEW_IMAGE="$1"
PREV_IMAGE="$(sed -n 's/^IMAGE=//p' .env | head -1)"

# Ubah HANYA baris IMAGE; variabel lain di .env (mis. DB_PASSWORD) tetap utuh
set_image() {
  if grep -q '^IMAGE=' .env; then
    sed -i "s|^IMAGE=.*|IMAGE=$1|" .env
  else
    echo "IMAGE=$1" >> .env
  fi
}

echo "deploy: $NEW_IMAGE (sebelumnya: ${PREV_IMAGE:-none})"
set_image "$NEW_IMAGE"
docker compose pull
docker compose up -d --remove-orphans

for i in $(seq 1 10); do
  if curl -fsS http://127.0.0.1:3000/health >/dev/null; then
    echo "health check OK"; exit 0
  fi
  sleep 3
done

echo "health check GAGAL, rollback ke ${PREV_IMAGE:-none}" >&2
docker compose logs --tail 50 api >&2 || true
if [ -n "$PREV_IMAGE" ]; then
  set_image "$PREV_IMAGE"
  docker compose up -d --remove-orphans
fi
exit 1
SH
chmod +x ~/apps/app/deploy.sh
```

2. Panggil dari workflow.

```yaml
          script: |
            set -e
            cd ~/apps/app
            ./deploy.sh "ghcr.io/${{ github.repository }}:sha-${{ github.event.workflow_run.head_sha }}"
```

3. Smoke test setelah deploy, dari runner.

```yaml
      - name: Smoke test publik
        run: |
          for i in $(seq 1 5); do
            code=$(curl -s -o /dev/null -w '%{http_code}' https://app.example.com/health || true)
            [ "$code" = "200" ] && exit 0
            sleep 5
          done
          echo "smoke test gagal (kode terakhir: $code)"; exit 1
```

4. Rollback manual saat butuh (tetap cara yang sama).

```bash
ssh deploy@203.0.113.10 'cd ~/apps/app && ./deploy.sh ghcr.io/USER/REPO:sha-<commit-sebelumnya>'
```

5. Batasi akses: environment `production` dengan required reviewer, dan `concurrency.group` yang sama untuk semua deploy.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Dua deploy bertabrakan | `concurrency` tidak diatur | Tambahkan grup concurrency per environment |
| Rollback juga gagal | Tag lama sudah tidak ada di registry | Simpan minimal beberapa tag terakhir, jangan hapus otomatis |
| Health check hijau tapi situs rusak | Health check terlalu dangkal | Tambahkan pengecekan endpoint yang menyentuh database |
| Job deploy jalan untuk PR | `on:` terlalu longgar | Batasi ke `push` branch `main` atau lewat `workflow_run` |

### Latihan

1. Terapkan `deploy.sh` beserta rollback otomatis, lalu sengaja deploy image yang tidak punya `/health` dan buktikan rollback terjadi.
2. Tambahkan `concurrency` dan jelaskan skenariobe yang dicegahnya.

**Kriteria selesai:** kamu bisa menunjukkan log deploy yang gagal health check dan server kembali ke versi lama.

### Cek pemahaman

1. Kenapa tidak memperbaiki langsung di server? — Perubahan hilang pada deploy berikutnya dan tidak tercatat.
2. Apa yang dicegah `concurrency`? — Dua deploy paralel yang saling menimpa.
3. Kenapa tag lama harus disimpan? — Agar rollback selalu mungkin.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Buat `deploy.sh` dengan health check dan rollback otomatis, plus `concurrency` pada workflow deploy.

**Siapkan:** Uji dengan sengaja men-deploy tag yang tidak punya `/health`.

**Bawa bukti:** Log deploy yang gagal health check + bukti server kembali ke versi sebelumnya + penjelasan fungsi `concurrency`.

Dibahas di awal **Pelajaran 4.8** — 10 menit pertama, sebelum materi baru.

---

## Pelajaran 4.8 — Menaikkan level

### Tujuan

Menghilangkan duplikasi dan menambah visibilitas tanpa menambah kerumitan.

### Konsep

- **Reusable workflow** (`workflow_call`): satu workflow dipanggil beberapa repo/alur.
- **Composite action**: sekumpulan step dibungkus jadi satu action lokal.
- **`needs`**: mengurutkan job dan berbagi `outputs`.
- **Badge**: status workflow di README.
- **Notifikasi**: webhook ke Discord/Slack saat deploy gagal.

### Praktik

1. Mengurutkan job dan berbagi nilai.

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    outputs:
      image: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4        # tanpa ini `git rev-parse` gagal: bukan repo git
      - id: meta
        run: echo "version=sha-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - run: echo "deploy ${{ needs.build.outputs.image }}"
```

2. Reusable workflow.

```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '22'
jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '${{ inputs.node-version }}', cache: npm }
      - run: npm ci && npm test
```

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '22'
```

3. Notifikasi kegagalan ke webhook.

```yaml
      - name: Notifikasi gagal
        if: failure()
        run: |
          curl -fsS -X POST "${{ secrets.DISCORD_WEBHOOK }}" \
            -H 'Content-Type: application/json' \
            -d "{\"content\":\"Deploy gagal: ${{ github.repository }}@${{ github.sha }}\"}" || true
```

4. Badge di README.

```markdown
![CI](https://github.com/USER/REPO/actions/workflows/ci.yml/badge.svg)
```

5. Least privilege dan retensi.

```yaml
permissions:
  contents: read
```

Batasi log: repo → Settings → Actions → Artifact and log retention.

### Kalau error

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| `workflow_call` tidak dikenali | Reusable workflow harus punya `on: workflow_call` dan dipanggil dari job `uses` (bukan step) | Perbaiki strukturnya |
| Output kosong | `id` step tidak cocok dengan `steps.<id>.outputs`, atau nilai ditulis ke `$GITHUB_ENV` padahal dibaca sebagai output step | Tulis ke `$GITHUB_OUTPUT` dan pastikan `id` sama |
| Notifikasi terkirim saat sukses | `if:` salah | Gunakan `if: failure()` atau `if: cancelled()` |

### Latihan

1. Pisahkan test ke reusable workflow dan panggil dari dua workflow berbeda.
2. Tambahkan badge dan buktikan statusnya berubah setelah pipeline gagal.

**Kriteria selesai:** kamu bisa menjelaskan kapan reusable workflow lebih baik daripada copy-paste.

### Cek pemahaman

1. Kapan composite action lebih tepat daripada reusable workflow? — Saat yang dibagi hanya beberapa langkah, bukan keseluruhan job.
2. Kenapa `if: failure()` perlu dipasang eksplisit? — Step default tidak berjalan setelah kegagalan.

### Tugas untuk pertemuan berikutnya

**Kerjakan:** Pisahkan test ke reusable workflow, tambahkan badge status di README, dan notifikasi kegagalan ke webhook.

**Siapkan:** Siapkan repo yang akan dipakai untuk proyek akhir (aplikasi + Dockerfile + compose).

**Bawa bukti:** Badge yang berubah status + satu notifikasi gagal yang benar-benar terkirim + struktur workflow akhir.

Dibahas di awal **Pelajaran 5.1** — 10 menit pertama, sebelum materi baru.

---

## Proyek mini modul

Bangun pipeline lengkap untuk aplikasi modul 3:

1. CI: `npm ci`, lint, test, matrix dua versi Node, berjalan pada `push` ke `main` dan semua `pull_request`.
2. Build: image multi-stage, push ke GHCR dengan tag `sha-<commit>` dan `latest`, cache `type=gha`, `permissions` minimum.
3. Deploy: SSH ke VPS memakai key khusus deploy, environment `production`, `concurrency` aktif.
4. Verifikasi: health check di server + smoke test publik; gagal → rollback otomatis.
5. Notifikasi kegagalan ke Discord/Slack, badge di README.
6. Tulis `catatan-cicd.md`: nama workflow, secret yang dipakai, cara rollback manual, dan cara membaca log.

**Kriteria selesai:** `git push` ke `main` membuat versi baru melayani `https://app.example.com/health` dalam sekali jalan, dan versi lama bisa dikembalikan dengan satu perintah.

## Cheat sheet workflow

| Kebutuhan | Sintaks |
| --- | --- |
| Pemicu | `on: {push: {branches: [main]}, pull_request:, workflow_dispatch:}` |
| Runner | `runs-on: ubuntu-24.04` |
| Ambil kode | `uses: actions/checkout@v4` |
| Cache npm | `actions/setup-node@v4` + `cache: npm` |
| Variabel matrix | `${{ matrix.node }}` |
| Output antar job | `outputs:` + `${{ needs.job.outputs.x }}` |
| Secret / variable | `${{ secrets.X }}` / `${{ vars.X }}` |
| Izin token | `permissions: {contents: read, packages: write}` |
| Login GHCR | `docker/login-action@v3`, `registry: ghcr.io`, password `${{ secrets.GITHUB_TOKEN }}` |
| Build & push | `docker/build-push-action@v6` + `cache-from/to: type=gha` |
| Deploy SSH | `appleboy/ssh-action@v1` + `envs:` + skrip diawali `set -e` |
| Hanya saat gagal | `if: failure()` |
| Anti tabrakan | `concurrency: {group: deploy-production, cancel-in-progress: false}` |

## Referensi

- GitHub Actions documentation — https://docs.github.com/en/actions
- Workflow syntax — https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions
- Metadata action (tag) — https://github.com/docker/metadata-action
- build-push-action — https://github.com/docker/build-push-action
- GHCR — https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- appleboy/ssh-action — https://github.com/appleboy/ssh-action
- Encrypted secrets — https://docs.github.com/en/actions/security-guides/encrypted-secrets
