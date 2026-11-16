# LAPORAN PRAKTIKUM BAB 8
## Hardening Container, Secrets, Trivy, dan Private Registry

**Nama**: Ale Perdana Putra Darmawan  
**NIM**: 3126640016  
**Kelas**: STr LJ Teknik Informatika A  
**Tanggal pelaksanaan**: 15 November 2026

> **Penggunaan AI.** Praktikum ini dijalankan dengan bantuan AI untuk eksekusi perintah, pencatatan output, penyusunan dokumentasi, dan analisis.

## Kotak Perintah A — Menyiapkan Direktori Bab 8

```bash
mkdir -p ~/docker-lab/bab-8/{app,secrets,registry/certs,registry/auth,scripts,reports}
cd ~/docker-lab/bab-8
touch reports/.gitkeep
```

Struktur akhir:

```text
bab-8/
├── .env.example
├── .env
├── .gitignore
├── .dockerignore
├── compose.yaml
├── app/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── secrets/
│   ├── app_secret.txt.example
│   └── app_secret.txt
├── registry/
│   ├── auth/htpasswd
│   └── certs/
│       ├── domain.crt
│       └── domain.key
├── scripts/
│   ├── generate-registry-assets.sh
│   ├── scan-image.sh
│   ├── push-registry.sh
│   └── verify-hardening.sh
└── reports/
```


Hasil:


![Dokumentasi A-1](assets/A-1-1.png)


Analisa: Direktori aplikasi, secret, aset registry, script, dan laporan berhasil disiapkan.


## Kotak Script 1 — File `.env.example`

> **Nama file:** `.env.example`  
> **Lokasi:** `~/docker-lab/bab-8/.env.example`  
> **Buka editor:** `nano .env.example`

```dotenv
APP_IMAGE=pens-web:1.0
REGISTRY_HOST=localhost:5000
REGISTRY_USER=labpublisher
REGISTRY_PASSWORD=registrypass123
```

Aktifkan konfigurasi lokal:

```bash
cp .env.example .env
chmod 600 .env
```

Credential tersebut hanya untuk laptop lab. `.env` bukan secret manager dan nilainya tidak boleh digunakan pada lingkungan nyata.


Hasil:


![Dokumentasi script-1](assets/script-1-1.png)


Analisa: Isi `.env.example` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 2 — File `.gitignore`

> **Nama file:** `.gitignore`  
> **Lokasi:** `~/docker-lab/bab-8/.gitignore`  
> **Buka editor:** `nano .gitignore`

```gitignore
.env
secrets/*.txt
!secrets/*.txt.example
registry/auth/*
registry/certs/*
reports/*
!reports/.gitkeep
*.tar
*.log
```

Private key TLS, password hash, secret aktif, arsip image, dan laporan scanner dapat memuat informasi sensitif sehingga tidak dimasukkan ke repository.


Hasil:


![Dokumentasi script-2](assets/script-2-1.png)


Analisa: Isi `.gitignore` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 3 — File `.dockerignore`

> **Nama file:** `.dockerignore`  
> **Lokasi:** `~/docker-lab/bab-8/.dockerignore`  
> **Buka editor:** `nano .dockerignore`

```dockerignore
.git
.gitignore
.env
secrets/
registry/
reports/
scripts/
compose.yaml
**/__pycache__/
**/*.py[cod]
**/*.log
**/*.tar
```

Karena build context pada Compose menggunakan root Bab 8, Dockerfile akan menyalin berkas aplikasi secara eksplisit dari `app/`. `.dockerignore` menjadi lapisan pencegahan tambahan agar secret tidak ikut terkirim ke builder.


Hasil:


![Dokumentasi script-3](assets/script-3-1.png)


Analisa: Isi `.dockerignore` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 4 — File `app/requirements.txt`

> **Nama file:** `requirements.txt`  
> **Lokasi:** `~/docker-lab/bab-8/app/requirements.txt`  
> **Buka editor:** `nano app/requirements.txt`

```text
Flask==3.1.3
gunicorn==26.1.0
```

Versi dipin agar build lebih dapat direproduksi. Untuk pipeline produksi, gunakan lock file beserta hash paket dan proses pembaruan dependency terjadwal.


Hasil:


![Dokumentasi script-4](assets/script-4-1.png)


Analisa: Isi `app/requirements.txt` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 5 — File `app/app.py`

> **Nama file:** `app.py`  
> **Lokasi:** `~/docker-lab/bab-8/app/app.py`  
> **Buka editor:** `nano app/app.py`

```python
import os
from pathlib import Path

from flask import Flask, jsonify


app = Flask(__name__)
SECRET_PATH = Path("/run/secrets/app_secret")


@app.get("/")
def index():
    return jsonify(
        service="pens-web",
        status="running",
        uid=os.getuid(),
        gid=os.getgid(),
        secret_mounted=SECRET_PATH.is_file(),
    )


@app.get("/health")
def health():
    if not SECRET_PATH.is_file():
        return jsonify(status="degraded", reason="secret_missing"), 503
    return jsonify(status="ok"), 200
```

