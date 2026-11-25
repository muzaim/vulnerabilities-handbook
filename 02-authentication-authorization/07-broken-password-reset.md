# Broken Password Reset (Celah Pemulihan Kata Sandi)

> **Metadata Keamanan:**
> - **CWE**: [CWE-640: Weak Password Recovery Mechanism for Forgotten Password](https://cwe.mitre.org/data/definitions/640.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.8)

---

## 1. Pengertian

**Broken Password Reset** adalah kelemahan dalam alur logika fitur "Lupa Kata Sandi" (*Forgot Password*) yang memungkinkan pihak ketiga menyetel ulang kata sandi milik korban tanpa izin, berujung pada pengambilalihan akun secara penuh (*Account Takeover*).

**Analogi Sederhana:**  
Bayangkan Anda lupa kunci gembok loker di tempat gym. Anda meminta petugas membuka gembok tersebut.  
Petugas: "Berapa nomor loker Anda?"  
Anda: "Nomor 50."  
Petugas langsung memotong gembok nomor 50 dan memberikan isi loker kepada Anda tanpa meminta Anda menunjukkan kartu anggota gym atau kartu identitas sama sekali. Siapa pun yang mengetahui nomor loker orang lain bisa mengambil alih loker tersebut dengan sangat mudah.

---

## 2. Cara Kerja

Alur pemulihan kata sandi yang aman seharusnya memenuhi prinsip-prinsip:
1. Token reset harus dibuat acak kriptografis dengan entropi tinggi.
2. Token hanya boleh dikirim ke saluran terverifikasi milik pengguna (email/SMS terdaftar).
3. Token harus bersifat sekali pakai (*single-use*) dan memiliki masa berlaku singkat (misal 15 menit).
4. Saat password baru disimpan, seluruh sesi login yang ada harus dibatalkan.

Celah broken password reset terjadi ketika:
- **Token Mudah Ditebak**: Token dibuat dari hash timestamp waktu saat ini (`md5(timestamp)`), nomor urut, atau kode OTP 4 digit tanpa batas percobaan tebakan.
- **Host Header Injection**: Tautan reset password di email dibentuk menggunakan header `Host` dari request HTTP (`https://` + `req.headers.host` + `/reset?token=...`). Penyerang dapat memanipulasi header tersebut agar tautan reset dikirimkan ke server milik penyerang.
- **Token Tidak Hangus**: Token reset tetap bisa digunakan berulang kali meskipun kata sandi sudah diubah.
- **Kebocoran Token di Referer Header**: Tautan reset memuat aset pihak ketiga (seperti widget analytics), sehingga token di URL bocor melalui HTTP header `Referer`.

---

## 3. Penyebab

- **Penggunaan Generator Angka/String Lemah**: Menggunakan fungsi acak standar (seperti `Math.random()` di JS atau `rand.Int()` di Go) alih-alih generator acak aman (*CSPRNG*).
- **Ketiadaan Validasi Host Header**: Mempercayai header HTTP `Host` mentah untuk membangun tautan URL absolut di badan email.
- **Ketiadaan Status 'Used' pada Token**: Tidak menandai token sebagai *expired/used* di database setelah password berhasil diperbarui.
- **Ketiadaan Rate Limiting pada Verifikasi Kode OTP**: Membiarkan penyerang mencoba menebak kode OTP 4-6 digit secara otomatis.

---

## 4. Skenario Serangan

1. **Permintaan Reset dengan Header Berbahaya**: Penyerang mengirimkan permintaan lupa password untuk email korban `korban@example.com`, tetapi mengubah header:
   ```http
   POST /api/forgot-password HTTP/1.1
   Host: penyerang-evil.com
   Content-Type: application/json

   { "email": "korban@example.com" }
   ```
2. **Pengiriman Email ke Korban**: Server aplikasi mengirim email resmi ke korban:  
   *"Klik tautan ini untuk mereset kata sandi Anda: `https://penyerang-evil.com/reset-password?token=RAHASIA_123`"*.
3. **Klik Tautan oleh Korban**: Korban yang mengira email itu resmi mengklik tautan tersebut.
4. **Pencurian Token**: Browser korban membuka server `penyerang-evil.com`, sehingga token `RAHASIA_123` tercatat di log server penyerang.
5. **Pengambilalihan Akun**: Penyerang menggunakan token tersebut di website asli untuk mengganti password korban.

---

## 5. Dampak

- **Account Takeover (ATO) Penuh**: Penyerang menguasai akun pengguna target secara permanen.
- **Bypass 2FA**: Pada beberapa implementasi yang buruk, proses reset password otomatis mematikan proteksi 2FA korban.
- **Kerugian Finansial & Data**: Akses langsung ke data sensitif dan saldo akun pengguna.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint permintaan reset kata sandi (`/forgot-password`).
- Endpoint konfirmasi kata sandi baru (`/reset-password`).
- Modul pengirim email / SMS gateway transaksional.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem E-Commerce BelanjaCepat:**  
BelanjaCepat memiliki fitur reset password menggunakan kode PIN 4 digit angka yang dikirim ke SMS.

Developer tidak memasang rate-limiting pada endpoint verifikasi PIN:
`POST /api/auth/verify-reset-pin` dengan body `{"email": "korban@mail.com", "pin": "1234"}`.

Karena PIN 4 digit hanya memiliki 10.000 kemungkinan kombinasi (`0000` s.d. `9999`), penyerang meminta reset password untuk email target, lalu menjalankan skrip untuk mencoba seluruh 10.000 kemungkinan kombinasi dalam waktu kurang dari 30 detik. Sistem merespons sukses saat kode yang tepat tercapai, dan penyerang langsung memasukkan password baru untuk akun korban.

---

## 8. Cara Mendeteksi

- **Uji Manipulasi Host Header**: Kirim request reset password dengan header `Host: attacker.com` atau `X-Forwarded-Host: attacker.com`. Periksa apakah URL di email yang masuk mengarah ke domain penyerang.
- **Uji Brute Force Kode Reset**: Coba kirimkan kode verifikasi yang salah lebih dari 5 kali berturut-turut. Jika sistem tidak memblokir atau menolak permintaan, sistem rentan.
- **Uji Penggunaan Berulang (Replay Test)**: Gunakan tautan token yang sama untuk mengganti password kedua kalinya. Jika berhasil, token tidak sekali pakai.

---

## 9. Pencegahan

1. **Gunakan CSPRNG dengan Entropi Tinggi**: Buat token acak kriptografis sepanjang minimal 32 byte (256 bit) acak.
2. **Gunakan Domain Statis yang Dikonfigurasi di Server (Whitelist)**: Jangan pernah menggunakan `req.headers.host` untuk merangkai tautan email. Gunakan konstanta konfigurasi server: `process.env.APP_BASE_URL`.
3. **Wajibkan Karakteristik Sekali Pakai (Single-Use Token)**: Simpan status token di database; begitu digunakan, ubah status menjadi `is_used = true` atau langsung hapus token.
4. **Masa Berlaku Sangat Singkat**: Batasi masa berlaku token maksimal 10-15 menit.
5. **Putuskan Seluruh Sesi Aktif Lainnya**: Saat password berhasil diperbarui, batalkan (*revoke*) seluruh sesi dan refresh token yang sedang aktif di perangkat lain.

---

## 10. Penanganan

1. **Batalkan Seluruh Token Reset yang Masih Aktif**: Hapus seluruh tabel token reset yang belum terpakai.
2. **Kunci Endpoint Reset Sementara**: Jika terdeteksi brute force masif pada kode OTP, pasang proteksi Captcha dan Rate Limiting ketat.
3. **Perbaiki Kode Pembuatan URL Email**: Ganti pembentukan URL dinamis dengan konfigurasi basis domain tetap dari environment server.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"os"
	"time"
)

