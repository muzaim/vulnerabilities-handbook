# JWT Vulnerabilities (JSON Web Token)

> **Metadata Keamanan:**
> - **CWE**: [CWE-345: Insufficient Verification of Data Authenticity](https://cwe.mitre.org/data/definitions/345.html) & [CWE-347: Improper Verification of Cryptographic Signature](https://cwe.mitre.org/data/definitions/347.html)
> - **OWASP**: [A02:2021 – Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/) / [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.8)

---

## 1. Pengertian

**JWT (JSON Web Token) Vulnerabilities** adalah kumpulan celah keamanan yang muncul akibat implementasi atau verifikasi token JWT yang salah di sisi backend, sehingga penyerang dapat memalsukan isi klaim token (seperti mengubah ID pengguna atau peran menjadi admin) tanpa ditolak oleh server.

**Analogi Sederhana:**  
Bayangkan surat izin berkendara (SIM) yang dicetak di atas kertas dan ditandatangani oleh kepala kepolisian dengan stempel khusus tahan air.  
- **Token Valid**: Polisi memeriksa tanda tangan dan stempel untuk memastikan SIM tidak diedit.
- **Celah Algoritma 'None'**: Polisi menerima surat ketikan biasa tanpa tanda tangan atau stempel sama sekali ("Pak, ini jenis surat tanpa stempel!"), dan polisi mempercayainya begitu saja.
- **Kunci Lemah**: Stempel polisi dipalsukan dengan sangat mudah karena bentuknya hanya lingkaran kosong tanpa pengaman.

---

## 2. Cara Kerja

JWT terdiri dari tiga bagian yang dipisahkan oleh tanda titik (`.`):  
`HEADER.PAYLOAD.SIGNATURE`

1. **Header**: Menentukan jenis token dan algoritma penandatanganan (misal: `{"alg": "HS256", "typ": "JWT"}`).
2. **Payload**: Berisi data klaim pengguna (misal: `{"sub": "123", "role": "user"}`).
3. **Signature**: Tanda tangan kriptografis yang dibuat dengan menggabungkan Header + Payload + Kunci Rahasia Server.

### Jenis Celah Utama pada JWT:
- **Algoritma `none` (Unsigned Tokens)**: Beberapa pustaka JWT lama secara default mengizinkan algoritma `"alg": "none"`. Penyerang cukup menghapus tanda tangan dan mengubah isi payload. Server yang menerima token tanpa verifikasi tanda tangan akan langsung mempercayai isi payload.
- **Kunci Rahasia Lemah (Weak HMAC Secret)**: Menggunakan kata rahasia yang pendek atau mudah ditebak seperti `secret`, `123456`, atau nama perusahaan. Penyerang dapat melakukan brute force offline untuk menemukan kunci rahasia, lalu menandatangani token palsu apa pun.
- **Key Confusion (Algoritma Asimetris vs Simetris)**: Server menggunakan RSA (kunci publik/privat), tetapi library backend salah mengizinkan algoritma HMAC (HS256) menggunakan kunci publik (yang dapat diakses siapa saja) sebagai kunci simetris untuk memverifikasi tanda tangan.
- **Tidak Memvalidasi Masa Kadaluwarsa (`exp`)**: Server menerima token yang masa berlakunya sudah habis bertahun-tahun lalu.

---

## 3. Penyebab

- Mengizinkan library menerima sembarang algoritma dari header tanpa pembatasan (*algorithm whitelisting*).
- Menyetel parameter `verify: false` atau mengabaikan penanganan error tanda tangan (*signature verification error ignored*).
- Menyimpan secret key di kode sumber atau menggunakan string rahasia yang terlalu pendek.
- Tidak memiliki mekanisme pencabutan (*token revocation/blacklist*) untuk token yang dicuri sebelum masa kadaluwarsa habis.

---

## 4. Skenario Serangan

1. **Dekode Token**: Penyerang login sebagai pengguna biasa dan menerima token JWT. Penyerang mendekode token di `jwt.io`.
2. **Manipulasi Header & Payload**:
   - Header diubah: `{"alg": "none"}`
   - Payload diubah: `{"sub": "99", "role": "admin"}`
3. **Penghapusan Signature**: Penyerang menghapus bagian signature ketiga, menyisakan string: `eyJhbGciOiJub25lIn0.eyJzdWIiOiI5OSIsInJvbGUiOiJhZG1pbiJ9.` (tetap diakhiri tanda titik).
4. **Bypass Autentikasi**: Penyerang mengirim token palsu ini ke endpoint admin: `GET /api/admin/system`. Server yang rentan memproses klaim admin dan memberikan akses penuh.

---

## 5. Dampak

- **Bypass Autentikasi Total**: Penyerang dapat login sebagai siapa saja (termasuk akun administrator atau CEO).
- **Pemalsuan Hak Akses**: Menaikkan hak akses akun sendiri secara instan (*Privilege Escalation*).
- **Pencurian Data Sensitif**: Mengakses seluruh resource API privat yang dilindungi oleh token.

---

## 6. Bagian Sistem yang Terdampak

- Middleware verifikasi JWT API Gateway atau backend microservice.
- Mekanisme Single Sign-On (SSO) berbasis OAuth / OpenID Connect.
- Endpoint refresh token dan sesi otentikasi seluler.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem FinTech DompetKilat:**  
DompetKilat menggunakan arsitektur microservices. Service autentikasi mengeluarkan token JWT yang ditandatangani dengan algoritma HS256. Namun, developer junior menggunakan kata sandi rahasia:
`JWT_SECRET="dompetkilat"`

Penyerang mengunduh alat bantu pemecah hash offline (*hashcat / jwt-cracker*) menggunakan wordlist rockyou.txt. Dalam waktu kurang dari 2 detik, kunci rahasia `"dompetkilat"` berhasil ditemukan. Penyerang kemudian membuat token baru atas nama ID nasabah korporat dengan saldo miliaran rupiah dan menandatanganinya menggunakan kunci tersebut. Penyerang berhasil mencairkan saldo korban tanpa pernah mengetahui password aslinya.

---

## 8. Cara Mendeteksi

- **Uji Coba Algoritma `none`**: Kirim token dengan `"alg": "none"` dan signature kosong. Jika server merespons 200 OK, sistem rentan.
- **Audit Kekuatan Kunci Rahasia**: Periksa apakah panjang kunci secret HMAC minimal 256-bit (32 karakter acak dengan entropi tinggi).
- **SAST**: Periksa apakah ada konfigurasi pustaka JWT yang mengabaikan verifikasi tanda tangan (misal: `jwt.decode` alih-alih `jwt.verify`).

---

## 9. Pencegahan

1. **Wajibkan Algoritma Eksplisit (Algorithm Whitelist)**: Jangan biarkan header token menentukan algoritma. Di kode backend, secara tegas tentukan algoritma yang diizinkan (misal hanya `algorithms: ['HS256']` atau `['RS256']`).
2. **Tolak Tegas Algoritma `none`**: Pastikan library JWT menganggap algoritma `none` sebagai token tidak sah secara default.
3. **Gunakan Secret Key yang Kuat**: Gunakan kunci acak kriptografis berkekuatan minimal 256-bit yang disimpan aman di environment variable (bukan hardcoded).
4. **Validasi Wajib Klaim `exp` (Expiry) dan `iss` (Issuer)**: Pastikan token memiliki batas waktu kadaluwarsa yang pendek (misal 15 menit) dan gunakan refresh token yang tersimpan di server untuk pembaruan.

---

## 10. Penanganan

1. **Rotasi Kunci Rahasia Segera**: Ganti `JWT_SECRET` di server produksi dengan string acak baru. Seluruh token yang beredar otomatis tidak berlaku lagi.
2. **Pasang Validasi Algoritma Ketat**: Perbarui kode middleware otentikasi agar membatasi algoritma verifikasi secara eksplisit.
3. **Audit Token Aktif**: Terapkan revocation list di Redis untuk mencabut token yang dicurigai disalahgunakan.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (`golang-jwt/jwt/v5`)

```go
package main

import (
	"errors"
	"fmt"
	"github.com/golang-jwt/jwt/v5"
	"time"
)

var mySecretKey = []byte("kunci_rahasia_sangat_kuat_dan_panjang_minimal_32_karakter_acak!")

type CustomClaims struct {
	UserID string `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

// ❌ KODE RENTAN (Vulnerable)
// Menerima algoritma apa saja dari token tanpa validasi metode penandatanganan
func ParseTokenVulnerable(tokenString string) (*CustomClaims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (interface{}, error) {
		// RENTAN: Jika token header adalah "alg": "none", fungsi ini tetap mengembalikan secret atau nil
		// sehingga tanda tangan palsu/kosong bisa lolos!
		return mySecretKey, nil
	})

	if claims, ok := token.Claims.(*CustomClaims); ok && token.Valid {
		return claims, nil
	}
	return nil, err
}