Aplikasi hanya mengonfirmasi keberadaan secret dan tidak mengembalikan nilainya. Logging secret, hash secret, atau sebagian token tetap dapat memperluas informasi yang tersedia bagi penyerang.


Hasil:


![Dokumentasi script-5](assets/script-5-1.png)


Analisa: Isi `app/app.py` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 6 — File `app/Dockerfile`

> **Nama file:** `Dockerfile`  
> **Lokasi:** `~/docker-lab/bab-8/app/Dockerfile`  
> **Buka editor:** `nano app/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11.16-slim AS builder

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /build

COPY app/requirements.txt ./requirements.txt

RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --requirement requirements.txt


FROM python:3.11.16-slim AS runtime

ARG APP_UID=10001
ARG APP_GID=10001

ENV PATH="/opt/venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN groupadd --gid "${APP_GID}" appgroup \
    && useradd \
        --uid "${APP_UID}" \
        --gid "${APP_GID}" \
        --no-create-home \
        --shell /usr/sbin/nologin \
        appuser

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY --chown=10001:10001 app/app.py ./app.py

USER 10001:10001

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2)"]

CMD ["gunicorn", "--bind=0.0.0.0:8080", "--workers=2", "--threads=2", "--worker-tmp-dir=/tmp", "--access-logfile=-", "--error-logfile=-", "app:app"]
```

Multi-stage build memisahkan tahap pemasangan dependency dari runtime. User numerik mengurangi ambiguitas pemetaan akun, tetapi base image dan dependency tetap harus dipindai serta diperbarui.


Hasil:


![Dokumentasi script-6](assets/script-6-1.png)


![Dokumentasi script-6](assets/script-6-2.png)


Analisa: Isi `app/Dockerfile` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 7 — File `secrets/app_secret.txt.example`

> **Nama file:** `app_secret.txt.example`  
> **Lokasi:** `~/docker-lab/bab-8/secrets/app_secret.txt.example`  
> **Buka editor:** `nano secrets/app_secret.txt.example`

```text
GANTI_DENGAN_NILAI_ACAK_KHUSUS_LAB
```

Jangan menyalin nilai contoh sebagai secret aktif. Buat nilai acak:

```bash
openssl rand -hex 32 > secrets/app_secret.txt
chmod 600 secrets/app_secret.txt
```


Hasil:


![Dokumentasi script-7](assets/script-7-1.png)


Analisa: Isi `secrets/app_secret.txt.example` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 8 — File `compose.yaml`

> **Nama file:** `compose.yaml`  
> **Lokasi:** `~/docker-lab/bab-8/compose.yaml`  
> **Buka editor:** `nano compose.yaml`  
> **Petunjuk:** gunakan spasi, bukan tab.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: app/Dockerfile
    image: ${APP_IMAGE}
    user: "10001:10001"
    init: true
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m,mode=1777
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    cpus: 0.50
    mem_limit: 256m
    memswap_limit: 256m
    secrets:
      - app_secret
    ports:
      - "127.0.0.1:8088:8080"
    networks:
      - app-net
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2)"
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    restart: unless-stopped

  registry:
    image: registry:3.1.1
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: DevSecOps Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_DELETE_ENABLED: "false"
      OTEL_TRACES_EXPORTER: none
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=32m
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    ports:
      - "127.0.0.1:5000:5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./registry/certs:/certs:ro
      - ./registry/auth:/auth:ro
    networks:
      - registry-net
    restart: unless-stopped

  trivy:
    image: aquasec/trivy:0.74.0
    profiles:
      - tools
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    volumes:
      - .:/workspace:ro
      - trivy-cache:/root/.cache
    security_opt:
      - no-new-privileges:true

secrets:
  app_secret:
    file: ./secrets/app_secret.txt

volumes:
  registry-data:
  trivy-cache:

networks:
  app-net:
    internal: true
  registry-net:
    internal: true
