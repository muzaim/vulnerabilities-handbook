# Replay Attack (Serangan Pemutaran Ulang)

> **Metadata Keamanan:**
> - **CWE**: [CWE-294: Authentication Bypass by Capture-replay](https://cwe.mitre.org/data/definitions/294.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 6.5 - 7.5)

---

## 1. Pengertian

**Replay Attack** (Serangan Pemutaran Ulang) adalah serangan di mana penyerang menangkap atau menyalin transmisi data transaksi yang sah dan valid di masa lalu, lalu mengirimkannya kembali (*re-transmitting*) ke server berulang kali tanpa memodifikasi datanya, untuk memicu eksekusi tindakan berulang yang tidak sah.

Meskipun paket data tersebut memiliki tanda tangan digital yang valid atau terenkripsi secara sah saat dibuat, server yang tidak memiliki mekanisme pencegahan pemutaran ulang akan menganggap setiap pengiriman ulang tersebut sebagai perintah baru yang sah.

**Analogi Sederhana:**  
Bayangkan Anda memesan paket makanan dari katering dengan mengirimkan kupon bertuliskan: *"Berikan 1 porsi nasi kotak kepada pembawa kupon ini (Tertanda: Direktur)"*.  
Pihak katering memberikan nasi kotak pertama kepada Anda, tetapi katering lupa menyobek kupon tersebut dan mengembalikannya utuh ke Anda.  
Keesokan harinya, seorang office boy yang menemukan kupon utuh tersebut membawanya kembali ke katering yang sama. Karena katering hanya memeriksa keaslian tanda tangan direktur (dan tidak mencatat apakah kupon itu sudah pernah ditukarkan), katering memberikan nasi kotak kedua secara gratis.

---

## 2. Cara Kerja

Setiap transaksi yang dikirimkan klien memiliki payload tertentu, misalnya:
```http
POST /api/v1/payments/transfer
Authorization: Bearer <valid_token>
Content-Type: application/json

{
  "from_account": "1001",
  "to_account": "9999",
  "amount": 500000
}
```

Jika server tidak menerapkan konsep **Idempotency Key**, **Timestamp Expiry**, atau **Nonce** (Number used Once):
1. Penyerang menyalin persis paket request di atas dari riwayat alat uji (Postman, browser history, atau proxy jaringan).
2. Penyerang menembakkan ulang request tersebut sebanyak 10 kali.
3. Server memvalidasi token otentikasi -> Token sah.
4. Server memproses pemotongan saldo sebanyak 10 kali (total Rp 5.000.000).

---

## 3. Penyebab

- **Ketiadaan Parameter Nonce (*Number Used Once*)**: Tidak ada angka acak sekali pakai yang dicatat di database server.
- **Ketiadaan Pembatasan Masa Berlaku Timestamp**: Server menerima request yang dibuat berjam-jam atau berhari-hari yang lalu.
- **Ketiadaan Idempotency Key pada Endpoint Mutasi Finansial**: Endpoint POST pembayaran tidak memverifikasi apakah request dengan ID transaksi yang sama sudah pernah diproses sebelumnya.

---

## 4. Skenario Serangan

1. **Penyadapan / Observasi Request**: Penyerang mengamati lalu lintas API saat pengguna melakukan penukaran reward poin: `POST /api/rewards/redeem`.
2. **Ketiadaan Idempotensi**: Permintaan tersebut tidak memiliki tanda unik transaksi sekali pakai.
3. **Pengiriman Ulang Berantai (Replaying)**: Penyerang menggunakan skrip untuk mengirim ulang paket data yang persis sama sebanyak 50 kali dalam hitungan menit.
4. **Duplikasi Pengiriman**: Sistem memproses ke-50 permintaan tersebut dan mengirimkan 50 voucher hadiah ke akun penyerang, padahal poin yang dimiliki hanya cukup untuk 1 voucher.

---

## 5. Dampak

- **Pengurasan Saldo Finansial Berganda**: Mengirimkan transaksi pengeluaran dana berulang kali.
- **Bypass Autentikasi Challenge-Response**: Mengirimkan ulang paket respon otentikasi lama untuk masuk ke sistem tanpa kata sandi.
- **Eksploitasi Perangkat IoT / Smart Home**: Menangkap sinyal perintah pembuka gerbang otomatis (*unlock car / garage door*) dan memutarnya kembali saat pemilik rumah pergi.

---

## 6. Bagian Sistem yang Terdampak

- API pembayaran dan transfer perbankan / e-wallet.
- Webhook callback dari payment gateway (Midtrans, Xendit, Stripe).
- Alur login berbasis tanda tangan digital (Web3 / Ethereum signature / Kerberos).
- Perintah kendali perangkat keras jarak jauh (IoT / Smart Lock).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Pembayaran Merchant PayKopi:**  
Aplikasi kasir PayKopi mengirimkan request pembayaran saat pelanggan memindai QRIS di meja:
`POST /api/pos/charge`

Backend memverifikasi tanda tangan digital payload, tetapi tidak memeriksa *timestamp* atau *transaction id*.

Seorang pelanggan yang mengerti jaringan menyalin request HTTP pembayaran kopi seharga Rp 35.000 miliknya. Dua minggu kemudian, saat memesan kopi lagi di gerai yang sama, pelanggan tersebut tidak membayar ke kasir, melainkan menembakkan ulang paket request dua minggu lalu melalui cURL di ponselnya. Mesin kasir mencetak struk "LUNAS" karena server memvalidasi tanda tangan lama tersebut sebagai transaksi yang sah.

---

## 8. Cara Mendeteksi

- **Uji Coba Pengiriman Ulang (Replay Test)**: Tangkap request transaksi di Burp Suite Repeater, lalu klik tombol "Send" untuk kedua kalinya. Jika aksi kedua berhasil diproses dan status 200 OK, sistem rentan Replay Attack.
- **Inspeksi Header Request**: Periksa apakah ada header `Idempotency-Key`, `X-Timestamp`, atau `X-Nonce`.

---

## 9. Pencegahan

1. **Wajibkan Header Idempotency-Key pada Operasi Finansial (Standar Industri)**:  
   Klien wajib menyertakan UUID unik pada header (misal `Idempotency-Key: 550e8400...`). Server mencatat ID ini di Redis/Database. Jika ada request dengan Idempotency-Key yang sama dalam 24 jam, server mengembalikan respon yang sama tanpa memproses transaksi ulang.
2. **Gunakan Validasi Timestamp dengan Toleransi Waktu Sempit**:  
   Klien mengirimkan waktu request (`X-Timestamp: 1700000000`). Server menolak permintaan jika selisih waktu klien dan server melebihi toleransi wajar (misal toleransi maksimal 30–60 detik).
3. **Gunakan Nonce Kriptografis Sekali Pakai**:  
   Server menerbitkan nonce acak. Setiap request wajib menyertakan nonce tersebut. Begitu nonce diterima, server langsung menghapusnya dari daftar aktif.

---

## 10. Penanganan

1. **Implementasikan Caching Idempotensi di Redis**: Pasang middleware pencatat request ID di API gateway.
2. **Koreksi Transaksi Terduplikasi**: Identifikasi transaksi dengan payload identik dan batalkan transaksi ganda.
3. **Sinkronisasi Jam Server (NTP)**: Pastikan server menggunakan Network Time Protocol (NTP) yang akurat untuk validasi timestamp.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Idempotency Key Middleware dengan Redis)

