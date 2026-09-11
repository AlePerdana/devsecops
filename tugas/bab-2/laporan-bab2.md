# Laporan Bab 2 

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

1. Menjelaskan perbedaan virtual machine dan container dari sisi isolasi, ukuran, startup time, dan overhead.
2. Mengidentifikasi komponen Docker: client, daemon, registry, image, container, network, dan volume.
3. Menginstal Docker Engine serta menjalankan container pertama, melakukan inspeksi image, melihat log, dan membangun image custom sederhana.
4. Memahami konfigurasi jaringan dan penyimpanan container beserta implikasi keamanannya.

---

## 2. Alat dan Bahan

| No | Komponen | Keterangan |
| --- | --- | --- |
| 1 | Host Linux / Mesin Virtual Ubuntu | Lingkungan khusus laboratorium |
| 2 | Docker Engine, Docker CLI | Runtime container |
| 3 | containerd dan runc | High-level dan low-level runtime |
| 4 | Plugin Buildx dan Compose | Build serta orkestrasi container |
| 5 | Image `hello-world`, `nginx:1.26`, `ubuntu:22.04`, `nginx:1.26-alpine` | Bahan praktikum |
| 6 | curl dan browser | Verifikasi akses layanan web |
| 7 | Terminal / Shell | Eksekusi perintah |

---

## 3. Langkah Praktikum

### 3.0 Menyiapkan Direktori Kerja

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan.

```bash
mkdir -p ~/docker-lab/bab-2
cd ~/docker-lab/bab-2
```

### 3.1 Praktikum 1 — Instalasi Docker Engine Ubuntu

Langkah berikut dijalankan secara berurutan, lalu diverifikasi sebelum melanjutkan.

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker version
```

Hasil:

![Verifikasi client dan server Docker](assets/1.png)

*Gambar 1. Keluaran `docker version` menampilkan client dan server Docker Engine.*

```bash
docker run hello-world
```

Hasil:

![Eksekusi image hello-world](assets/2.png)

*Gambar 2. Eksekusi image `hello-world` sebagai verifikasi instalasi.*

### 3.2 Praktikum 2 — Container Nginx dan Ubuntu Interaktif

```bash
docker pull nginx:1.26
```

Hasil:

![Unduh image nginx:1.26](assets/3.png)

*Gambar 3. Proses `docker pull nginx:1.26`.*

Menjalankan container web, kemudian memeriksa status dan log-nya:

```bash
docker run -d --name web-public -p 8080:80 nginx:1.26
docker ps
docker logs --tail 20 web-public
```

Hasil:

![Container Nginx berjalan dan log-nya](assets/4.png)

*Gambar 4. Container `web-public` berjalan dengan pemetaan port 8080 ke 80, beserta log eksekusi.*

Mengakses layanan dari host:

```bash
curl http://localhost:8080
```

Hasil:

![Hasil curl ke localhost:8080](assets/5.png)

*Gambar 5. Hasil `curl http://localhost:8080` menampilkan halaman default Nginx.*

Menguji shell interaktif pada container:

```bash
docker run -it --name ubuntu-test ubuntu:22.04 /bin/bash
cat /etc/os-release
exit
docker rm -f web-public ubuntu-test
```

Hasil:

![Akses shell interaktif pada container ubuntu:22.04](assets/6.png)

*Gambar 6. Pemeriksaan `/etc/os-release` di dalam container `ubuntu:22.04`.*

### 3.3 Praktikum 3 — Dockerfile Custom Web Statis

```bash
mkdir -p ~/docker-lab/custom-web && cd ~/docker-lab/custom-web
cat > index.html << 'EOF'
<h1>Docker Lab PENS</h1>
<p>Container berhasil berjalan.</p>
EOF
cat > Dockerfile << 'EOF'
FROM nginx:1.26-alpine
LABEL maintainer="admin@pens.ac.id"
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

Hasil:

![Pembuatan index.html dan Dockerfile](assets/7.png)

*Gambar 7. Pembuatan `index.html` dan `Dockerfile` pada direktori `custom-web`.*

Melakukan build image dan menjalankan container hasil build:

```bash
docker build -t pens-web:1.0 .
docker run -d --name pens-app -p 9090:80 pens-web:1.0
curl http://localhost:9090
```

Hasil:

![Proses build dan hasil eksekusi image pens-web:1.0](assets/8.png)

*Gambar 8. Proses build image `pens-web:1.0`, eksekusi container `pens-app`, dan hasil `curl http://localhost:9090`.*

