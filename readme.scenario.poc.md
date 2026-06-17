# Skenario PoC — Middleware API Gateway

## Step 1: Deploy Middleware API

### Prasyarat

- Server / VM / laptop dengan **Docker Engine** ≥ 20.x
- File deployment package:
  - `middleware-api-secure.tar.gz`
  - `docker-compose.yaml`
- Mengetahui **URL upstream API** yang akan di-proxy
- Menyiapkan **secret key** (harus tepat 32 karakter)

---

### 1.1 — Load Docker Image

```bash
docker load -i middleware-api-secure.tar.gz
```

Verifikasi image berhasil di-load:

```bash
docker images | grep middleware-api
```

Output yang diharapkan:

```
middleware-api   secure   abc123def456   ...   ~50MB
```

---

### 1.2 — Konfigurasi Environment

Edit file `docker-compose.yaml`, sesuaikan dua variable berikut:

```yaml
environment:
  UPSTREAM_URL: http://192.168.1.100:8080   # Ganti dengan URL upstream API
  ENCRYPT_SECRET: abcdefghijklmnopqrstuvwxyz123456   # Ganti dengan secret 32 karakter
```

> ⚠️ **Penting — Kedua nilai ini harus dikonfirmasi oleh tim Jasa Raharja:**
>
> | Variable | Yang perlu ditanyakan |
> |---|---|
> | `UPSTREAM_URL` | Minta ke tim Jasa Raharja: URL API internal yang mau diproteksi. Contoh: `http://10.x.x.x:8080/api/v1` atau `https://api-internal.jasaraharja.co.id`. URL ini adalah API yang selama ini diakses langsung, yang nantinya akan dilewatkan melalui middleware ini. |
> | `ENCRYPT_SECRET` | Tentukan bersama tim Jasa Raharja: secret key 32 karakter yang akan dipakai untuk enkripsi. Secret ini harus **sama persis** di sisi middleware dan di sisi client yang melakukan decrypt. Disarankan menggunakan kombinasi huruf, angka, dan simbol. |
>
> Tanpa kedua informasi ini, middleware tidak bisa diarahkan ke API yang benar dan client tidak bisa mendekripsi response.

**Catatan tambahan:**
- `UPSTREAM_URL` → base URL API yang ingin di-proxy (tanpa trailing slash)
- `ENCRYPT_SECRET` → harus **tepat 32 karakter**, digunakan untuk enkripsi AES-256-CBC
- Port default gateway: **8081** (bisa diubah di bagian `ports`)

---

### 1.3 — Jalankan Container

```bash
docker compose up -d
```

---

### 1.4 — Verifikasi Deployment

**Cek container berjalan:**

```bash
docker compose ps
```

Output yang diharapkan:

```
NAME              IMAGE                    STATUS          PORTS
middleware-api    middleware-api:secure    Up X minutes    0.0.0.0:8081->3000/tcp
```

**Cek log aplikasi:**

```bash
docker compose logs -f
```

Output yang diharapkan:

```
middleware-api  | Middleware API is running on port 3000
middleware-api  | Upstream: http://192.168.1.100:8080
```

Tekan `Ctrl+C` untuk keluar dari log.

**Health check:**

```bash
curl http://localhost:8081/health
```

Output yang diharapkan:

```json
{"status":"ok"}
```

**Cek info service:**

```bash
curl http://localhost:8081/
```

Output yang diharapkan:

```json
{
  "message": "Middleware API is running 🚀",
  "upstream": "http://192.168.1.100:8080",
  "encryption": "AES-256-CBC"
}
```

---

### 1.5 — Troubleshooting

| Masalah | Penyebab | Solusi |
|---|---|---|
| `docker load` error | File `.tar.gz` corrupt atau tidak lengkap | Download ulang / copy ulang file |
| Container exit / restart loop | `ENCRYPT_SECRET` bukan 32 karakter | Cek panjang secret, harus tepat 32 |
| Health check timeout | Port 8081 tidak terbuka | Cek firewall, pastikan port 8081 tidak diblokir |
| `502 Bad Gateway` saat hit endpoint | Upstream tidak bisa diakses dari container | Cek `UPSTREAM_URL`, pastikan bisa diakses dari dalam Docker network |
| Upstream di localhost / host lokal | Docker container tidak bisa akses `localhost` host | Tambahkan `extra_hosts: ["host.docker.internal:host-gateway"]` di `docker-compose.yaml` |

**Contoh tambahan config untuk upstream di host lokal:**