// ✅ KODE AMAN (Secure)
// Memverifikasi secara tegas bahwa algoritma penandatanganan WAJIB HMAC SHA256
func ParseTokenSecure(tokenString string) (*CustomClaims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (interface{}, error) {
		// Validasi eksplisit: Tolak jika bukan HMAC (misal penyerang kirim 'none' atau 'RS256')
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("algoritma tidak valid: %v", token.Header["alg"])
		}
		return mySecretKey, nil
	}, jwt.WithValidMethods([]string{"HS256"})) // Hanya izinkan HS256

	if err != nil {
		return nil, errors.New("token tidak valid atau kadaluwarsa")
	}

	if claims, ok := token.Claims.(*CustomClaims); ok && token.Valid {
		return claims, nil
	}
	return nil, errors.New("klaim token tidak sah")
}
```

### B. Contoh dalam TypeScript (Node.js & `jsonwebtoken`)

```typescript
import jwt from 'jsonwebtoken';

const SECRET_KEY = process.env.JWT_SECRET || 'kunci_rahasia_sangat_panjang_dan_acak_32_byte_entropy';

// ❌ KODE RENTAN (Vulnerable)
// 1. Mengabaikan daftar algoritma (membiarkan header menentukan algoritma)
// 2. Atau menggunakan jwt.decode() alih-alih jwt.verify()
export function verifyTokenVulnerable(token: string) {
  // RENTAN: jwt.decode hanya membaca payload tanpa memeriksa tanda tangan kriptografi!
  const decoded = jwt.decode(token);
  return decoded;
}

