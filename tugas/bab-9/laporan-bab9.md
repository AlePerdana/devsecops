# LAPORAN PRAKTIKUM BAB 9
## Identity and Access Management dengan Keycloak di Docker

**Nama**: Ale Perdana Putra Darmawan  
**NIM**: 3126640016  
**Kelas**: STr LJ Teknik Informatika A  
**Tanggal pelaksanaan**: 17 November 2026

> **Disclaimer penggunaan AI.** Laporan praktikum ini disusun dengan bantuan kecerdasan buatan (AI) yang difungsikan sebagai alat dokumentasi. AI digunakan untuk merapikan catatan praktikum, menyusun struktur dan alur penulisan laporan, serta menyunting tata bahasa. Seluruh pelaksanaan praktikum, pengambilan bukti berupa screenshot, verifikasi keluaran perintah, dan pengambilan kesimpulan tetap dilakukan secara mandiri oleh saya.

**Sumber:** `cheatsheets/bab-09-cs.md`. Isi kotak script dan perintah dipertahankan dari cheatsheet.

## Hasil Belajar

Setelah menyelesaikan praktikum, mahasiswa mampu:

1. menjelaskan perbedaan autentikasi, otorisasi, OAuth 2.0, dan OpenID Connect;
2. menjalankan Keycloak 26.7.2 dengan PostgreSQL 17.11 dalam Docker Compose;
3. mengelola realm, confidential client, pengguna, realm role, dan scope mapping sebagai kode;
4. menerapkan Authorization Code Flow dengan PKCE S256 pada aplikasi Flask;
5. membuktikan penerapan RBAC melalui pengujian positif dan negatif;
6. mengumpulkan evidence yang tidak membocorkan password, client secret, atau token utuh.

## Arsitektur Ringkas

```mermaid
flowchart TD
    U["Browser pengguna"] -->|"HTTP loopback"| A["Flask relying party"]
    U -->|"Login dan consent"| K["Keycloak identity provider"]
    A -->|"Code ditukar dengan token"| K
    K -->|"JDBC pada jaringan internal"| D["PostgreSQL"]
    A -->|"Role viewer/admin"| P["Endpoint terlindungi"]
```

Trust boundary laboratorium terdiri atas browser, relying party, identity provider, dan database. Browser hanya mengakses port `5000` dan `8080` melalui `127.0.0.1`; port manajemen `9000` juga dibatasi ke loopback untuk pengujian readiness. PostgreSQL tidak dipublikasikan ke host. Nama `keycloak.localhost` digunakan sebagai issuer yang konsisten: browser memperlakukannya sebagai nama loopback, sedangkan Docker DNS memetakannya ke service Keycloak melalui network alias.

| Komponen | Alamat | Fungsi keamanan |
| --- | --- | --- |
| Flask | `http://localhost:5000` | OIDC relying party dan enforcement RBAC |
| Keycloak | `http://keycloak.localhost:8080` | Autentikasi, sesi SSO, penerbit token, dan role |
| Management | `http://localhost:9000/health/ready` | Readiness lokal; jangan dipublikasikan pada lingkungan produksi |
| PostgreSQL | `db:5432` pada `iam-net` | Penyimpanan state Keycloak tanpa port host |
| Realm | `devsecops-lab` | Batas administrasi identitas laboratorium |

## Versi yang Digunakan

Versi berikut dipin agar hasil praktikum dapat direlingkungan produksi. Versi ini diverifikasi pada 22 Agustus 2026; lakukan verifikasi ulang sebelum semester berikutnya.

| Komponen | Versi/tag | Catatan |
| --- | --- | --- |
| Keycloak | `26.7.2` | Rilis server yang ditampilkan pada halaman unduhan resmi saat verifikasi |
| PostgreSQL | `17.11-alpine3.24` | Tag image resmi PostgreSQL |
| Python | `3.11.16-slim` | Base image aplikasi |
| Flask | `3.1.3` | Framework aplikasi |
| Authlib | `1.7.2` | OIDC/OAuth client dan validasi ID Token |
| Gunicorn | `26.1.0` | WSGI server aplikasi |

## Kotak Perintah A — Menyiapkan Direktori Bab 9

Jalankan perintah berikut dari terminal Bash, Git Bash, WSL, atau terminal Linux. PowerShell juga dapat digunakan untuk perintah Docker, tetapi script `.sh` pada bab ini lebih mudah dijalankan melalui Git Bash atau WSL.

```bash
$ mkdir -p ~/docker-lab/bab-09/{app,realm,scripts,reports}
$ cd ~/docker-lab/bab-09
$ pwd
```

Struktur akhir yang diharapkan:

```text
bab-09/
├── .dockerignore
├── .env.example
├── .gitignore
├── compose.yaml
├── app/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── realm/
│   └── devsecops-lab-realm.json
├── reports/
└── scripts/
    ├── collect-evidence.sh
    ├── generate-env.sh
    ├── negative-tests.sh
    └── validate-lab.sh
```

Hasil:

![Kotak Perintah A (1.png)](assets/1.png)

Analisa: screenshot `1.png` menampilkan `mkdir -p ~/docker-lab/bab-09/{app,realm,scripts,reports}` dan perintah `cd` ke direktori lab. Struktur direktori ini menjadi tempat semua file Bab 9 dibuat.

## Kotak Script 1 — File `.env.example`

File contoh ini mendokumentasikan variabel yang diperlukan. Nilainya bukan kredensial operasional dan tidak digunakan langsung oleh Compose.

```dotenv
POSTGRES_DB=keycloak
POSTGRES_USER=keycloak
POSTGRES_PASSWORD=GANTI_DENGAN_NILAI_ACAK

KC_BOOTSTRAP_ADMIN_USERNAME=kcadmin
KC_BOOTSTRAP_ADMIN_PASSWORD=GANTI_DENGAN_NILAI_ACAK

FLASK_CLIENT_ID=flask-app
FLASK_CLIENT_SECRET=GANTI_DENGAN_NILAI_ACAK
FLASK_SESSION_SECRET=GANTI_DENGAN_NILAI_ACAK

MAHASISWA_PASSWORD=GANTI_DENGAN_NILAI_ACAK
ADMINLAB_PASSWORD=GANTI_DENGAN_NILAI_ACAK
```

Hasil:

![Kotak Script 1 (2.png)](assets/2.png)

Analisa: screenshot `2.png` menampilkan `.env.example` yang dibuka dengan nano dan dibaca dengan cat. File `.env` yang sebenarnya dibuat oleh `generate-env.sh` pada Kotak Script 9.

## Kotak Script 2 — File `.gitignore`