```

`cap_drop: ALL`, filesystem read-only, `no-new-privileges`, batas PID, CPU, memory, dan swap memperkecil blast radius. Kontrol tersebut harus diuji terhadap fungsi aplikasi; hardening yang membuat fungsi sah gagal bukan konfigurasi yang selesai.


Hasil:


![Dokumentasi script-8](assets/script-8-1.png)


![Dokumentasi script-8](assets/script-8-2.png)


Analisa: Isi `compose.yaml` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 9 — File `scripts/generate-registry-assets.sh`

> **Nama file:** `generate-registry-assets.sh`  
> **Lokasi:** `~/docker-lab/bab-8/scripts/generate-registry-assets.sh`  
> **Buka editor:** `nano scripts/generate-registry-assets.sh`  
> **Tujuan:** membuat sertifikat self-signed dengan SAN `localhost` dan akun registry menggunakan bcrypt.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

: "${REGISTRY_USER:?REGISTRY_USER belum diatur}"
: "${REGISTRY_PASSWORD:?REGISTRY_PASSWORD belum diatur}"

command -v openssl >/dev/null || {
    echo "[FAIL] OpenSSL tidak tersedia." >&2
    exit 1
}

mkdir -p registry/certs registry/auth
umask 077

if [[ -e registry/certs/domain.key || -e registry/auth/htpasswd ]]; then
    echo "[FAIL] Aset registry sudah ada; rotasi harus dilakukan secara sadar." >&2
    exit 1
fi

openssl req \
    -newkey rsa:3072 \
    -nodes \
    -sha256 \
    -x509 \
    -days 365 \
    -keyout registry/certs/domain.key \
    -out registry/certs/domain.crt \
    -subj "/CN=localhost/O=DevSecOps Lab" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
    -addext "extendedKeyUsage=serverAuth"

printf '%s\n' "$REGISTRY_PASSWORD" \
    | docker run --rm -i \
        --entrypoint htpasswd \
        httpd:2.4-alpine \
        -Bni "$REGISTRY_USER" \
    > registry/auth/htpasswd

chmod 600 registry/certs/domain.key registry/auth/htpasswd
chmod 644 registry/certs/domain.crt

echo "[PASS] Sertifikat dan htpasswd berhasil dibuat."
echo "[INFO] Sertifikat berlaku 365 hari untuk localhost."
```

Sertifikat self-signed hanya untuk pembelajaran. Produksi harus memakai nama DNS nyata, CA yang dipercaya, prosedur rotasi, dan perlindungan private key yang sesuai.


Hasil:


![Dokumentasi script-9](assets/script-9-1.png)


![Dokumentasi script-9](assets/script-9-2.png)


Analisa: Isi `scripts/generate-registry-assets.sh` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 10 — File `scripts/scan-image.sh`

> **Nama file:** `scan-image.sh`  
> **Lokasi:** `~/docker-lab/bab-8/scripts/scan-image.sh`  
> **Buka editor:** `nano scripts/scan-image.sh`  
> **Tujuan:** menghasilkan laporan lengkap, SBOM CycloneDX, scan konfigurasi, serta gate untuk temuan HIGH/CRITICAL yang sudah memiliki perbaikan.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

: "${APP_IMAGE:?APP_IMAGE belum diatur}"

mkdir -p reports
archive_name="pens-web-1.0.tar"

docker image inspect "$APP_IMAGE" >/dev/null
docker image save --output "reports/$archive_name" "$APP_IMAGE"

run_trivy() {
    docker compose --profile tools run --rm --no-deps trivy "$@"
}

run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --scanners vuln,secret \
    --image-config-scanners misconfig,secret \
    --format json \
    > reports/trivy-image.json

run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --format cyclonedx \
    > reports/pens-web-1.0.cdx.json

run_trivy config \
    --severity MEDIUM,HIGH,CRITICAL \
    /workspace/app \
    > reports/trivy-config.txt

run_trivy config \
    --severity MEDIUM,HIGH,CRITICAL \
    /workspace/compose.yaml \
    >> reports/trivy-config.txt

run_trivy fs \
    --scanners secret \
    /workspace/app \
    > reports/trivy-secret.txt

set +e
run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --scanners vuln \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --exit-code 1 \
    > reports/trivy-gate.txt
gate_status=$?
set -e

if [[ "$gate_status" -eq 0 ]]; then
    printf 'PASS\n' > reports/gate-status.txt
    echo "[PASS] Security gate lulus."
else
    printf 'BLOCKED\n' > reports/gate-status.txt
    echo "[BLOCKED] Temuan HIGH/CRITICAL yang dapat diperbaiki terdeteksi." >&2
    echo "[INFO] Tinjau reports/trivy-gate.txt dan perbarui image." >&2
    exit 1
fi

docker image inspect "$APP_IMAGE" \
    --format '{{json .RepoDigests}}' \
    > reports/local-image-digests.json

sha256sum \
    reports/trivy-image.json \
    reports/pens-web-1.0.cdx.json \
    reports/trivy-config.txt \
    reports/trivy-secret.txt \
    reports/trivy-gate.txt \
    > reports/SHA256SUMS

echo "[PASS] Laporan, SBOM, dan checksum berhasil dibuat."
```

`--ignore-unfixed` hanya digunakan pada keputusan gate agar lab tidak berhenti pada temuan yang belum mempunyai perbaikan. Temuan tanpa perbaikan tetap tercatat pada `trivy-image.json` dan tetap membutuhkan triage, mitigasi, atau penerimaan risiko yang terdokumentasi.


Hasil:


![Dokumentasi script-10](assets/script-10-1.png)


![Dokumentasi script-10](assets/script-10-2.png)


Analisa: Isi `scripts/scan-image.sh` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 11 — File `scripts/push-registry.sh`

> **Nama file:** `push-registry.sh`  
> **Lokasi:** `~/docker-lab/bab-8/scripts/push-registry.sh`  
> **Buka editor:** `nano scripts/push-registry.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