```yaml
services:
  middleware-api:
    image: middleware-api:secure
    container_name: middleware-api
    restart: unless-stopped
    ports:
      - "8081:3000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      UPSTREAM_URL: http://host.docker.internal:8080
      ENCRYPT_SECRET: abcdefghijklmnopqrstuvwxyz123456
```

---

### ✅ Checklist Step 1

- [ ] Docker image berhasil di-load
- [ ] `docker-compose.yaml` sudah dikonfigurasi (UPSTREAM_URL + ENCRYPT_SECRET)
- [ ] Container berjalan (`docker compose ps` → status Up)
- [ ] Health check berhasil (`/health` → `{"status":"ok"}`)
- [ ] Info service menampilkan upstream yang benar (`/`)

> Jika semua checklist di atas sudah ✅, lanjut ke **Step 2: Registrasi Route di Kong API Gateway**.

---

## Step 2: Registrasi Route di Kong API Gateway

Setelah middleware berhasil di-deploy, langkah selanjutnya adalah meminta tim Jasa Raharja untuk mendaftarkan route baru di **Kong API Gateway** yang mengarah ke middleware ini.

> ⚠️ **Step ini dilakukan oleh tim Jasa Raharja**, karena Kong API Gateway dikelola oleh mereka.

---

### 2.1 — Informasi yang Perlu Disiapkan

Berikut informasi yang perlu disampaikan ke tim Jasa Raharja untuk didaftarkan di Kong:

| Informasi | Nilai | Keterangan |
|---|---|---|
| **Service URL** | `http://<IP_SERVER_MIDDLEWARE>:8081` | IP/hostname server tempat middleware di-deploy, port 8081 |
| **Path / Route** | Disepakati bersama | Path yang akan diakses client melalui Kong, misalnya `/api/secure/*` |
| **Protocol** | HTTP / HTTPS | Sesuai kebijakan Jasa Raharja |
| **Method** | Semua (GET, POST, PUT, DELETE, dll) | Middleware mendukung semua HTTP method |

---

### 2.2 — Yang Perlu Diminta ke Tim Jasa Raharja

Kirimkan request ke tim infra / API management Jasa Raharja dengan informasi berikut:

```
Subject: Request Registrasi Route Baru di Kong API Gateway — PoC Middleware Encryption

Yth. Tim API Management Jasa Raharja,

Dalam rangka PoC Middleware API Encryption, mohon dibantu untuk mendaftarkan 
route baru di Kong API Gateway dengan konfigurasi berikut:

1. Service (Upstream Target)
   - URL : http://<IP_SERVER_MIDDLEWARE>:8081
   - Health Check : GET /health

2. Route
   - Path       : <disepakati bersama>
   - Methods    : GET, POST, PUT, PATCH, DELETE
   - Protocols  : http / https
   - Strip Path : sesuai kebutuhan

3. Catatan
   - Middleware ini akan mem-forward semua request ke upstream API 
     yang sudah dikonfigurasi di dalamnya
   - Response dari middleware sudah dalam format terenkripsi (AES-256-CBC)
   - Endpoint /health tersedia untuk health check / monitoring

Mohon informasikan jika ada tambahan konfigurasi yang diperlukan dari sisi kami.

Terima kasih.
```

---

### 2.3 — Alur Setelah Route Terdaftar

Setelah tim Jasa Raharja mendaftarkan route di Kong, alur request menjadi:

```
Client (Mobile App / Frontend)
  │
  │  request via Kong
  ▼
Kong API Gateway
  │
  │  forward ke middleware
  ▼
Middleware API Gateway (:8081)
  │
  │  forward ke upstream API (yang sudah dikonfigurasi)
  ▼
Upstream API Jasa Raharja
  │
  │  response asli
  ▼
Middleware API Gateway
  │
  │  encrypt response (AES-256-CBC)
  ▼
Kong API Gateway
  │
  ▼
Client  ←  { encrypted: true, algorithm: "AES-256-CBC", data: "..." }
```

---

### 2.4 — Verifikasi Route

Setelah route terdaftar, test dari sisi Kong:

```bash
# Health check melalui Kong
curl https://<KONG_URL>/<path-yang-didaftarkan>/health
```

Output yang diharapkan:

```json
{"status":"ok"}
```

```bash
# Test hit endpoint melalui Kong
curl https://<KONG_URL>/<path-yang-didaftarkan>/users?id=1
```

Output yang diharapkan (response terenkripsi):

```json
{
  "encrypted": true,
  "algorithm": "AES-256-CBC",
  "data": "<iv_hex>:<base64_ciphertext>"
}
```

---

### ✅ Checklist Step 2