type PasswordResetToken struct {
	HashedToken string
	Email       string
	ExpiresAt   time.Time
	IsUsed      bool
}

// ✅ KODE AMAN: Menggunakan CSPRNG kriptografis dan menyimpan hash token di DB
func GenerateResetToken(email string) (string, *PasswordResetToken, error) {
	// 1. Buat token acak 32 bytes menggunakan crypto/rand (CSPRNG)
	rawBytes := make([]byte, 32)
	if _, err := rand.Read(rawBytes); err != nil {
		return "", nil, err
	}
	rawToken := hex.EncodeToString(rawBytes)

	// 2. Simpan HASH token di database (bukan token mentah, agar aman jika DB bocor)
	hash := sha256.Sum256([]byte(rawToken))
	hashedToken := hex.EncodeToString(hash[:])

	record := &PasswordResetToken{
		HashedToken: hashedToken,
		Email:       email,
		ExpiresAt:   time.Now().Add(15 * time.Minute), // Kadaluwarsa 15 menit
		IsUsed:      false,
	}

	// 3. Bangun tautan email menggunakan environment variabel tetap, BUKAN dari HTTP Request header!
	appURL := os.Getenv("APP_BASE_URL") // Misal: "https://toko.example.com"
	resetLink := appURL + "/reset-password?token=" + rawToken

	_ = resetLink // Kirimkan link ini ke email pengguna
	return rawToken, record, nil
}

