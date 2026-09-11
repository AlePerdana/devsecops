# Laporan Bab 3

<div align="center">
  <h1 style="text-align: center;font-weight: bold">LAPORAN RESMI<br>WORKSHOP DEVOPS</h1>
  <h4 style="text-align: center;">Dosen Pengampu : Dr. Ferry Astika Saputra, S.T., M.Sc.</h4>
</div>
<br />
<div align="center">
  <img src="https://upload.wikimedia.org/wikipedia/id/4/44/Logo_PENS.png" alt="Logo PENS">
  <h3 style="text-align: center;">Disusun Oleh : </h3>
  <p style="text-align: center;">
    <strong>Ale Perdana Putra Darmawan (3126640016) </strong><br>
  </p>
<h3 style="text-align: center;line-height: 1.5">Politeknik Elektronika Negeri Surabaya<br>Departemen Teknik Informatika Dan Komputer<br>Program Studi Teknik Informatika<br>2026</h3>
  <hr><hr>
</div>

---

## 1. Tujuan Praktikum

1. Membuat user-defined bridge network dan membuktikan resolusi nama antar container.
2. Membedakan volume, bind mount, dan tmpfs dari sisi persistensi, portabilitas, dan keamanan.
3. Menulis file Compose untuk aplikasi multi-container yang memiliki service, network, volume, dan healthcheck.
4. Mengelola lifecycle aplikasi dengan `docker compose up`, `ps`, `logs`, `stop`, `start`, `down`, dan `down -v`.

---

## 2. Alat dan Bahan

| No | Komponen | Keterangan |
| --- | --- | --- |
| 1 | Host Linux / Mesin Virtual Ubuntu | Lingkungan khusus laboratorium |
| 2 | Docker Engine dan Docker Compose (plugin v2) | Runtime dan orkestrasi container satu host |
| 3 | Image `nginx:alpine`, `alpine:3.20`, `postgres:16-alpine`, `python:3.12-slim` | Bahan praktikum network, volume, dan Compose |
| 4 | curl dan browser | Verifikasi endpoint aplikasi |
| 5 | Terminal / Shell | Eksekusi perintah |

---

## 3. Ringkasan Eksekusi

Direktori kerja dan struktur berkas:

```bash
mkdir -p ~/docker-lab/bab-3/{app,html}
cd ~/docker-lab/bab-3
```