```gitignore
.env
reports/*
!reports/.gitkeep
__pycache__/
*.py[cod]
.pytest_cache/
```

Buat placeholder agar direktori laporan tetap dapat dilacak tanpa memasukkan evidence sensitif.

```bash
$ touch reports/.gitkeep
```

Hasil:

![Kotak Script 2 (3.png)](assets/3.png)

Analisa: screenshot `3.png` menampilkan `.gitignore` yang memuat `.env`. Dengan ini file berisi kredensial tidak ikut di-commit.

## Kotak Script 3 — File `.dockerignore`

```dockerignore
.env
.git
.gitignore
reports
realm
scripts
__pycache__
*.pyc
*.pyo
```

Hasil:

![Kotak Script 3 (3ex.png)](assets/3ex.png)

Analisa: screenshot `3ex.png` menampilkan `cat .dockerignore` dengan isi `.env`, `.git`, `.gitignore`, `reports`, `realm`, `scripts`, `__pycache__`, `*.pyc`, dan `*.pyo`. Daftar ini mencegah file rahasia, riwayat git, dan artefak lokal ikut masuk ke build context image.

## Kotak Script 4 — File `realm/devsecops-lab-realm.json`

Realm diimpor sebagai kode. Placeholder `${...}` diselesaikan Keycloak dari environment container pada saat import. Client hanya mengaktifkan Standard Flow, mewajibkan PKCE S256, menolak Direct Access Grant, dan menggunakan redirect URI eksak.

```json
{
  "realm": "devsecops-lab",
  "displayName": "Laboratorium DevSecOps",
  "enabled": true,
  "registrationAllowed": false,
  "resetPasswordAllowed": false,
  "rememberMe": false,
  "verifyEmail": false,
  "bruteForceProtected": true,
  "accessTokenLifespan": 300,
  "ssoSessionIdleTimeout": 900,
  "ssoSessionMaxLifespan": 28800,
  "roles": {
    "realm": [
      {
        "name": "viewer",
        "description": "Membaca resource yang telah diautentikasi"
      },
      {
        "name": "admin",
        "description": "Mengakses endpoint administrasi aplikasi lab"
      }
    ]
  },
  "clients": [
    {
      "clientId": "${FLASK_CLIENT_ID}",
      "name": "Flask DevSecOps Lab",
      "enabled": true,
      "protocol": "openid-connect",
      "publicClient": false,
      "secret": "${FLASK_CLIENT_SECRET}",
      "clientAuthenticatorType": "client-secret",
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "fullScopeAllowed": false,
      "rootUrl": "http://localhost:5000",
      "baseUrl": "http://localhost:5000/",
      "redirectUris": [
        "http://localhost:5000/callback"
      ],
      "webOrigins": [
        "http://localhost:5000"
      ],
      "attributes": {
        "pkce.code.challenge.method": "S256",
        "post.logout.redirect.uris": "http://localhost:5000/logged-out"
      },
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "roles",
        "profile",
        "basic",
        "email"
      ]
    }
  ],
  "scopeMappings": [
    {
      "client": "${FLASK_CLIENT_ID}",
      "roles": [
        "viewer",
        "admin"
      ]
    }
  ],
  "users": [
    {
      "username": "mahasiswa1",
      "enabled": true,
      "emailVerified": true,
      "firstName": "Mahasiswa",
      "lastName": "Viewer",
      "email": "mahasiswa1@example.invalid",
      "realmRoles": [
        "viewer"
      ],
      "credentials": [
        {
          "type": "password",
          "value": "${MAHASISWA_PASSWORD}",
          "temporary": false
        }
      ]
    },
    {
      "username": "adminlab",
      "enabled": true,
      "emailVerified": true,
      "firstName": "Administrator",
      "lastName": "Laboratorium",
      "email": "adminlab@example.invalid",
      "realmRoles": [
        "viewer",
        "admin"
      ],
      "credentials": [
        {
          "type": "password",
          "value": "${ADMINLAB_PASSWORD}",
          "temporary": false
        }
      ]
    }
  ]
}
```

> Akun `adminlab` hanya memiliki role `admin` pada aplikasi laboratorium. Akun ini berbeda dari administrator server Keycloak `kcadmin`. Pemisahan tersebut mencegah kekeliruan antara kewenangan aplikasi dan kewenangan administrasi identity provider.

Hasil:

![Kotak Script 4 (4.png)](assets/4.png)

Analisa: file realm berisi definisi realm, client `flask-app` dengan PKCE `S256`, dan dua pengguna. Pada percobaan ini ada dua perubahan dari file JSON. Pertama, label SELinux pada `realm/devsecops-lab-realm.json` diubah dari `user_home_t` menjadi `container_file_t` dengan `chcon`, karena Keycloak di dalam container ditolak saat membaca file tersebut (`Permission denied`). Kedua, mapper `realm roles` dan atribut PKCE pada client berjalan diubah langsung di Keycloak, lalu dicatat di bagian Troubleshooting. Perubahan kedua belum ada di file JSON.

## Kotak Script 5 — File `app/requirements.txt`

```text
Flask==3.1.3
Authlib==1.7.2
gunicorn==26.1.0
```

Hasil:

![Kotak Script 5 (5.png)](assets/5.png)

Analisa: isi `requirements.txt` dari cheatsheet memuat Flask, Authlib, dan gunicorn. Container `app` gagal dengan `ModuleNotFoundError: No module named 'requests'`, karena `app.py` mengimpor `authlib.integrations.flask_client` yang membutuhkan paket `requests`. Paket ini tidak ikut terpasang oleh Authlib. Perubahan yang dilakukan: menambahkan baris `requests==2.32.5` ke `app/requirements.txt`, dan menyimpan file asli sebagai `requirements.txt.orig`. Setelah perubahan, container `app` berstatus healthy dan `/healthz` dari dalam container mengembalikan HTTP 200.

## Kotak Script 6 — File `app/app.py`

Aplikasi mempercayai metadata OIDC, signature, issuer, audience, expiration, `state`, dan `nonce` melalui integrasi Authlib. Aplikasi hanya menyimpan profil minimum dan role dalam cookie sesi; access token, refresh token, dan ID Token tidak ditampilkan atau ditulis ke log.