// Verifikasi dan penggunaan token secara aman
func ConsumeResetToken(record *PasswordResetToken, rawTokenInput string) error {
	if record.IsUsed {
		return errors.New("token pemulihan ini sudah pernah digunakan")
	}

	if time.Now().After(record.ExpiresAt) {
		return errors.New("token pemulihan telah kadaluwarsa")
	}

	hash := sha256.Sum256([]byte(rawTokenInput))
	if hex.EncodeToString(hash[:]) != record.HashedToken {
		return errors.New("token tidak valid")
	}

	// Tandai token langsung menjadi USED
	record.IsUsed = true
	return nil
}
```

### B. Contoh dalam TypeScript (Node.js & Express)

```typescript
import crypto from 'crypto';
import { Request, Response } from 'express';

// ❌ KODE RENTAN (Host Header Injection)
export async function sendResetEmailVulnerable(req: Request, res: Response) {
  const token = Math.random().toString(36).substring(2); // RENTAN: Math.random() mudah diprediksi!
  
  // RENTAN: Mengambil domain dari header Host request penyerang
  const host = req.headers.host; 
  const resetLink = `https://${host}/reset-password?token=${token}`;

  // Kirim email dengan tautan yang mengarah ke server penyerang jika Host diubah
  return res.json({ message: 'Tautan pemulihan telah dikirim' });
}

// ✅ KODE AMAN (Secure)
export async function sendResetEmailSecure(req: Request, res: Response) {
  const email = req.body.email;
  const user = await findUser(email);

  if (user) {
    // 1. Buat token berkekuatan kriptografis tinggi
    const rawToken = crypto.randomBytes(32).toString('hex');
    const tokenHash = crypto.createHash('sha256').update(rawToken).digest('hex');

    // 2. Simpan hash token dengan masa aktif 15 menit
    await savePasswordResetToken({
      userId: user.id,
      tokenHash,
      expiresAt: new Date(Date.now() + 15 * 60 * 1000),
      used: false,
    });

    // 3. Gunakan BASE URL statis dari environment variable
    const baseUrl = process.env.PUBLIC_APP_URL || 'https://aplikasiku.example.com';
    const safeResetLink = `${baseUrl}/auth/reset-password?token=${rawToken}`;

    await sendEmail(user.email, safeResetLink);
  }

  // Selalu berikan respons generik baik email ada maupun tidak (mencegah enumerasi)
  return res.json({ message: 'Jika email terdaftar, petunjuk pemulihan telah dikirimkan ke kotak masuk Anda.' });
}

async function findUser(e: string): Promise<any> { return null; }
async function savePasswordResetToken(d: any) {}
async function sendEmail(to: string, link: string) {}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Host Header Injection
Kirimkan permintaan pemulihan kata sandi dengan memalsukan header:
```bash
curl -X POST http://localhost:8080/api/forgot-password \
  -H "Host: evil-attacker.com" \
  -H "Content-Type: application/json" \
  -d '{"email": "tester@example.com"}'
```
- **Verifikasi**: Buka email uji coba yang masuk. Pastikan tautan yang tercantum **tetap mengarah ke domain resmi aplikasi Anda**, bukan ke `evil-attacker.com`.

---

## Referensi

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [PortSwigger: Password Reset Poisoning](https://portswigger.net/web-security/host-header/exploiting/password-reset-poisoning)
- [MITRE CWE-640](https://cwe.mitre.org/data/definitions/640.html)