---

## 4. Verifikasi dan Skenario Pengujian

- [ ] Docker Engine aktif dan `docker version` menampilkan Client serta Server.
- [ ] User non-root dapat menjalankan `docker ps` tanpa `sudo`.
- [ ] Container Nginx dapat diakses dari browser melalui port host.
- [ ] Image `pens-web:1.0` berhasil dibangun dan dijalankan.
- [ ] Mahasiswa dapat menjelaskan perbedaan `EXPOSE` dan `-p`.

Prinsip troubleshooting: mulai dari status container, baca logs, cek network, cek volume, lalu validasi konfigurasi. Jangan langsung menghapus volume sebelum memahami apakah data masih dibutuhkan.

```bash
docker compose ps
docker compose logs --tail 100
curl -v http://localhost:8080
docker network ls
docker volume ls
docker inspect <container-name>
```

---

## 5. Hasil dan Pembahasan

### 5.1 Praktikum 1 — Instalasi Docker Engine

Dari instalasi Docker dan percobaan menjalankan sebuah container, hasilnya untuk lingkungan host memiliki daemon Docker yang aktif dan dapat diakses oleh pengguna biasa (non-root) tanpa memerlukan perintah `sudo` setiap kali eksekusi.

### 5.2 Praktikum 2 — Container Nginx dan Ubuntu Interaktif

Langkah ini mendemonstrasikan siklus hidup container, penggunaan network namespace untuk akses port dari host ke container (pemetaan port 8080 ke 80), serta pengambilan rekam jejak eksekusi (logs).

### 5.3 Praktikum 3 — Dockerfile Custom Web Statis

Langkah ini mendokumentasikan proses otomasi pembuatan image menggunakan layer yang read-only, di mana instruksi `COPY` menambahkan konten spesifik di atas base image yang sudah ada.

---

## 6. Analisis Wajib

### 6.1 Masalah yang Muncul dan Cara Mendiagnosisnya

**Masalah:** Muncul pesan error `Permission denied` saat mencoba menjalankan perintah `docker ps` atau `docker run` tanpa `sudo`.

**Jawaban:** Mengacu pada prinsip troubleshooting keamanan, masalah ini diperiksa dengan melihat konfigurasi grup sistem. Pesan tersebut menunjukkan pengguna saat ini belum memiliki akses ke soket Docker. Solusinya adalah memverifikasi keanggotaan pengguna dalam grup Docker (`groups $USER`), kemudian menjalankan `sudo usermod -aG docker $USER` dilanjutkan dengan `newgrp docker` agar sesi terminal diperbarui.

### 6.2 Risiko Keamanan atau Operasional yang Relevan

Risiko operasional dan keamanan tertinggi berkaitan dengan akses daemon dan lifecycle image. Keanggotaan di dalam grup `docker` secara praktis memberikan privilege yang setara dengan akses root di sistem host, sehingga jika akun pengguna diretas, host dapat dikompromi penuh. Selain itu, batas isolasi container tidak sekuat Virtual Machine karena container menggunakan kernel yang sama dengan host, sehingga attack surface daemon harus dibatasi ketat.

### 6.3 Rekomendasi Perbaikan untuk Production-like Environment

Untuk lingkungan produksi, konfigurasi arsitektur container harus diperkeras (*hardened*). **Pertama**, gunakan *rootless mode* untuk memitigasi risiko daemon mengeksekusi proses sebagai root pada host. **Kedua**, gunakan tag digest yang definitif, bukan `latest`, serta terapkan *multi-stage build* untuk meminimalkan attack surface dan ukuran image. **Ketiga**, atur kuota memori dan CPU menggunakan konfigurasi cgroup (misalnya penambahan `--memory` atau `--cpus`) untuk mencegah satu container menghabiskan resource sistem secara penuh.

---

## 7. Evaluasi dan Latihan Mandiri

**1. Mengapa penggunaan tag `latest` tidak dianjurkan untuk deployment yang harus reproducible?**