```text
bab-3/
├── compose.yaml
├── nginx.conf
├── html/
│   └── index.html
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

Model aplikasi tiga lapis dinyatakan pada `compose.yaml`:

- `web` (`nginx:alpine`) — reverse proxy, port publikasi dibatasi ke loopback `127.0.0.1:8080:80`;
- `app` — Flask + gunicorn hasil build `./app`, berjalan sebagai user non-root (`USER appuser`);
- `db` (`postgres:16-alpine`) — healthcheck `pg_isready`, data pada named volume `pg-data`;
- bind mount `./html` dan `./nginx.conf` diberi opsi `:ro`;
- network terpisah: `frontend` (web–app) dan `backend` (app–db);
- `app` menunggu database dengan `condition: service_healthy`.

Validasi dan menjalankan stack:

```bash
docker compose config --services
docker compose up -d --build
```

---

## 4. Bukti Eksekusi

### 4.1 Status Service Berjalan

```bash
docker compose ps
```

Hasil:

![Status service Compose](assets/1.png)

*Gambar 1. `docker compose ps` menunjukkan service `web` dan `app` berstatus `Up` serta `db` berstatus `Up (healthy)`.*

### 4.2 Hasil Pengujian Endpoint

```bash
curl http://localhost:8080/
curl -i http://localhost:8080/health
curl http://localhost:8080/static.html
```

Keluaran yang diharapkan pada endpoint utama:

```json
{
  "database": "PostgreSQL 16...",
  "message": "Nginx, Flask, dan PostgreSQL berhasil terhubung",
  "status": "ok"
}
```

Hasil:

![Hasil curl endpoint aplikasi](assets/2.png)

*Gambar 2. Endpoint `/` mengembalikan `status: ok` beserta versi PostgreSQL, dan `/health` mengembalikan HTTP 200 `healthy`.*

### 4.3 Cuplikan Log/Query yang Membuktikan Sistem Bekerja

```bash
docker compose logs --tail 100
docker compose logs app
docker compose logs db
```

Hasil:

![Cuplikan log service](assets/3.png)

*Gambar 3. Log menunjukkan request Nginx diteruskan ke gunicorn dan query `SELECT 1;` berhasil dijawab PostgreSQL.*

---

## 5. Analisis Wajib

### 5.1 Masalah yang Muncul dan Cara Mendiagnosisnya

**Masalah:** Nginx menampilkan `502 Bad Gateway` ketika endpoint aplikasi diakses, atau container `db` berstatus `unhealthy`.

**Cara mendiagnosis:** Sesuai prinsip troubleshooting, pemeriksaan dilakukan berlapis mulai dari status container, log, network, volume, lalu validasi konfigurasi. Langkah pertama adalah `docker compose ps` untuk melihat status service, kemudian `docker compose logs app` dan `docker compose logs db` untuk memastikan apakah aplikasi Flask gagal terhubung ke database. `502 Bad Gateway` menandakan Nginx tidak menemukan upstream yang sehat pada `app:5000`, umumnya karena aplikasi belum siap atau gagal berjalan. Container `db` yang `unhealthy` dapat disebabkan oleh password, nama database, atau healthcheck yang salah; apabila perubahan password tidak berlaku, penyebabnya adalah volume lama yang masih menyimpan database sebelumnya. Validasi akhir dilakukan dengan `docker compose config`, dan reset hanya dilakukan secara sadar melalui `docker compose down -v`.

### 5.2 Risiko Keamanan atau Operasional yang Relevan

Risiko utama pada bab ini adalah **perluasan attack surface akibat publikasi port dan mount yang terlalu permisif**. Konfigurasi `-p 8080:80` berpotensi mengikat seluruh interface host sehingga layanan dapat dijangkau dari jaringan eksternal, sedangkan `127.0.0.1:8080:80` membatasinya ke loopback. Bind mount memiliki akses tulis secara default sehingga proses container dapat mengubah atau menghapus berkas host, dan mount ke direktori container yang sudah berisi file akan menutupi isi tersebut selama mount aktif. Kredensial pada `environment` (`POSTGRES_PASSWORD`, `DB_PASS`) juga berpotensi terlihat melalui inspect, process environment, log, maupun crash report. Perlu ditegaskan bahwa network segmentation tidak menggantikan authorization: service `app` yang terhubung ke `frontend` dan `backend` merupakan jalur yang sah antara dua zona, sehingga bila aplikasi dikompromikan, penyerang dapat menggunakan jalur backend tersebut. Secara operasional, `docker compose down -v` bersifat destruktif karena menghapus volume `pg-data` yang menyimpan data PostgreSQL.

### 5.3 Rekomendasi Perbaikan untuk Production-like Environment

**Pertama**, terapkan prinsip least exposure: publikasikan hanya satu ingress yang diperlukan dan ikat ke alamat spesifik (`127.0.0.1:8080:80`), sementara database dan dashboard administratif tetap internal. **Kedua**, gunakan mount `:ro` bila write tidak diperlukan, pertimbangkan filesystem `read_only`, jalankan container sebagai user non-root (sudah diterapkan melalui `USER appuser`), dan batasi capability seminimal mungkin. **Ketiga**, pindahkan data sensitif dari `environment` ke Compose secret atau secret manager, batasi permission sumber secret, dan lakukan rotasi setelah penggunaan — password pada praktikum ini bersifat sintetis untuk lingkungan disposable dan tidak boleh dikomit ke repository. **Keempat**, pin image dengan digest atau tag versi spesifik, pertahankan pemisahan network `frontend`/`backend`, serta lengkapi dengan healthcheck yang bermakna, resource limit, dan logging. Terakhir, seluruh kontrol keamanan harus diuji pada model yang telah di-resolve dengan `docker compose config`, bukan hanya pada fragmen YAML.

---

## 6. Kesimpulan

1. Compose stack `web`–`app`–`db` berhasil berjalan dan terbukti terhubung end-to-end (Nginx → Flask → PostgreSQL) melalui `docker compose ps`, pengujian endpoint `curl`, dan cuplikan log.
2. Pembatasan publikasi port ke loopback, mount `:ro`, dan eksekusi aplikasi sebagai user non-root merupakan kontrol keamanan minimum yang diterapkan pada stack ini.
3. Risiko utama bab ini adalah publikasi port yang terlalu luas, bind mount yang dapat menulis ke host, serta kredensial pada environment; untuk production-like diperlukan secret terkelola, least exposure, pemisahan network, dan pinning image.
4. Perintah `down -v` bersifat destruktif terhadap data volume, sehingga penggunaannya harus disadari dan dibatasi pada lingkungan disposable.
