# 🔐 Middleware API Gateway — Deployment Package

Package ini berisi Docker image siap deploy untuk **Middleware API Gateway** dengan enkripsi response AES-256-CBC.

---

## 📦 Isi Package

| File | Keterangan |
|---|---|
| `middleware-api-secure.tar.gz` | Docker image siap deploy |
| `docker-compose.yaml` | Konfigurasi Docker Compose |
| `README.md` | Panduan ini |

---

## ✅ Prasyarat

- **Docker Engine** ≥ 20.x terinstall di server
- Tidak perlu Node.js, npm, atau apapun selain Docker

---

## 🚀 Cara Deploy

### Langkah 1 — Load image

```bash
docker load -i middleware-api-secure.tar.gz
```

### Langkah 2 — Sesuaikan konfigurasi

Edit file `docker-compose.yaml`, ganti nilai berikut:

```yaml
environment:
  UPSTREAM_URL: https://api-target.com        # ← ganti dengan URL upstream Anda
  ENCRYPT_SECRET: your-32-char-secret-here!   # ← ganti dengan secret 32 karakter Anda
```

> ⚠️ `ENCRYPT_SECRET` harus tepat **32 karakter**.

### Langkah 3 — Jalankan

```bash
docker compose up -d
```

### Langkah 4 — Verifikasi

```bash
# Cek container berjalan
docker compose ps

# Cek log
docker compose logs -f

# Health check
curl http://localhost:8081/health
# Expected: {"status":"ok"}
```

---

## ⚙️ Konfigurasi

| Variable | Keterangan |
|---|---|
| `UPSTREAM_URL` | Base URL API yang dituju (upstream) |
| `ENCRYPT_SECRET` | Secret key AES-256-CBC, **harus 32 karakter** |

Port default yang diekspos: **`8081`**

Untuk mengubah port, edit bagian `ports` di `docker-compose.yaml`:
```yaml
ports:
  - "PORT_ANDA:3000"
```

---

## 🛑 Stop & Remove

```bash
# Stop container
docker compose down

# Stop + hapus image
docker compose down --rmi all
```

---

## 📡 Cara Kerja

Semua request yang masuk ke gateway akan:
1. Diteruskan ke `UPSTREAM_URL` dengan path, query param, dan header yang sama
2. Response dari upstream **dienkripsi AES-256-CBC** sebelum dikembalikan ke client

Format response:
```json
{
  "encrypted": true,
  "algorithm": "AES-256-CBC",
  "data": "<iv_hex>:<base64_ciphertext>"
}
```

---

## 🔓 Cara Decrypt Response di Client

**Node.js**
```js
const crypto = require("crypto");

function decrypt(encryptedData, secret) {
  const [ivHex, enc] = encryptedData.split(":");
  const iv = Buffer.from(ivHex, "hex");
  const key = Buffer.from(secret.slice(0, 32));
  const decipher = crypto.createDecipheriv("aes-256-cbc", key, iv);
  let decrypted = decipher.update(enc, "base64", "utf8");
  decrypted += decipher.final("utf8");
  return JSON.parse(decrypted);
}

const response = await fetch("http://localhost:8081/your-endpoint");
const body = await response.json();
const data = decrypt(body.data, "your-32-char-secret-here!");
console.log(data);
```

**Python**
```python
import base64, json
from Crypto.Cipher import AES

SECRET = b"your-32-char-secret-here!"

def decrypt(encrypted_data: str):
    iv_hex, enc = encrypted_data.split(":")
    iv = bytes.fromhex(iv_hex)
    enc_bytes = base64.b64decode(enc)
    cipher = AES.new(SECRET[:32], AES.MODE_CBC, iv)
    decrypted = cipher.decrypt(enc_bytes)
    pad = decrypted[-1]
    return json.loads(decrypted[:-pad].decode())
```

---

## ❤️ Endpoint Internal

| Endpoint | Keterangan |
|---|---|
| `GET /` | Info service |
| `GET /health` | Health check untuk monitoring |

Semua endpoint lainnya akan di-proxy ke upstream.