Tag `latest` hanyalah sebuah label dinamis yang dapat berpindah untuk menunjuk ke versi baru kapan saja. Penggunaannya menyebabkan hilangnya reproduksibilitas karena pipeline atau lingkungan produksi mungkin menarik versi peranti lunak yang berbeda di hari yang berbeda, yang dapat mengubah perilaku aplikasi. Reproducible deployment memerlukan pengikatan ke versi spesifik agar perubahan dapat dilacak secara eksplisit.

**2. Jelaskan peran containerd dan runc dalam arsitektur Docker.**

Dalam arsitektur runtime modern, daemon Docker mendelegasikan manajemen lifecycle kontainer tingkat tinggi kepada containerd. Selanjutnya, containerd akan memerintahkan low-level runtime bernama runc untuk benar-benar berinteraksi dengan kernel OS guna membentuk batas isolasi sesuai dengan OCI Runtime Specification.

**3. Apa konsekuensi keamanan dari memasukkan user ke group `docker`?**

Memasukkan user ke dalam grup `docker` memberikan kemampuan mengeksekusi perintah pada daemon Docker yang berjalan sebagai root. Akses ini setara dengan hak root tanpa `sudo`, karena user tersebut dapat dengan mudah meluncurkan container dengan flag `--privileged` atau melakukan bind mount pada filesystem host, yang memungkinkan pengambilalihan sistem host secara menyeluruh.

**4. Bandingkan layer image `nginx:1.26-alpine` dan image custom yang Anda buat.**

Image `nginx:1.26-alpine` bertindak sebagai base image atau fondasi awal yang berisi layer-layer sistem operasi minimal Alpine dan instalasi Nginx standar. Sedangkan image custom (`pens-web:1.0`) berisi seluruh layer dari base image tersebut, ditambah dengan layer baru yang tercipta dari eksekusi instruksi `COPY` untuk memasukkan dokumen `index.html` kustom ke dalam direktori Nginx.

**5. Kapan sebaiknya memilih VM daripada container?**

Mesin Virtual (VM) sebaiknya dipilih apabila lingkungan membutuhkan isolasi infrastruktur dan boundary keamanan yang jauh lebih ketat melalui hypervisor.

---

## 8. Troubleshooting dan Analisis Hasil

| Gejala | Penyebab yang mungkin | Tindakan korektif |
| --- | --- | --- |
| Gate berbeda antara lokal dan pipeline | Versi tool, input efektif, atau konfigurasi tidak sama | Pin versi; simpan konfigurasi efektif dan identitas artefak. |
| Service sehat tetapi security gate gagal | Healthcheck hanya memeriksa availability | Tinjau policy, scan, identity, signature, dan evidence secara terpisah. |
| Evidence tidak dapat ditelusuri | Commit, digest, waktu, atau owner tidak dicatat | Gunakan manifest evidence dan metadata yang konsisten. |
| Deployment gagal dipulihkan | Rollback, backup, atau credential rotation belum diuji | Lakukan recovery exercise dan dokumentasikan hasilnya. |

---

## 9. Kesimpulan

1. Container adalah mekanisme isolasi proses berbasis kernel yang membungkus aplikasi bersama dependensinya; karena berbagi kernel host, container lebih ringan dan cepat dibanding virtual machine, tetapi boundary isolasinya perlu hardening tambahan.
2. Docker Engine berhasil diinstal dan daemon dapat diakses oleh pengguna non-root setelah keanggotaan grup `docker` diterapkan, dengan verifikasi melalui `docker version` dan eksekusi image `hello-world`.
3. Siklus hidup container berhasil didemonstrasikan melalui container Nginx dengan pemetaan port 8080 ke 80, pemeriksaan status via `docker ps`, pembacaan log, serta akses shell interaktif pada `ubuntu:22.04`.
4. Image custom `pens-web:1.0` berhasil dibangun dari base `nginx:1.26-alpine` dan dijalankan pada port 9090, menunjukkan bahwa instruksi `COPY` menambahkan layer baru di atas layer base image yang bersifat read-only.
5. Keanggotaan dalam grup `docker` setara dengan akses root pada host; untuk lingkungan produksi diperlukan rootless mode, penggunaan digest alih-alih `latest`, multi-stage build, serta pembatasan resource melalui cgroup.
