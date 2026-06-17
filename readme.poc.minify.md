# PoC Middleware API Encryption — Quick Guide

## Step 1: Deploy

```bash
# Load image
docker load -i middleware-api-secure.tar.gz

# Edit docker-compose.yaml → isi UPSTREAM_URL dan ENCRYPT_SECRET
# ⚠️ Kedua nilai ini ditanyakan/disepakati dengan tim Jasa Raharja
#    - UPSTREAM_URL  : URL API internal JR yang mau diproteksi
#    - ENCRYPT_SECRET: secret 32 karakter, harus sama di middleware & client

# Jalankan
docker compose up -d

# Verifikasi
curl http://localhost:8081/health   # → {"status":"ok"}
```

---

## Step 2: Registrasi Route di Kong

> Step ini dilakukan oleh **tim Jasa Raharja**.

Sampaikan ke tim JR untuk mendaftarkan route baru di Kong API Gateway:

- **Service URL:** `http://<IP_SERVER_MIDDLEWARE>:8081`
- **Health Check:** `GET /health`
- **Methods:** GET, POST, PUT, PATCH, DELETE

Alur setelah route terdaftar:

```
Client → Kong → Middleware (:8081) → Upstream API JR
                                          ↓
Client ← Kong ← Middleware (encrypt) ← response asli
```

Verifikasi:

```bash
curl https://<KONG_URL>/<route-baru>/health   # → {"status":"ok"}
```

---

## Step 3: Perbandingan Existing vs Encrypted

**Route existing** (tanpa enkripsi):

```bash
curl -s https://<KONG_URL>/<route-existing>/users?id=1 -H "apiKey: <key>" | jq .
```
```json
{ "id": 1, "name": "John Doe", "nik": "3201234567890001", "saldo": 15000000 }
```

**Route baru** (dengan enkripsi):

```bash
curl -s https://<KONG_URL>/<route-baru>/users?id=1 -H "apiKey: <key>" | jq .
```
```json
{ "encrypted": true, "algorithm": "AES-256-CBC", "data": "a1b2c3...:U2FsdGVk..." }
```

> Data sama, upstream sama, tapi response sudah tidak bisa dibaca tanpa secret key.
> Upstream API **zero change** — tidak perlu ubah satu baris pun di backend.

---

## Step 4: Encrypt & Decrypt

### Setup crypto-client

```bash
git clone https://github.com/dhomanhuri/crypto-client.git
cd crypto-client
docker compose --profile nodejs --profile python --profile golang build
```

### Decrypt response dari middleware

```bash
RESPONSE=$(curl -s https://<KONG_URL>/<route-baru>/users?id=1 -H "apiKey: <key>" | jq -r '.data')

# Pilih salah satu bahasa:
docker compose --profile nodejs run --rm crypto-nodejs decrypt "$RESPONSE"
docker compose --profile python run --rm crypto-python decrypt "$RESPONSE"
docker compose --profile golang run --rm crypto-golang decrypt "$RESPONSE"
```

### Encrypt payload

```bash
docker compose --profile nodejs run --rm crypto-nodejs encrypt '{"name":"Ilyas"}'
```

### Cross-language test

```bash
ENCRYPTED=$(docker compose --profile nodejs run --rm crypto-nodejs encrypt '{"test":true}' 2>/dev/null)
docker compose --profile python run --rm crypto-python decrypt "$ENCRYPTED"
docker compose --profile golang run --rm crypto-golang decrypt "$ENCRYPTED"
# → Semua menghasilkan output yang sama
```

> ⚠️ Secret key di crypto-client harus sama dengan `ENCRYPT_SECRET` di middleware.