// ✅ KODE AMAN (Secure)
// 1. Menggunakan jwt.verify()
// 2. Menentukan whitelist algoritma secara eksplisit
export function verifyTokenSecure(token: string) {
  try {
    const verified = jwt.verify(token, SECRET_KEY, {
      algorithms: ['HS256'],   // WAJIB: Menolak keras algoritma 'none' atau key confusion
      ignoreExpiration: false, // WAJIB: Tolak token yang sudah kadaluwarsa
      maxAge: '1h',            // Batasi masa aktif token
    });
    return verified;
  } catch (error) {
    throw new Error('Token otentikasi tidak valid atau telah kadaluwarsa');
  }
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Token Palsu dengan Algoritma None
Buat token tanpa tanda tangan di script lokal:
```javascript
const fakeHeader = Buffer.from(JSON.stringify({ alg: "none", typ: "JWT" })).toString('base64url');
const fakePayload = Buffer.from(JSON.stringify({ user_id: "1", role: "admin" })).toString('base64url');
const unsignedToken = `${fakeHeader}.${fakePayload}.`;
```
Kirimkan token ini ke API backend:
```bash
curl -H "Authorization: Bearer <unsignedToken>" http://localhost:8080/api/admin/users
```
- **Kondisi Rentan**: API mengembalikan data pengguna admin (Status 200 OK).
- **Kondisi Aman**: API mengembalikan status `401 Unauthorized` dengan error `algoritma tidak valid`.

---

## Referensi

- [OWASP JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [PortSwigger: JWT Attacks](https://portswigger.net/web-security/jwt)
- [RFC 7519: JSON Web Token (JWT) Standard](https://datatracker.ietf.org/doc/html/rfc7519)