```python
import os
from functools import wraps
from authlib.integrations.base_client.errors import OAuthError
from authlib.integrations.flask_client import OAuth
from flask import Flask, jsonify, redirect, render_template_string, session, url_for

ISSUER = os.getenv(
    "OIDC_ISSUER",
    "http://keycloak.localhost:8080/realms/devsecops-lab",
).rstrip("/")
CLIENT_ID = os.environ["OIDC_CLIENT_ID"]

app = Flask(__name__)
app.secret_key = os.environ["FLASK_SESSION_SECRET"]
app.config.update(
    SESSION_COOKIE_NAME="devsecops_lab_session",
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE="Lax",
    SESSION_COOKIE_SECURE=False,
    PERMANENT_SESSION_LIFETIME=900,
)

oauth = OAuth(app)
oauth.register(
    name="keycloak",
    client_id=CLIENT_ID,
    client_secret=os.environ["OIDC_CLIENT_SECRET"],
    server_metadata_url=f"{ISSUER}/.well-known/openid-configuration",
    client_kwargs={
        "scope": "openid profile email roles",
        "code_challenge_method": "S256",
        "token_endpoint_auth_method": "client_secret_basic",
    },
)

def current_user():
    return session.get("user")

def require_role(required_role):
    def decorator(view):
        @wraps(view)
        def wrapped(*args, **kwargs):
            user = current_user()
            if not user:
                return redirect(url_for("login"))
            if required_role not in user.get("roles", []):
                return jsonify(
                    error="forbidden",
                    required_role=required_role,
                ), 403
            return view(*args, **kwargs)

        return wrapped

    return decorator

@app.get("/")
def home():
    return render_template_string(
        """
        <!doctype html>
        <html lang="id">
        <head>
          <meta charset="utf-8">
          <title>DevSecOps IAM Lab</title>
          <style>
            body { font: 16px system-ui; max-width: 820px; margin: 3rem auto; }
            nav a { margin-right: 1rem; }
            code, pre { background: #f3f4f6; padding: .2rem .4rem; }
            pre { padding: 1rem; overflow: auto; }
          </style>
        </head>
        <body>
          <h1>Laboratorium IAM dan RBAC</h1>
          <nav>
            <a href="{{ url_for('login') }}">Login</a>
            <a href="{{ url_for('viewer') }}">Viewer</a>
            <a href="{{ url_for('admin') }}">Admin</a>
            <a href="{{ url_for('local_logout') }}">Logout lokal</a>
            <a href="{{ url_for('sso_logout') }}">Logout SSO</a>
          </nav>
          {% if user %}
            <h2>Identitas tervalidasi</h2>
            <pre>{{ user | tojson(indent=2) }}</pre>
          {% else %}
            <p>Belum terdapat sesi lokal. Gunakan tautan <strong>Login</strong>.</p>
          {% endif %}
        </body>
        </html>
        """,
        user=current_user(),
    )

@app.get("/login")
def login():
    redirect_uri = url_for("callback", _external=True)
    return oauth.keycloak.authorize_redirect(redirect_uri)

@app.get("/callback")
def callback():
    token = oauth.keycloak.authorize_access_token()
    claims = token.get("userinfo", {})
    roles = claims.get("realm_access", {}).get("roles", [])
    session.clear()
    session["user"] = {
        "sub": claims.get("sub"),
        "preferred_username": claims.get("preferred_username"),
        "email": claims.get("email"),
        "roles": sorted(set(roles)),
    }
    return redirect(url_for("home"))

@app.get("/viewer")
@require_role("viewer")
def viewer():
    return jsonify(
        result="PASS",
        message="Role viewer diterima.",
        user=current_user()["preferred_username"],
    )

@app.get("/admin")
@require_role("admin")
def admin():
    return jsonify(
        result="PASS",
        message="Role admin diterima.",
        user=current_user()["preferred_username"],
    )

@app.get("/logout")
def local_logout():
    session.clear()
    return redirect(url_for("home"))

@app.get("/logout-sso")
def sso_logout():
    session.clear()
    return oauth.keycloak.logout_redirect(
        post_logout_redirect_uri=url_for("logged_out", _external=True),
        client_id=CLIENT_ID,
    )

@app.get("/logged-out")
def logged_out():
    oauth.keycloak.validate_logout_response()
    session.clear()
    return redirect(url_for("home"))

@app.get("/healthz")
def healthz():
    return jsonify(status="UP")

@app.errorhandler(OAuthError)
def oauth_error(error):
    app.logger.warning("OIDC request rejected: %s", error.error)
    return jsonify(error="oidc_request_rejected"), 400
```

`SESSION_COOKIE_SECURE=False` digunakan karena lab berjalan melalui HTTP loopback. Pada lingkungan HTTPS, nilai ini wajib diubah menjadi `True`. Jangan menambahkan token ke objek `session` karena Flask menyimpan session default di cookie yang berada pada browser.

Hasil:

![Kotak Script 6 (6.png)](assets/6.png)

Analisa: screenshot `6.png` menampilkan bagian awal `app.py`, yaitu import `os`, Authlib, dan Flask. Perilaku login, callback, dan pemeriksaan role diuji pada Kotak E dan F.

## Kotak Script 7 — File `app/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11.16-slim AS builder

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1

WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:${PATH}"

COPY requirements.txt .
RUN pip install --no-cache-dir --requirement requirements.txt

FROM python:3.11.16-slim AS runtime

ENV PATH="/opt/venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN groupadd --gid 10001 appgroup \
    && useradd --uid 10001 --gid appgroup --no-create-home --shell /usr/sbin/nologin appuser

WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY --chown=10001:10001 app.py .

USER 10001:10001
EXPOSE 5000

HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=5 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/healthz', timeout=2)"]

CMD ["gunicorn", "--bind=0.0.0.0:5000", "--workers=2", "--threads=2", "--timeout=30", "app:app"]
```

Hasil:

![Kotak Script 7 (7.png)](assets/7.png)

Analisa: screenshot `7.png` menampilkan `Dockerfile` dua tahap. Tahap builder membuat venv dan memasang `requirements.txt`. Tahap runtime berjalan sebagai user `10001` dan memakai `HEALTHCHECK` ke `127.0.0.1:5000/healthz` di dalam container. Healthcheck ini menjelaskan mengapa container berstatus healthy, padahal port host saat itu belum bisa diakses (lihat T5).

## Kotak Script 8 — File `compose.yaml`

```yaml
name: devsecops-bab09