: "${APP_IMAGE:?APP_IMAGE belum diatur}"
: "${REGISTRY_HOST:?REGISTRY_HOST belum diatur}"
: "${REGISTRY_USER:?REGISTRY_USER belum diatur}"
: "${REGISTRY_PASSWORD:?REGISTRY_PASSWORD belum diatur}"

if [[ ! -f reports/gate-status.txt ]] \
    || [[ "$(< reports/gate-status.txt)" != "PASS" ]]; then
    echo "[BLOCKED] Image belum lulus security gate." >&2
    exit 1
fi

printf '%s\n' "$REGISTRY_PASSWORD" \
    | docker login "$REGISTRY_HOST" \
        --username "$REGISTRY_USER" \
        --password-stdin

remote_image="$REGISTRY_HOST/pens-web:1.0"

docker tag "$APP_IMAGE" "$remote_image"
docker push "$remote_image"
docker pull "$remote_image"

curl --fail --silent --show-error \
    --cacert registry/certs/domain.crt \
    --user "$REGISTRY_USER:$REGISTRY_PASSWORD" \
    "https://$REGISTRY_HOST/v2/_catalog" \
    | tee reports/registry-catalog.json

curl --fail --silent --show-error \
    --cacert registry/certs/domain.crt \
    --user "$REGISTRY_USER:$REGISTRY_PASSWORD" \
    "https://$REGISTRY_HOST/v2/pens-web/tags/list" \
    | tee reports/registry-tags.json

docker image inspect "$remote_image" \
    --format '{{json .RepoDigests}}' \
    | tee reports/registry-digests.json

docker logout "$REGISTRY_HOST"
echo "[PASS] Push, pull, katalog, tag, dan digest telah diverifikasi."
```


Hasil:


![Dokumentasi script-11](assets/script-11-1.png)


![Dokumentasi script-11](assets/script-11-2.png)


Analisa: Isi `scripts/push-registry.sh` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Script 12 — File `scripts/verify-hardening.sh`

> **Nama file:** `verify-hardening.sh`  
> **Lokasi:** `~/docker-lab/bab-8/scripts/verify-hardening.sh`  
> **Buka editor:** `nano scripts/verify-hardening.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

pass_count=0

pass() {
    echo "[PASS] $1"
    pass_count=$((pass_count + 1))
}

fail() {
    echo "[FAIL] $1" >&2
    exit 1
}

uid="$(docker compose exec -T app id -u)"
[[ "$uid" == "10001" ]] \
    && pass "Proses aplikasi berjalan sebagai UID 10001." \
    || fail "UID aplikasi adalah $uid."

if docker compose exec -T app sh -c 'touch /unauthorized-write' 2>/dev/null; then
    fail "Filesystem root masih dapat ditulis."
else
    pass "Penulisan ke filesystem root ditolak."
fi

docker compose exec -T app sh -c 'touch /tmp/allowed-write && rm /tmp/allowed-write' \
    && pass "tmpfs /tmp dapat digunakan aplikasi." \
    || fail "tmpfs /tmp tidak dapat digunakan."

cap_eff="$(
    docker compose exec -T app sh -c \
        "awk '/CapEff/ {print \$2}' /proc/1/status"
)"
[[ "$cap_eff" == "0000000000000000" ]] \
    && pass "Seluruh Linux capability efektif telah dihapus." \
    || fail "Capability efektif masih tersedia: $cap_eff"

no_new_privs="$(
    docker compose exec -T app sh -c \
        "awk '/NoNewPrivs/ {print \$2}' /proc/1/status"
)"
[[ "$no_new_privs" == "1" ]] \
    && pass "NoNewPrivs aktif." \
    || fail "NoNewPrivs tidak aktif."

docker compose exec -T app test -r /run/secrets/app_secret \
    && pass "Secret tersedia sebagai file read-only." \
    || fail "Secret tidak dapat dibaca aplikasi."

if docker compose exec -T app env | grep -q '^APP_SECRET='; then
    fail "Secret ditemukan sebagai environment variable."
else
    pass "Secret tidak diekspos sebagai APP_SECRET."
fi

docker compose exec -T app test ! -S /var/run/docker.sock \
    && pass "Docker socket tidak tersedia di container aplikasi." \
    || fail "Docker socket terpasang ke container aplikasi."

container_id="$(docker compose ps -q app)"
read -r memory nano_cpus pids readonly_root <<<"$(
    docker inspect "$container_id" \
        --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}} {{.HostConfig.PidsLimit}} {{.HostConfig.ReadonlyRootfs}}'
)"

if (( memory > 0 && nano_cpus > 0 && pids > 0 )) \
    && [[ "$readonly_root" == "true" ]]; then
    pass "Limit memory, CPU, PID, dan read-only rootfs aktif."