```go
package main

import (
	"context"
	"net/http"
	"time"
	"github.com/go-redis/redis/v8"
)

var rdb = redis.NewClient(&redis.Options{Addr: "localhost:6379"})

// ✅ KODE AMAN: Middleware Pencegahan Replay Attack dengan Idempotency Key
func IdempotencyMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		idempotencyKey := r.Header.Get("Idempotency-Key")
		if idempotencyKey == "" {
			http.Error(w, "Header 'Idempotency-Key' wajib disertakan untuk transaksi ini", http.StatusBadRequest)
			return
		}

		ctx := context.Background()
		redisKey := "idempotency:" + idempotencyKey

		// Gunakan SETNX (Set if Not Exists) dengan masa aktif 24 jam
		// Mengembalikan true HANYA jika key belum pernah ada di Redis
		success, err := rdb.SetNX(ctx, redisKey, "PROCESSING", 24*time.Hour).Result()
		if err != nil || !success {
			// Jika key sudah ada, tolak keras pemutaran ulang (Replay Attack Ditolak!)
			http.Error(w, "Transaksi ini sudah pernah diproses sebelumnya (Duplikasi Ditolak)", http.StatusConflict) // 409 Conflict
			return
		}

		next(w, r)
	}
}

func ProcessPaymentHandler(w http.ResponseWriter, r *http.Request) {
	// Eksekusi pembayaran aman...
	w.Write([]byte("Pembayaran sukses diproses tepat satu kali"))
}
```

### B. Contoh dalam TypeScript (Validasi Timestamp & Toleransi Jeda Waktu)

```typescript
import { Request, Response, NextFunction } from 'express';

const MAX_ALLOWED_DRIFT_SECONDS = 60; // Toleransi waktu maksimal 60 detik

// ✅ KODE AMAN: Middleware Validasi Waktu Request untuk Mencegah Replay Attack
export function verifyTimestamp(req: Request, res: Response, next: NextFunction) {
  const requestTimestampHeader = req.headers['x-timestamp'];

  if (!requestTimestampHeader) {
    return res.status(400).json({ error: 'Header X-Timestamp wajib disertakan' });
  }

  const requestTime = parseInt(requestTimestampHeader as string, 10);
  const currentTime = Math.floor(Date.now() / 1000);

  // Periksa selisih waktu antara jam server dan jam pengiriman request klien
  const timeDifference = Math.abs(currentTime - requestTime);

  if (timeDifference > MAX_ALLOWED_DRIFT_SECONDS) {
    return res.status(400).json({
      error: 'Permintaan ditolak: Timestamp kadaluwarsa atau selisih waktu terlalu jauh (Potensi Replay Attack)',
    });
  }

  next();
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Replay dengan cURL
1. Kirim transaksi pertama dengan Idempotency Key acak:
   ```bash
   curl -i -X POST http://localhost:8080/api/transfer \
     -H "Idempotency-Key: TX-991283" \
     -d '{"amount": 50000}'
   ```
   *Respon: `200 OK - Pembayaran sukses diproses`*
2. Kirim ulang perintah yang persis sama untuk kedua kalinya dalam hitungan detik:
   ```bash
   curl -i -X POST http://localhost:8080/api/transfer \
     -H "Idempotency-Key: TX-991283" \
     -d '{"amount": 50000}'
   ```
- **Kondisi Rentan**: Respon kedua mengembalikan `200 OK` dan saldo berkurang dua kali.
- **Kondisi Aman**: Respon kedua mengembalikan status `409 Conflict` ("Transaksi ini sudah pernah diproses sebelumnya") dan saldo hanya terpotong satu kali.

---

## Referensi

- [IETF RFC Draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [Stripe API Documentation: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [MITRE CWE-294: Authentication Bypass by Capture-replay](https://cwe.mitre.org/data/definitions/294.html)