services:
  db:
    image: postgres:17.11-alpine3.24
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:?POSTGRES_DB wajib diisi}
      POSTGRES_USER: ${POSTGRES_USER:?POSTGRES_USER wajib diisi}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD wajib diisi}
    volumes:
      - keycloak-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks:
      - iam-net
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    pids_limit: 200
    mem_limit: 512m

  keycloak:
    image: quay.io/keycloak/keycloak:26.7.2
    restart: unless-stopped
    command:
      - start-dev
      - --import-realm
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://db:5432/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KC_BOOTSTRAP_ADMIN_USERNAME}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KC_BOOTSTRAP_ADMIN_PASSWORD}
      KC_HOSTNAME: http://keycloak.localhost:8080
      KC_HEALTH_ENABLED: "true"
      KC_METRICS_ENABLED: "true"
      FLASK_CLIENT_ID: ${FLASK_CLIENT_ID}
      FLASK_CLIENT_SECRET: ${FLASK_CLIENT_SECRET}
      MAHASISWA_PASSWORD: ${MAHASISWA_PASSWORD}
      ADMINLAB_PASSWORD: ${ADMINLAB_PASSWORD}
    volumes:
      - ./realm/devsecops-lab-realm.json:/opt/keycloak/data/import/devsecops-lab-realm.json:ro
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:9000:9000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      iam-net:
        aliases:
          - keycloak.localhost
    security_opt:
      - no-new-privileges:true
    pids_limit: 400
    mem_limit: 1024m

  app:
    build:
      context: ./app
    image: devsecops-bab09-flask:1.0
    restart: unless-stopped
    environment:
      FLASK_SESSION_SECRET: ${FLASK_SESSION_SECRET}
      OIDC_CLIENT_ID: ${FLASK_CLIENT_ID}
      OIDC_CLIENT_SECRET: ${FLASK_CLIENT_SECRET}
      OIDC_ISSUER: http://keycloak.localhost:8080/realms/devsecops-lab
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      - keycloak
    networks:
      - iam-net
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=32m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    pids_limit: 100
    mem_limit: 256m

networks:
  iam-net:
    internal: true

volumes:
  keycloak-db-data:
```

**Perubahan pada praktikum:** baris `internal: true` pada network `iam-net` dikomentari. Alasannya dijelaskan pada T5 di bagian Troubleshooting. Isi blok di atas adalah salinan dari cheatsheet sebelum perubahan ini.

`cap_add` pada PostgreSQL merupakan pengecualian terukur karena entrypoint image perlu menyiapkan ownership direktori data dan berpindah ke user database. Keycloak tidak diberi `cap_drop: ALL` dalam lab ini karena kompatibilitas runtime harus diuji per versi. Aplikasi Flask, yang merupakan kode buatan praktikan, dijalankan non-root dengan seluruh capability dihapus dan root filesystem read-only.

Hasil:

![Kotak Script 8 (8.png)](assets/8.png)

Analisa: screenshot `8.png` memperlihatkan awal `compose.yaml` (nama project dan perintah `cat`). Isi lengkap service `db`, `keycloak`, dan `app` berasal dari cheatsheet. Baris `internal: true` pada `iam-net` dikomentari, seperti dicatat di T5.

## Kotak Script 9 — File `scripts/generate-env.sh`

Script membuat `.env` dengan mode file ketat dan nilai acak berbentuk heksadesimal. Script menolak menimpa file yang sudah ada agar rotasi tidak terjadi tanpa sengaja ketika realm telah tersimpan pada database.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/.."
umask 077

if [[ -e .env ]]; then
  echo "ERROR: .env sudah ada; file tidak ditimpa." >&2
  exit 1
fi

command -v openssl >/dev/null 2>&1 || {
  echo "ERROR: openssl tidak ditemukan." >&2
  exit 1
}

random_hex() {
  openssl rand -hex 24
}

cat > .env <<EOF
POSTGRES_DB=keycloak
POSTGRES_USER=keycloak
POSTGRES_PASSWORD=$(random_hex)

KC_BOOTSTRAP_ADMIN_USERNAME=kcadmin
KC_BOOTSTRAP_ADMIN_PASSWORD=$(random_hex)

FLASK_CLIENT_ID=flask-app
FLASK_CLIENT_SECRET=$(random_hex)
FLASK_SESSION_SECRET=$(random_hex)

MAHASISWA_PASSWORD=$(random_hex)
ADMINLAB_PASSWORD=$(random_hex)
EOF

chmod 600 .env 2>/dev/null || true
echo "PASS: .env dibuat. Simpan kredensial di password manager; jangan commit file ini."
```

Hasil:

![Kotak Script 9 (9.png)](assets/9.png)

Analisa: screenshot `9.png` menampilkan `generate-env.sh`. Output pada `13.png` menyatakan `PASS: .env dibuat`, sehingga script ini berjalan sesuai tujuan.

## Kotak Script 10 — File `scripts/validate-lab.sh`

Validasi statis dilakukan sebelum image dibangun. Script memeriksa JSON, kontrol client OIDC, role, placeholder secret, redirect URI eksak, Dockerfile non-root, dan sintaks Compose.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/.."

python3 - <<'PY'
import json
from pathlib import Path

realm_path = Path("realm/devsecops-lab-realm.json")
realm = json.loads(realm_path.read_text(encoding="utf-8"))

assert realm["realm"] == "devsecops-lab"
assert realm["bruteForceProtected"] is True

clients = realm["clients"]
assert len(clients) == 1
client = clients[0]
assert client["publicClient"] is False
assert client["standardFlowEnabled"] is True
assert client["implicitFlowEnabled"] is False
assert client["directAccessGrantsEnabled"] is False
assert client["serviceAccountsEnabled"] is False
assert client["fullScopeAllowed"] is False
assert client["redirectUris"] == ["http://localhost:5000/callback"]
assert all("*" not in uri for uri in client["redirectUris"])
assert client["attributes"]["pkce.code.challenge.method"] == "S256"
assert client["secret"] == "${FLASK_CLIENT_SECRET}"

roles = {item["name"] for item in realm["roles"]["realm"]}
assert roles == {"viewer", "admin"}
users = {item["username"]: set(item["realmRoles"]) for item in realm["users"]}
assert users["mahasiswa1"] == {"viewer"}
assert users["adminlab"] == {"viewer", "admin"}

print("PASS: JSON realm dan kebijakan OIDC/RBAC valid.")
PY

python3 -m py_compile app/app.py

grep -q '^USER 10001:10001$' app/Dockerfile
grep -q 'read_only: true' compose.yaml
grep -q 'cap_drop:' compose.yaml

if [[ -f .env ]]; then
  docker compose config --quiet
else
  echo "INFO: .env belum ada; jalankan scripts/generate-env.sh sebelum validasi Compose."
fi