- [ ] Informasi service URL & port sudah disampaikan ke tim Jasa Raharja
- [ ] Tim Jasa Raharja sudah mendaftarkan route di Kong
- [ ] Health check melalui Kong berhasil (`/health` → `{"status":"ok"}`)
- [ ] Test endpoint melalui Kong mengembalikan response terenkripsi

> Jika semua checklist di atas sudah ✅, lanjut ke **Step 3: Perbandingan Route Existing vs Route Baru**.

---

## Step 3: Perbandingan Route Existing vs Route Baru (Tanpa Enkripsi vs Dengan Enkripsi)

Sebelum masuk ke pembahasan encrypt/decrypt, kita perlu buktikan dulu bahwa **data yang sama** bisa diakses lewat dua jalur — dan tunjukkan perbedaan dari sisi keamanan response.

> 🎯 **Tujuan step ini:** Menunjukkan ke tim Jasa Raharja bahwa di route existing, data terekspos dalam bentuk plain text. Sementara di route baru (via middleware), data yang sama sudah terenkripsi.

---

### 3.1 — Test via Route Existing (Tanpa Enkripsi)

Hit API melalui route Kong yang **sudah ada** (yang selama ini dipakai):

```bash
# Contoh: GET data user via route existing
curl -s https://<KONG_URL>/<route-existing>/users?id=1 \
  -H "apiKey: <api-key-jasa-raharja>" | jq .
```

Response (**plain text** — data terekspos langsung):

```json
{
  "id": 1,
  "name": "John Doe",
  "nik": "3201234567890001",
  "no_polis": "POL-2025-00001",
  "status": "active",
  "saldo": 15000000
}
```

> ⚠️ **Perhatikan:** Semua data sensitif (NIK, nomor polis, saldo) terlihat jelas di response. Siapa pun yang bisa intercept traffic atau punya akses ke log API bisa membaca data ini.

---

### 3.2 — Test via Route Baru (Dengan Enkripsi)

Hit API yang **sama** tapi melalui route baru (via middleware):

```bash
# Contoh: GET data user via route baru (melalui middleware)
curl -s https://<KONG_URL>/<route-baru>/users?id=1 \
  -H "apiKey: <api-key-jasa-raharja>" | jq .
```

Response (**terenkripsi** — data tidak bisa dibaca langsung):

```json
{
  "encrypted": true,
  "algorithm": "AES-256-CBC",
  "data": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6:U2FsdGVkX1+vupppZksvRf5pq5g0vJw..."
}
```

> ✅ **Data yang sama, tapi sekarang tidak bisa dibaca** tanpa secret key. Bahkan jika traffic di-intercept, yang terlihat hanya ciphertext.

---

### 3.3 — Perbandingan Side-by-Side

Buat dua terminal berdampingan, jalankan request ke endpoint yang sama:

| | Route Existing (Tanpa Enkripsi) | Route Baru (Dengan Enkripsi) |
|---|---|---|
| **Endpoint** | `https://<KONG>/<route-existing>/users?id=1` | `https://<KONG>/<route-baru>/users?id=1` |
| **Data di response** | Plain text JSON | Encrypted string |
| **NIK terlihat?** | ❌ Ya, terekspos | ✅ Tidak, terenkripsi |
| **Nomor polis terlihat?** | ❌ Ya, terekspos | ✅ Tidak, terenkripsi |
| **Saldo terlihat?** | ❌ Ya, terekspos | ✅ Tidak, terenkripsi |
| **Bisa dibaca di log/proxy?** | ❌ Ya | ✅ Tidak tanpa secret key |
| **Perubahan di upstream API?** | — | Tidak ada, zero change |

---

### 3.4 — Poin Penting untuk Presentasi

Saat demo ke tim Jasa Raharja, tekankan:

1. **Upstream API tidak berubah sama sekali** — middleware duduk di depannya dan hanya meng-encrypt response
2. **Route existing tetap jalan seperti biasa** — tidak terganggu, ini hanya menambahkan route baru
3. **Implementasi zero-code di sisi API** — tidak perlu ubah satu baris pun di backend
4. **Enkripsi terjadi di layer transport** — melindungi data dari man-in-the-middle, log exposure, dan unauthorized access
5. **Bisa di-rollback kapan saja** — tinggal hapus route baru di Kong, selesai

---

### ✅ Checklist Step 3

- [ ] Berhasil hit endpoint via route existing — response plain text
- [ ] Berhasil hit endpoint via route baru — response terenkripsi
- [ ] Data di route existing dan route baru mengarah ke upstream yang sama
- [ ] Perbandingan side-by-side sudah didemonstrasikan

> Jika semua checklist di atas sudah ✅, lanjut ke **Step 4: Testing Encrypt & Decrypt**.