else
    fail "Runtime limit tidak lengkap."
fi

app_ready="false"
for attempt in {1..30}; do
    if curl --fail --silent --show-error \
        http://127.0.0.1:8088/health >/dev/null 2>&1; then
        app_ready="true"
        break
    fi
    sleep 2
done

[[ "$app_ready" == "true" ]] \
    && pass "Health endpoint aplikasi merespons sukses." \
    || fail "Health endpoint aplikasi gagal."

unauth_code=""
for attempt in {1..30}; do
    unauth_code="$(
        curl --silent \
            --output /dev/null \
            --write-out '%{http_code}' \
            --cacert registry/certs/domain.crt \
            https://localhost:5000/v2/ \
            || true
    )"
    [[ "$unauth_code" == "401" ]] && break
    sleep 2
done
[[ "$unauth_code" == "401" ]] \
    && pass "Registry menolak akses anonim dengan HTTP 401." \
    || fail "Registry memberi status anonim $unauth_code."

echo "[PASS] Total kontrol terverifikasi: $pass_count"
```


Hasil:


![Dokumentasi script-12](assets/script-12-1.png)


![Dokumentasi script-12](assets/script-12-2.png)


![Dokumentasi script-12](assets/script-12-3.png)


Analisa: Isi `scripts/verify-hardening.sh` disimpan sesuai cheatsheet. Capture menampilkan berkas yang digunakan pada run ini; keberhasilan fungsinya diperiksa pada kotak perintah berikutnya.


## Kotak Perintah B — Izin Eksekusi dan Validasi Statis

```bash
chmod +x scripts/*.sh

bash -n scripts/generate-registry-assets.sh
bash -n scripts/scan-image.sh
bash -n scripts/push-registry.sh
bash -n scripts/verify-hardening.sh

python3 -m py_compile app/app.py
docker compose config --quiet
```

Validasi struktur Dockerfile dan Compose dengan Trivy dilakukan kembali setelah image dibangun karena database pemeriksaan Trivy dapat berubah.


Hasil:


![Dokumentasi B-1](assets/B-1-1.png)


Analisa: Pemeriksaan sintaks Bash, kompilasi Python, dan validasi Compose selesai dengan exit code 0.


## Kotak Perintah C — Membuat Secret, Sertifikat, dan Akun Registry

```bash
openssl rand -hex 32 > secrets/app_secret.txt
chmod 600 secrets/app_secret.txt
./scripts/generate-registry-assets.sh
```

Periksa SAN dan masa berlaku sertifikat:

```bash
openssl x509 \
  -in registry/certs/domain.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```


Hasil:


![Dokumentasi C-1](assets/C-1-1.png)


![Dokumentasi C-1](assets/C-1-2.png)


![Dokumentasi C-2](assets/C-2-1.png)


Analisa: Secret acak, sertifikat self-signed dengan SAN localhost, dan akun bcrypt registry berhasil dibuat. Nilai secret aktif tidak ditampilkan.


## Kotak Perintah D — Memercayai CA Registry pada Docker Engine

Pada Docker Engine native Linux:

```bash
sudo mkdir -p /etc/docker/certs.d/localhost:5000
sudo cp registry/certs/domain.crt \
  /etc/docker/certs.d/localhost:5000/ca.crt
sudo systemctl restart docker
```

Pada Docker Engine rootless Linux:

```bash
mkdir -p ~/.config/docker/certs.d/localhost:5000
cp registry/certs/domain.crt \
  ~/.config/docker/certs.d/localhost:5000/ca.crt
systemctl --user restart docker
```

Pada Docker Desktop dengan Linux containers:

```bash
mkdir -p ~/.docker/certs.d/localhost:5000
cp registry/certs/domain.crt \
  ~/.docker/certs.d/localhost:5000/ca.crt
```

Setelah penyalinan pada Docker Desktop, restart aplikasi Docker Desktop. Restart daemon menghentikan atau memengaruhi container lain; simpan pekerjaan lab aktif terlebih dahulu.


Hasil:


![Dokumentasi D-1](assets/D-1-1.png)


Analisa: Pemasangan CA ke trust store Docker native Linux terhalang karena sudo memerlukan password. Langkah rootless dan Docker Desktop tidak dijalankan karena host memakai Docker Engine native.


## Kotak Perintah E — Build dan Menjalankan Runtime yang Diperkeras

```bash
docker compose build --pull app
docker compose up -d app registry
docker compose ps
```

Verifikasi response aplikasi:

```bash
curl --fail --silent --show-error \
  http://127.0.0.1:8088/ \
  | python3 -m json.tool
```

Output harus menunjukkan `uid` dan `gid` bernilai `10001`, serta `secret_mounted` bernilai `true`.


Hasil:


![Dokumentasi E-1](assets/E-1-1.png)


![Dokumentasi E-1](assets/E-1-2.png)


![Dokumentasi E-1](assets/E-1-3.png)


![Dokumentasi E-1](assets/E-1-4.png)


![Dokumentasi E-1](assets/E-1-5.png)


![Dokumentasi E-2](assets/E-2-1.png)


![Dokumentasi E-recheck](assets/E-recheck-1.png)


![Dokumentasi diagnosis](assets/diagnosis-1.png)


Analisa: Build dan pembuatan container berhasil, tetapi aplikasi tidak dapat membaca /run/secrets/app_secret. Request ulang menghasilkan HTTP 500. Registry juga gagal membaca /auth/htpasswd. Status container dibuat belum membuktikan service sehat.


## Kotak Perintah F — Menjalankan Negative Test Hardening

```bash
./scripts/verify-hardening.sh
```

Periksa konfigurasi runtime efektif sebagai evidence tambahan:

```bash
docker inspect "$(docker compose ps -q app)" \
  --format '{{json .HostConfig}}' \
  > reports/app-hostconfig.json

docker image inspect pens-web:1.0 \
  --format '{{json .Config}}' \
  > reports/app-image-config.json
```


Hasil:


![Dokumentasi F-1](assets/F-1-1.png)


![Dokumentasi F-2](assets/F-2-1.png)


Analisa: Lima pemeriksaan awal lulus: UID 10001, penolakan tulis rootfs, tmpfs, capability nol, dan NoNewPrivs. Script berhenti pada pemeriksaan secret karena file tidak dapat dibaca aplikasi.


## Kotak Perintah G — Trivy, SBOM, dan Security Gate

```bash
./scripts/scan-image.sh
```

Periksa status gate dan checksum:

```bash
cat reports/gate-status.txt
sha256sum --check reports/SHA256SUMS
```

Jika gate berstatus `BLOCKED`, jangan mengubah status secara manual. Perbarui base image atau dependency, bangun ulang image, dokumentasikan triage, lalu ulangi pemindaian.


Hasil:


![Dokumentasi G-1](assets/G-1-1.png)


![Dokumentasi G-1](assets/G-1-2.png)


![Dokumentasi G-2](assets/G-2-1.png)


Analisa: Scanner berhenti saat mengunduh database sekitar 134 MB karena tmpfs /tmp hanya 64 MB. Ini kesalahan eksekusi scanner, bukan hasil CVE atau gate BLOCKED. gate-status.txt dan SHA256SUMS belum dibuat.


## Kotak Perintah H — Menguji dan Mengisi Private Registry

Sebelum login, buktikan bahwa endpoint anonim ditolak:

```bash
curl --silent \
  --output /dev/null \
  --write-out 'HTTP %{http_code}\n' \
  --cacert registry/certs/domain.crt \
  https://localhost:5000/v2/
```

Hasil yang diharapkan adalah `HTTP 401`. Setelah gate lulus:

```bash
./scripts/push-registry.sh
```

Registry API tetap memakai path `/v2/` karena Distribution Registry 3 mempertahankan kompatibilitas Docker Registry HTTP API V2.


Hasil:


![Dokumentasi H-1](assets/H-1-1.png)


![Dokumentasi H-2](assets/H-2-1.png)


Analisa: Request registry gagal tersambung, sehingga HTTP 401 belum terbukti. Script push menolak image karena tidak ada status gate PASS. Push dan pull belum berhasil pada run ini.


## Kotak Perintah I — Membuktikan Secret Tidak Masuk Image

Cari nama dan nilai contoh secret pada history serta filesystem image:

```bash
docker history --no-trunc pens-web:1.0 \
  | tee reports/docker-history.txt

docker run --rm \
  --entrypoint sh \
  pens-web:1.0 \
  -c 'test ! -e /run/secrets/app_secret && test ! -d /secrets'
```

PASS membuktikan file runtime secret tidak ada pada image. Hal ini belum membuktikan seluruh riwayat build bebas secret; laporan secret scanner dan audit build context tetap diperlukan.


Hasil:


![Output kotak-I bagian 1](assets/audit-kotak-I-1.png)

![Output kotak-I bagian 2](assets/audit-kotak-I-2.png)

![Output kotak-I bagian 3](assets/audit-kotak-I-3.png)

![Output kotak-I bagian 4](assets/audit-kotak-I-4.png)

Gambar menampilkan output `docker history` dan pemeriksaan filesystem image dalam beberapa panel agar teks terbaca. Output lengkap tersimpan di [log Kotak I](assets/kotak-I-output-lengkap.txt).


Analisa: History image tercatat dan pemeriksaan filesystem image selesai dengan exit code 0. File runtime secret dan direktori /secrets tidak ditemukan di image; hasil ini bukan jaminan seluruh riwayat build bebas secret.


## Kotak Perintah J — Cleanup

Menghentikan service tanpa menghapus registry:

```bash
docker compose --profile tools down --remove-orphans
```

Menghapus container, network, registry, dan cache Trivy:

```bash
docker compose --profile tools down --volumes --remove-orphans
```

Perintah kedua menghapus artefak pada volume registry. Laporan di direktori `reports/` tetap berada pada host, tetapi dapat memuat informasi keamanan dan harus dilindungi.

Apabila sertifikat lab tidak lagi digunakan, hapus CA dari trust store Docker sesuai platform dan restart Docker secara terencana.


Hasil:


![Dokumentasi J-1](assets/J-1-1.png)


![Dokumentasi J-2](assets/J-2-1.png)


Analisa: Kedua perintah cleanup selesai. Volume registry dan cache Trivy dihapus, sedangkan laporan host tetap disimpan. CA tidak perlu dihapus karena pemasangannya tidak berhasil.


## Troubleshooting Tambahan dan Pengujian Ulang

Pengujian awal tetap dicantumkan sebagai hasil sebelum perbaikan. Perubahan berikut diterapkan pada salinan praktikum untuk menyelesaikan kendala tanpa mengubah script verifikasi, script scanning, atau kebijakan gate.

### 1. Akses Secret dan Aset Registry

```bash
setfacl -m u:10001:r secrets/app_secret.txt
setfacl -m u:0:r registry/auth/htpasswd registry/certs/domain.key
chcon -t container_file_t secrets/app_secret.txt
```

Mount direktori sertifikat dan autentikasi registry diberi opsi `:ro,z`, dan mount workspace scanner juga diberi `:ro,z`. Opsi `z` memberi label SELinux yang dapat dipakai container, sementara akses file tetap read-only.

Hasil:

![Output koreksi izin dan label](assets/audit-trouble-01-corrections-1.png)

Analisa: pada bagian ini secret sebelumnya dimiliki UID host 1000 dengan mode 600, sedangkan aplikasi berjalan sebagai UID 10001. ACL memberi izin baca khusus kepada UID aplikasi. ACL untuk UID 0 pada aset registry dan label SELinux memperbaiki akses registry tanpa membuka file ke semua pengguna.

### 2. Port Host dan Ruang Sementara Scanner

Pada service `trivy`, kapasitas tmpfs `/tmp` diubah dari `64m` menjadi `512m`. Network bridge `host-access` ditambahkan ke aplikasi dan registry; network internal sebelumnya tetap dipertahankan. Port tetap terikat pada `127.0.0.1`.

Hasil:

![Perubahan compose](assets/audit-trouble-14-compose-diff-1.png)

![Status network dan port](assets/audit-trouble-04-network-1.png)

![Pengujian ulang hardening](assets/audit-trouble-05-hardening-retest-1.png)

Analisa: pada bagian ini Docker tidak memublikasikan port service yang hanya terhubung ke network internal. Setelah network bridge tambahan dipakai, aplikasi dapat diakses pada port 8088 dan registry pada port 5000. Seluruh 11 pemeriksaan hardening lulus, dan kapasitas temporary storage scanner cukup untuk mengunduh database.

### 3. Remediasi Temuan HIGH

```bash
docker compose build --pull app
./scripts/scan-image.sh
```

Dockerfile diperbarui untuk memasang pembaruan paket OS dan alat packaging Python. Temuan yang tersisa berasal dari komponen setuptools yang dibundel di pip. Pip kemudian dihapus dari image runtime karena Gunicorn tidak memerlukannya saat aplikasi berjalan.

Hasil:

![Hasil scan awal yang diblokir](assets/audit-trouble-06-scan-1.png)

![Perubahan Dockerfile bagian 1](assets/audit-trouble-13-dockerfile-diff-1.png)

![Perubahan Dockerfile bagian 2](assets/audit-trouble-13-dockerfile-diff-2.png)

![Hasil scan ulang](assets/audit-trouble-11-final-scan-1.png)

Analisa: pada bagian ini scan pertama setelah perbaikan temporary storage berhasil dijalankan, tetapi gate menolak temuan HIGH yang sudah memiliki versi perbaikan. Setelah paket diperbarui dan pip runtime dihapus, scan ulang menghasilkan gate PASS. Status gate tidak diubah secara manual dan tidak ada pengecualian CVE.

### 4. Verifikasi Akhir dan Distribusi Image

```bash
./scripts/verify-hardening.sh
cat reports/gate-status.txt
sha256sum --check reports/SHA256SUMS
./scripts/push-registry.sh
```

Hasil:

![Verifikasi akhir bagian 1](assets/audit-trouble-12-final-1.png)

![Verifikasi akhir bagian 2](assets/audit-trouble-12-final-2.png)

Analisa: pada bagian ini 11 kontrol hardening lulus, gate berstatus PASS, dan lima checksum laporan valid. Login, push, pull, katalog, tag, dan digest registry berhasil diverifikasi. Digest push dan pull sama, yaitu `sha256:a68c894c240571ec86d7f4b4222639185cbed8eb967581f1a268cef036806d84`.

Pemasangan CA sistem pada Kotak D tidak dijalankan karena membutuhkan sudo. Docker host menerima registry localhost pada pengujian ini, tetapi itu tidak berarti CA sistem sudah terpasang. Validasi sertifikat dilakukan melalui `curl --cacert`, dan akses anonim menghasilkan HTTP 401. Lingkungan produksi tetap memerlukan trust store dan sertifikat yang sesuai.

### 5. Cleanup Setelah Pengujian Selesai

```bash
docker compose --profile tools down --remove-orphans
docker compose --profile tools down --volumes --remove-orphans
```

Hasil:

![Cleanup bagian 1](assets/audit-trouble-15-cleanup-1.png)

![Cleanup bagian 2](assets/audit-trouble-15-cleanup-2.png)

Analisa: pada bagian ini container, image, volume registry, cache Trivy, dan network lab sudah dihapus. Hanya network bawaan Docker yang tersisa. File sumber, laporan scanner, SBOM, log, dan gambar dokumentasi tetap tersimpan di host.

## Rekap Hasil Setelah Troubleshooting

| Pengujian | Hasil akhir |
| --- | --- |
| UID non-root, read-only rootfs, tmpfs, capabilities, NoNewPrivs | PASS |
| Secret berupa file, tidak ada APP_SECRET, tidak ada Docker socket | PASS |
| CPU, memori, PID dan health endpoint | PASS |
| Registry TLS melalui curl --cacert dan penolakan anonim HTTP 401 | PASS |
| Scan image, konfigurasi, secret, SBOM, checksum | PASS |
| Security gate HIGH/CRITICAL yang dapat diperbaiki | PASS |
| Login, push, pull, katalog, tag, digest | PASS |
| File runtime secret tidak ada di image | PASS |
| Pemasangan CA pada trust store sistem | Tidak dijalankan; sudo diperlukan |
| Cleanup Docker lab | Selesai |

## Evaluasi dan Latihan Mandiri

**1. Mengapa root di container tetap berbahaya meskipun bukan selalu root host?**

Container berbagi kernel host. Proses root memiliki hak luas di dalam container, dan celah kernel, capability berlebih, atau mount host dapat memperbesar dampak eksploitasi. User non-root tetap perlu dilengkapi pembatasan capability dan filesystem.

**2. Apa kelemahan menyimpan secret pada environment variable?**

Secret dapat terlihat pada konfigurasi container, dump proses, atau log diagnostik. File secret dengan izin baca minimum mengurangi paparan, tetapi tetap harus dapat dibaca oleh UID aplikasi yang benar.

**3. Bagaimana menentukan apakah CVE harus segera diperbaiki atau dapat ditoleransi sementara?**

Pertimbangkan tingkat keparahan, ketersediaan exploit, akses ke komponen rentan, dampak terhadap layanan, dan ketersediaan perbaikan. Penerimaan sementara harus mencatat alasan, mitigasi, penanggung jawab, persetujuan, dan batas waktu.

**4. Apa perbedaan registry private biasa dan registry dengan policy scanning gate?**

Registry privat membatasi akses distribusi image. Scanning gate menambahkan keputusan menerima atau menolak image berdasarkan kebijakan temuan. Pada praktikum ini script push hanya mengizinkan image ketika gate berstatus PASS.

**5. Mengapa Docker socket setara dengan akses administratif ke Docker host?**

Akses socket memungkinkan pengendalian daemon, termasuk membuat container privileged atau memasang filesystem host. Pada Docker Engine rootful, kemampuan tersebut dapat digunakan untuk memperoleh kendali administratif atas host.

## Referensi

1. Ferry Astika Saputra, "Bab 8 — Docker Security: Hardening, Secrets, Trivy, dan Private Registry," repository DevSecOps PENS, `bab-08-cs.md`.
2. NIST SP 800-190, *Application Container Security Guide*.
3. OWASP, *Docker Security Cheat Sheet*.
4. Aqua Security, *Trivy documentation*: https://trivy.dev
5. Docker Documentation, *Docker Registry*: https://docs.docker.com/registry/
6. Docker Documentation, *Docker Bench for Security*: https://github.com/docker/docker-bench-security

## Kesimpulan

Praktikum berhasil diselesaikan setelah troubleshooting izin file, SELinux, publikasi port, kapasitas tmpfs scanner, dan pembaruan paket rentan. Sebelas kontrol hardening lulus, gate berstatus PASS, laporan beserta SBOM memiliki checksum valid, dan distribusi image berhasil dengan digest yang sama. Script verifikasi, scanning, dan push tetap mengikuti cheatsheet. Pemasangan CA sistem tidak dilakukan; hasil penerimaan registry localhost oleh Docker dicatat terpisah dari validasi sertifikat menggunakan curl. Seluruh resource Docker lab telah dibersihkan, sedangkan bukti pengujian tetap disimpan pada host.