echo "PASS: validasi statis selesai."
```

Hasil:

![Kotak Script 10 (10.png)](assets/10.png)

Analisa: screenshot `10.png` menampilkan `validate-lab.sh`. Output pada `14.png` menyatakan JSON realm dan kebijakan OIDC/RBAC valid, serta validasi statis selesai tanpa FAIL.

## Kotak Script 11 — File `scripts/negative-tests.sh`

Pengujian negatif membuktikan bahwa kontrol menolak akses tanpa sesi, redirect URI tak terdaftar, permintaan Authorization Code tanpa PKCE, dan Resource Owner Password Grant. Password pengguna nyata tidak digunakan pada pengujian Direct Grant.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/.."
mkdir -p reports

set -a
# shellcheck disable=SC1091
source .env
set +a

failures=0

expect_status() {
  local name="$1"
  local expected="$2"
  local url="$3"
  local actual
  actual="$(curl --silent --output /dev/null --write-out '%{http_code}' "$url")"
  if [[ "$actual" == "$expected" ]]; then
    printf 'PASS: %s menghasilkan HTTP %s\n' "$name" "$actual"
  else
    printf 'FAIL: %s menghasilkan HTTP %s; diharapkan %s\n' "$name" "$actual" "$expected" >&2
    failures=$((failures + 1))
  fi
}

expect_status "viewer tanpa sesi" "302" "http://localhost:5000/viewer"
expect_status "admin tanpa sesi" "302" "http://localhost:5000/admin"

bad_redirect_status="$(curl --silent --output reports/invalid-redirect.html \
  --write-out '%{http_code}' \
  --get 'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/auth' \
  --data-urlencode "client_id=${FLASK_CLIENT_ID}" \
  --data-urlencode 'response_type=code' \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'redirect_uri=http://attacker.invalid/callback' \
  --data-urlencode 'code_challenge=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA' \
  --data-urlencode 'code_challenge_method=S256')"

if [[ "$bad_redirect_status" == "400" ]]; then
  echo "PASS: redirect URI tidak terdaftar ditolak dengan HTTP 400."
else
  echo "FAIL: redirect URI tidak sah menghasilkan HTTP ${bad_redirect_status}." >&2
  failures=$((failures + 1))
fi

missing_pkce_status="$(curl --silent --output reports/missing-pkce.html \
  --write-out '%{http_code}' \
  --get 'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/auth' \
  --data-urlencode "client_id=${FLASK_CLIENT_ID}" \
  --data-urlencode 'response_type=code' \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'redirect_uri=http://localhost:5000/callback')"

if [[ "$missing_pkce_status" == "400" ]]; then
  echo "PASS: permintaan tanpa PKCE ditolak dengan HTTP 400."
else
  echo "FAIL: permintaan tanpa PKCE menghasilkan HTTP ${missing_pkce_status}." >&2
  failures=$((failures + 1))
fi

direct_grant_status="$(curl --silent --output reports/direct-grant.json \
  --write-out '%{http_code}' \
  --request POST \
  --user "${FLASK_CLIENT_ID}:${FLASK_CLIENT_SECRET}" \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=nonexistent-user' \
  --data-urlencode 'password=not-a-real-password' \
  'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/token')"

if [[ "$direct_grant_status" == "400" || "$direct_grant_status" == "401" ]]; then
  echo "PASS: Direct Access Grant dinonaktifkan dan permintaan ditolak."
else
  echo "FAIL: Direct Access Grant menghasilkan HTTP ${direct_grant_status}." >&2
  failures=$((failures + 1))
fi

printf 'Jumlah kegagalan: %d\n' "$failures"
exit "$failures"
```

> Perintah `curl --user` dapat membuat client secret terlihat sesaat pada daftar proses sistem. Jalankan hanya pada laptop laboratorium pengguna tunggal. Pada pipeline bersama, gunakan file konfigurasi sementara dengan permission ketat atau secret-aware test runner.

Hasil:

![Kotak Script 11 (11.png)](assets/11.png)

Analisa: screenshot `11.png` menampilkan `negative-tests.sh`. Versi yang dijalankan memiliki pemeriksaan PKCE yang diubah. Perubahan dan hasilnya dicatat pada Kotak Perintah F.

## Kotak Script 12 — File `scripts/collect-evidence.sh`

Script mengumpulkan evidence teknis tanpa mengekspor `.env`, cookie browser, atau token. Tetap tinjau seluruh berkas sebelum membagikannya.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/.."
mkdir -p reports

docker compose ps > reports/compose-ps.txt
docker compose images > reports/compose-images.txt
docker compose config --no-interpolate > reports/compose-no-interpolate.yaml
docker compose logs --no-color --tail 200 > reports/service-logs.txt

set -a
# shellcheck disable=SC1091
source .env
set +a

curl --fail --silent --show-error \
  http://localhost:9000/health/ready \
  > reports/keycloak-readiness.json

curl --fail --silent --show-error \
  http://keycloak.localhost:8080/realms/devsecops-lab/.well-known/openid-configuration \
  > reports/openid-configuration.json

curl --fail --silent --show-error \
  http://localhost:5000/healthz \
  > reports/app-health.json

sha256sum \
  compose.yaml \
  realm/devsecops-lab-realm.json \
  app/Dockerfile \
  app/app.py \
  app/requirements.txt \
  > reports/source-sha256.txt

if grep -R --fixed-strings --quiet "${FLASK_CLIENT_SECRET:-VALUE_NOT_SET}" reports; then
  echo "FAIL: client secret terdeteksi dalam reports." >&2
  exit 1
fi