---

## Step 4: Testing Encrypt & Decrypt

Sekarang kita buktikan bahwa response terenkripsi dari middleware **bisa didekripsi kembali** menjadi data asli menggunakan tool crypto-client.

---

### 4.1 — Setup Crypto Client

Clone tool crypto-client:

```bash
git clone https://github.com/dhomanhuri/crypto-client.git
cd crypto-client
```

Build image (tersedia dalam 3 bahasa: Node.js, Python, Go):

```bash
docker compose --profile nodejs --profile python --profile golang build
```

> Tool ini menggunakan algoritma dan format yang **identik** dengan middleware:
>
> | Parameter | Value |
> |---|---|
> | Algorithm | AES-256-CBC |
> | Key length | 32 bytes |
> | IV | Random 16 bytes per request |
> | Format output | `iv_hex:ciphertext_base64` |

---

### 4.2 — Test Encrypt (Simulasi Client Mengirim Data Terenkripsi)

Encrypt sebuah JSON payload:

```bash
# Node.js
docker compose --profile nodejs run --rm crypto-nodejs encrypt '{"name":"Ilyas","nik":"3201234567890001"}'

# Python
docker compose --profile python run --rm crypto-python encrypt '{"name":"Ilyas","nik":"3201234567890001"}'

# Go
docker compose --profile golang run --rm crypto-golang encrypt '{"name":"Ilyas","nik":"3201234567890001"}'
```

Output (contoh):

```
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6:U2FsdGVkX1+vupppZksvRf5pq5g0vJw...
```

---

### 4.3 — Test Decrypt Response dari Middleware

Ambil response terenkripsi dari middleware, lalu decrypt:

```bash
# 1. Hit endpoint via route baru, simpan response
RESPONSE=$(curl -s https://<KONG_URL>/<route-baru>/users?id=1 \
  -H "apiKey: <api-key>" | jq -r '.data')

echo "Encrypted: $RESPONSE"

# 2. Decrypt menggunakan crypto-client

# Node.js
docker compose --profile nodejs run --rm crypto-nodejs decrypt "$RESPONSE"

# Python
docker compose --profile python run --rm crypto-python decrypt "$RESPONSE"

# Go
docker compose --profile golang run --rm crypto-golang decrypt "$RESPONSE"
```

Output (data asli kembali):

```json
{
  "id": 1,
  "name": "John Doe",
  "nik": "3201234567890001",
  "no_polis": "POL-2025-00001",
  "status": "active",
  "saldo": 15000000
}
```

> ✅ Data berhasil didekripsi — identik dengan response plain text di route existing.

---

### 4.4 — Cross-Language Compatibility

Buktikan bahwa encrypt di satu bahasa bisa di-decrypt di bahasa lain:

```bash
# Encrypt di Node.js
ENCRYPTED=$(docker compose --profile nodejs run --rm crypto-nodejs encrypt '{"test":"cross-language"}' 2>/dev/null)

echo "Encrypted by Node.js: $ENCRYPTED"

# Decrypt di Python
docker compose --profile python run --rm crypto-python decrypt "$ENCRYPTED"

# Decrypt di Go
docker compose --profile golang run --rm crypto-golang decrypt "$ENCRYPTED"
```

Semua bahasa menghasilkan output yang sama:

```json
{"test":"cross-language"}
```

> 💡 Ini penting karena artinya tim Jasa Raharja **bebas memilih bahasa** untuk implementasi di sisi client — tidak terikat satu teknologi.

---

### 4.5 — Custom Secret Key

Jika menggunakan secret key yang berbeda dari default:

```bash
# Set secret via environment variable
ENCRYPT_SECRET=your-32-char-secret-key-here!! docker compose --profile nodejs run --rm crypto-nodejs encrypt '{"data":"rahasia"}'
```

> ⚠️ **Secret key di crypto-client harus sama persis** dengan `ENCRYPT_SECRET` yang dikonfigurasi di middleware. Jika berbeda, decrypt akan gagal.

---

### ✅ Checklist Step 4

- [ ] Crypto-client berhasil di-clone dan di-build
- [ ] Encrypt berhasil — output berupa `iv_hex:ciphertext_base64`
- [ ] Decrypt response dari middleware berhasil — data kembali ke plain text asli
- [ ] Cross-language test berhasil (encrypt di satu bahasa, decrypt di bahasa lain)
- [ ] Secret key sudah disesuaikan dengan yang dipakai di middleware

> Jika semua checklist di atas sudah ✅, PoC selesai. Middleware API Gateway siap untuk evaluasi lebih lanjut oleh tim Jasa Raharja.