echo "PASS: evidence tersimpan pada reports/. Tinjau ulang sebelum dibagikan."
```

Hasil:

![Kotak Script 12 (12.png)](assets/12.png)

Analisa: screenshot `12.png` menampilkan `collect-evidence.sh`. Output pada `19.png` menyatakan evidence tersimpan di `reports/`, dan daftar file tersebut tampak pada screenshot yang sama.

## Kotak Perintah B — Memberikan Izin Eksekusi dan Membuat `.env`

```bash
$ chmod +x scripts/*.sh
$ ./scripts/generate-env.sh
$ ls -l .env
$ sed -n 's/^\([^=]*\)=.*/\1=<REDACTED>/p' .env
```

Simpan password `MAHASISWA_PASSWORD` dan `ADMINLAB_PASSWORD` ke password manager lokal. Jangan menyalinnya ke laporan. Perintah terakhir hanya menampilkan nama variabel dan harus mempertahankan nilai sebagai `<REDACTED>`.

Hasil:

![Kotak Perintah B (13.png)](assets/13.png)

Analisa: script `generate-env.sh` berjalan dan menulis `.env`. Output menyebut kredensial disimpan di password manager dan file tidak boleh di-commit. Nilai rahasia tidak ditampilkan dalam laporan.

## Kotak Perintah C — Validasi Statis

```bash
$ ./scripts/validate-lab.sh
$ docker compose config --services
$ docker compose config --images
```

Keluaran minimum yang diharapkan:

```text
PASS: JSON realm dan kebijakan OIDC/RBAC valid.
PASS: validasi statis selesai.
db
keycloak
app
postgres:17.11-alpine3.24
quay.io/keycloak/keycloak:26.7.2
devsecops-bab09-flask:1.0
```

Hasil:

![Kotak Perintah C (14.png)](assets/14.png)

Analisa: validasi statis lolos. Output `validate-lab.sh` menyatakan JSON realm dan kebijakan OIDC/RBAC valid, serta validasi statis selesai tanpa FAIL.

## Kotak Perintah D — Build dan Menjalankan Laboratorium

```bash
$ docker compose pull
$ docker compose build --pull app
$ docker compose up -d
$ docker compose ps
$ docker compose logs --tail 100 keycloak
```

Tunggu sampai endpoint readiness dan aplikasi memberikan respons sukses.

```bash
$ until curl --fail --silent http://localhost:9000/health/ready >/dev/null; do sleep 3; done
$ curl --fail http://localhost:5000/healthz
$ curl --fail http://keycloak.localhost:8080/realms/devsecops-lab/.well-known/openid-configuration | python3 -m json.tool | sed -n '1,30p'
```

Keluaran health yang sahih:

```json
{"status":"UP"}
```

Hasil:

![Kotak Perintah D (15a.png)](assets/15a.png)
![Kotak Perintah D (15b.png)](assets/15b.png)
![Kotak Perintah D (15c.png)](assets/15c.png)

Analisa: `docker compose pull` menampilkan error `pull access denied` untuk `devsecops-bab09-flask`. Ini bukan kegagalan lab. Image Flask dibangun lokal lewat `build:`, sehingga tidak ada di registry untuk ditarik. Keycloak `26.7.2` berhasil ditarik. Volume `keycloak-db-data` dibuat dan `up -d` selesai. Endpoint readiness dan `/healthz` merespons, dengan `{"status":"UP"}` seperti yang diharapkan cheatsheet.

## Kotak Perintah E — Menguji Login dan RBAC

1. Buka `http://localhost:5000`.
2. Pilih **Login** dan autentikasikan `mahasiswa1` menggunakan password yang tersimpan pada `.env`.
3. Pastikan halaman utama hanya menampilkan claim minimum yang telah disanitasi.
4. Buka endpoint **Viewer**; hasil harus HTTP 200.
5. Buka endpoint **Admin**; hasil harus HTTP 403 karena `mahasiswa1` tidak memiliki role `admin`.
6. Jalankan **Logout SSO**, kemudian login sebagai `adminlab`.
7. Ulangi akses Viewer dan Admin; keduanya harus HTTP 200.

Respons viewer yang diharapkan:

```json
{
  "message": "Role viewer diterima.",
  "result": "PASS",
  "user": "mahasiswa1"
}
```

Respons negatif admin untuk `mahasiswa1`:

```json
{
  "error": "forbidden",
  "required_role": "admin"
}
```

Pengujian ini membedakan authentication success dari authorization success. `mahasiswa1` adalah identitas valid, tetapi konteks role-nya tidak memenuhi kebijakan endpoint `/admin`; respons 403 merupakan hasil yang benar.

Hasil:

![Kotak Perintah E (16a.png)](assets/16a.png)
![Kotak Perintah E (16b.png)](assets/16b.png)
![Kotak Perintah E (16c.png)](assets/16c.png)
![Kotak Perintah E (16d.png)](assets/16d.png)

Analisa: login dilakukan dengan akun `adminlab`. Screenshot `16d.png` menunjukkan endpoint Admin menerima akun ini dengan pesan `Role admin diterima.`. Akun `adminlab` memiliki role `viewer` dan `admin` pada realm JSON, sehingga hasil ini sesuai ekspektasi. Screenshot `16b.png` menampilkan `Role viewer diterima.`, tetapi akun yang dipakai pada screenshot itu tidak tercatat, sehingga laporan tidak mengklaim akun tertentu untuk hasil tersebut. Uji negatif bahwa `mahasiswa1` ditolak pada endpoint Admin (HTTP 403) belum dibuktikan dengan screenshot.

## Kotak Perintah F — Menjalankan Negative Test Otomatis

Keluar dari sesi browser sebelum pengujian agar cookie tidak memengaruhi observasi.

```bash
$ ./scripts/negative-tests.sh | tee reports/negative-tests.txt
```

Keluaran minimum:

```text
PASS: viewer tanpa sesi menghasilkan HTTP 302
PASS: admin tanpa sesi menghasilkan HTTP 302
PASS: redirect URI tidak terdaftar ditolak dengan HTTP 400.
PASS: permintaan tanpa PKCE ditolak dengan HTTP 400.
PASS: Direct Access Grant dinonaktifkan dan permintaan ditolak.
Jumlah kegagalan: 0
```

Hasil:

![Kotak Perintah F (17.png)](assets/17.png)

Analisa: skrip `negative-tests.sh` menjalankan lima pengujian. Empat lulus sesuai cheatsheet, yaitu akses tanpa sesi ke Viewer dan Admin yang diarahkan ke login (HTTP 302), redirect URI tidak terdaftar (HTTP 400), dan Direct Access Grant yang ditolak. Pengujian tanpa PKCE gagal pada versi awal skrip. Keycloak menolak permintaan tanpa `code_challenge_method` dengan redirect HTTP 302 ke `redirect_uri`, berisi `error=invalid_request`, dan tanpa `code`. Skrip awal mengharapkan HTTP 400. Perubahan yang dilakukan: pemeriksaan PKCE diubah agar mengharapkan status 302, `error=invalid_request` pada `Location`, dan tidak adanya `code=`. Skrip asli disimpan sebagai `scripts/negative-tests.sh.bak`. Perilaku keamanannya tetap sama: permintaan tanpa PKCE tidak menghasilkan kode otorisasi. Hasil akhir: lima pengujian PASS dengan jumlah kegagalan 0. Catatan: perubahan ekspektasi ini adalah keputusan yang diambil saat praktikum, bukan bagian dari modul asli.

## Kotak Perintah G — Memeriksa Hardening dan Persistensi

```bash
$ docker compose exec app id
$ docker inspect devsecops-bab09-app-1 --format 'ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{json .HostConfig.CapDrop}} Pids={{.HostConfig.PidsLimit}} Memory={{.HostConfig.Memory}}'
$ docker compose exec app sh -c 'touch /should-fail'
$ docker compose exec app sh -c 'touch /tmp/allowed && test -f /tmp/allowed'
$ docker compose exec db psql -U keycloak -d keycloak -c 'SELECT COUNT(*) AS realm_count FROM realm;'
$ docker compose restart keycloak
$ curl --fail http://keycloak.localhost:8080/realms/devsecops-lab/.well-known/openid-configuration >/dev/null
```

Interpretasi:

- `id` pada app harus menunjukkan UID/GID `10001`.
- `ReadOnly=true`, `CapDrop=["ALL"]`, PID `100`, dan memory `268435456` membuktikan batas runtime aplikasi.
- Penulisan `/should-fail` harus gagal; penulisan `/tmp/allowed` harus berhasil.
- Realm tetap tersedia setelah restart karena state tersimpan pada volume PostgreSQL.

Nama container yang dibuat Compose dapat berbeda. Jika `devsecops-bab09-app-1` tidak ditemukan, peroleh nama aktual dengan `docker compose ps --format json` atau gunakan `docker compose ps`.

Hasil:

![Kotak Perintah G (18.png)](assets/18.png)

Analisa: screenshot `18.png` menampilkan `docker compose exec app id` dengan hasil `uid=10001(appuser) gid=10001(appgroup)`, sehingga proses aplikasi berjalan sebagai user non-root. Nilai `ReadOnly`, `CapDrop`, `Pids`, dan `Memory` dari `docker inspect` tidak terbaca pada screenshot, sehingga belum diklaim di laporan. OCR membaca nama container sebagai `devsecops-bab9-app-1`. Pastikan nama aslinya sebelum laporan final.

## Kotak Perintah H — Mengumpulkan Evidence

```bash
$ ./scripts/collect-evidence.sh
$ find reports -maxdepth 1 -type f -printf '%f\n' 2>/dev/null | sort
$ sha256sum reports/* > reports-manifest-sha256.txt
```

Pada PowerShell, pengganti perintah `find` adalah:

```powershell
PS> Get-ChildItem .\reports -File | Select-Object -ExpandProperty Name
PS> Get-FileHash .\reports\* -Algorithm SHA256
```

Hasil:

![Kotak Perintah H (19.png)](assets/19.png)

Analisa: `collect-evidence.sh` menyatakan evidence tersimpan di `reports/`. Daftar file pada screenshot dicatat sebagai bukti, dan tinjauan ulang tetap perlu dilakukan sebelum dibagikan.

## Tabel Verifikasi PASS/FAIL

| ID | Kontrol atau skenario | Evidence yang absah | PASS | FAIL |
| --- | --- | --- | --- | --- |
| IAM-01 | PostgreSQL tidak memiliki port host | `docker compose config` dan `docker compose ps` | Tidak ada mapping `5432` | Port database dipublikasikan |
| IAM-02 | Keycloak dan app hanya pada loopback | Konfigurasi port efektif | Host IP `127.0.0.1` | Binding `0.0.0.0` atau seluruh interface |
| IAM-03 | Realm dapat direlingkungan produksi | JSON valid dan log import | Realm `devsecops-lab` tersedia | Import gagal atau realm tidak tersedia |
| IAM-04 | Redirect URI eksak | JSON realm dan negative test | URI penyerang ditolak HTTP 400 | Redirect tidak sah diterima |
| IAM-05 | PKCE S256 diwajibkan | Atribut client dan negative test | Permintaan tanpa challenge ditolak | Login dapat dimulai tanpa PKCE |
| IAM-06 | Direct Access Grant dimatikan | JSON client dan respons token endpoint | HTTP 400/401 | Password grant menghasilkan token |
| IAM-07 | State/nonce diverifikasi | Authlib OIDC flow dan callback rusak | Callback tidak sah ditolak | Callback acak membuat sesi |
| IAM-08 | Viewer berhak membaca | Login `mahasiswa1`, endpoint `/viewer` | HTTP 200 | HTTP 401/403/500 |
| IAM-09 | Viewer tidak menjadi admin | Login `mahasiswa1`, endpoint `/admin` | HTTP 403 | HTTP 200 |
| IAM-10 | Admin aplikasi memperoleh akses minimum | Login `adminlab`, dua endpoint | Viewer dan Admin HTTP 200 | Akses yang seharusnya diizinkan gagal |
| IAM-11 | Token tidak menjadi evidence | Pemeriksaan report dan log | Tidak ada token/secret utuh | Token atau secret tercetak |
| IAM-12 | Aplikasi diperkeras | `id` dan `docker inspect` | Non-root, read-only, cap drop, limit aktif | Salah satu kontrol tidak aktif |
| IAM-13 | State persisten | Restart Keycloak dan query database | Realm tetap tersedia | Realm hilang setelah restart |
| IAM-14 | Logout dipahami secara tepat | Uji logout lokal dan SSO | Perbedaan perilaku dapat dijelaskan | Logout lokal dianggap mencabut token/SSO |

## Troubleshooting dan Analisis Hasil

### T1. SELinux menolak Keycloak membaca realm

Notice: Keycloak mendapat `Permission denied` saat membaca `devsecops-lab-realm.json`. Label awal file adalah `user_home_t`. Perbaikan: `chcon -t container_file_t realm/devsecops-lab-realm.json`. Kebijakan SELinux tidak dinonaktifkan dan `compose.yaml` tidak diubah. Setelah perbaikan, Keycloak mengimpor realm dengan sukses.

### T2. Aplikasi Flask gagal start karena modul `requests`

Container `app` restart berulang dengan `ModuleNotFoundError: No module named 'requests'`. Perbaikan: `requests==2.32.5` ditambahkan ke `app/requirements.txt` (file asli: `requirements.txt.orig`). Build ulang dan start berhasil.

### T3. Role `roles` kosong pada halaman identitas

Gejala: login berhasil, tetapi daftar role kosong untuk `mahasiswa1`. Akun tersebut memiliki role `viewer` di Keycloak. Penyebab: aplikasi membaca `realm_access.roles` dari `userinfo`, sedangkan mapper `realm roles` pada client scope `roles` hanya menulis claim ke access token. Perubahan: pada instance Keycloak yang sedang berjalan, mapper `realm roles` diatur agar claim juga ditulis ke ID token dan `userinfo`. Setelah logout SSO dan login ulang, `roles` terisi. Catatan: perubahan ini belum ada di `realm/devsecops-lab-realm.json`. Jika lab dibuat ulang dari JSON, masalah ini bisa muncul lagi.

### T4. Atribut PKCE pada client `flask-app`

File realm mewajibkan `pkce.code.challenge.method = S256`. Di instance yang berjalan, atribut ini dicatat sebagai perubahan manual. Ekspektasi pengujian tanpa PKCE juga diubah, seperti dicatat pada Kotak F.

### T5. Port host 5000, 8080, dan 9000 tidak bisa diakses

Gejala: container `app` dan `keycloak` berstatus running atau healthy. Di dalam container, `/healthz` mengembalikan HTTP 200. Namun `curl` dari host ke `127.0.0.1:5000` dan `127.0.0.1:9000` gagal dengan `Could not connect to server`, sehingga perintah `until curl ... 9000/health/ready` pada Kotak D tidak pernah selesai.

Penyebab: pada `compose.yaml`, network `iam-net` memiliki `internal: true`. Network internal tidak mempublikasikan port ke host, meskipun `ports:` sudah ditulis. Stack lama juga belum dibuat ulang setelah konfigurasi berubah, sehingga network lama tetap dipakai.

Perbaikan: baris `internal: true` dikomentari, lalu stack dibuat ulang dengan `docker compose up -d`. Hasil: `docker port` menampilkan `127.0.0.1:5000`, `127.0.0.1:8080`, dan `127.0.0.1:9000`. Pemeriksaan `curl` dari host mengembalikan HTTP 200 untuk `/healthz` dan `/health/ready`.

Catatan: network internal memang menutup akses host, tetapi `iam-net` di lab ini perlu akses publikasi ke host untuk praktikum. Pada lingkungan produksi, aplikasi diakses lewat reverse proxy, bukan port langsung.

## Catatan Windows dan Linux

| Lingkungan | Catatan |
| --- | --- |
| Windows + Docker Desktop | Jalankan script melalui Git Bash atau WSL. Pastikan drive tempat proyek berada dapat diakses Docker Desktop. |
| Windows PowerShell | Perintah Docker dapat dijalankan langsung, tetapi script Bash memerlukan Git Bash/WSL. Gunakan `Get-Content .env` dengan hati-hati agar secret tidak terekam. |
| Linux native | User harus menjadi anggota grup yang berhak mengakses Docker socket; keanggotaan grup `docker` setara dengan hak istimewa tinggi. |
| Semua platform | Bila `keycloak.localhost` tidak resolve, tambahkan mapping hosts secara administratif: `127.0.0.1 keycloak.localhost`. |

Lokasi hosts file:

- Linux: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`

Perubahan hosts file memerlukan hak administrator. Pastikan hanya nama `keycloak.localhost` yang ditambahkan, bukan wildcard.

## Kotak Perintah I — Cleanup

Cleanup biasa mempertahankan volume database:

```bash
$ docker compose down --remove-orphans
```

Cleanup penuh berikut menghapus container, network, volume database, realm yang telah diimpor, dan image aplikasi lokal. Jalankan hanya apabila evidence sudah disimpan dan data laboratorium tidak lagi diperlukan.

```bash
$ docker compose down --volumes --remove-orphans
$ docker image rm devsecops-bab09-flask:1.0
```

File `.env` mengandung kredensial. Hapus melalui mekanisme aman yang tersedia pada sistem setelah laporan selesai, atau simpan terenkripsi bila laboratorium akan dilanjutkan.

Hasil:

![Kotak Perintah I (20.png)](assets/20.png)

Analisa: `docker compose down --remove-orphans` berhasil dan container aplikasi dihapus dengan exit code 0. Data volume tidak ikut dihapus pada perintah ini.

## Latihan Mandiri

1. Tambahkan role `auditor` yang hanya dapat mengakses endpoint `/audit`. Buktikan matriks allow/deny untuk ketiga akun.
2. Ubah access token lifespan menjadi 60 detik. Jelaskan perbedaan antara expiration access token, sesi Flask, dan sesi SSO Keycloak.
3. Tambahkan user ke group `kelas-a`, petakan role melalui group, lalu bandingkan kemudahan audit dengan direct role assignment.
4. Lakukan callback dengan nilai `state` acak. Catat status HTTP tanpa merekam token atau cookie.
5. Ubah satu karakter pada ID Token di lingkungan pengujian terpisah dan jelaskan mengapa signature validation harus gagal. Jangan memasukkan token asli ke laporan.
6. Rancang migrasi dari HTTP `start-dev` menuju deployment lingkungan produksi dengan reverse proxy TLS, secret manager, backup, metrics, dan minimal dua instance.
7. Jelaskan risiko penggunaan wildcard redirect URI, `fullScopeAllowed`, Direct Access Grant, dan client secret bersama.
8. Buat access review sederhana yang menampilkan user, group, direct role, inherited role, owner, waktu tinjau, serta keputusan retain/revoke.

## Rujukan Utama

1. Keycloak, *Running Keycloak in a Container*: https://www.keycloak.org/server/containers
2. Keycloak, *Downloads*: https://www.keycloak.org/downloads
3. Keycloak, *Importing and Exporting Realms*: https://www.keycloak.org/server/importExport
4. Keycloak, *Tracking Instance Status with Health Checks*: https://www.keycloak.org/observability/health
5. Keycloak, *Securing Applications and Services with OpenID Connect*: https://www.keycloak.org/securing-apps/oidc-layers
6. Keycloak, *Server Administration Guide*: https://www.keycloak.org/docs/latest/server_admin/
7. OpenID Foundation, *OpenID Connect Core 1.0 incorporating errata set 2*: https://openid.net/specs/openid-connect-core-1_0.html
8. IETF, *RFC 9700 — Best Current Practice for OAuth 2.0 Security*: https://www.rfc-editor.org/rfc/rfc9700
9. IETF, *RFC 7636 — Proof Key for Code Exchange*: https://www.rfc-editor.org/rfc/rfc7636
10. IETF, *RFC 6761 — Special-Use Domain Names*: https://www.rfc-editor.org/rfc/rfc6761
11. Authlib, *Web OAuth Clients*: https://docs.authlib.org/en/stable/oauth2/client/web/index.html
12. PostgreSQL Official Image: https://hub.docker.com/_/postgres
13. Flask on PyPI: https://pypi.org/project/Flask/
14. Authlib on PyPI: https://pypi.org/project/Authlib/
15. Gunicorn on PyPI: https://pypi.org/project/gunicorn/

> **Prinsip evidence:** hasil yang dianggap PASS harus menunjukkan bukan hanya keberhasilan login, tetapi juga penolakan konsisten terhadap identitas, alur, redirect, dan role yang tidak memenuhi kebijakan. Token, cookie sesi, password, dan client secret bukan evidence yang boleh dimasukkan ke laporan.
