# Broken Authentication

> **Metadata Keamanan:**
> - **CWE**: [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.8)

---

## 1. Pengertian

**Broken Authentication** (Kegagalan Autentikasi) adalah kondisi di mana sistem salah mengidentifikasi siapa pengguna yang sedang login, atau membiarkan celah dalam proses verifikasi kredensial sehingga penyerang dapat menyamar sebagai pengguna sah.

**Analogi Sederhana:**  
Bayangkan kunci kamar hotel. Petugas resepsionis memberikan kartu kunci kamar dengan nomor kamar yang ditulis menggunakan pensil di kartu tersebut.  
Jika tamu dapat menghapus tulisan nomor kamar tersebut dan menggantinya dengan nomor kamar VIP mana pun lalu pintu kamar VIP terbuka begitu saja, maka sistem kunci hotel tersebut mengalami kegagalan autentikasi.

---

## 2. Cara Kerja

Autentikasi mencakup siklus hidup verifikasi identitas:
1. Pendaftaran akun (*Registration*)
2. Verifikasi saat masuk (*Login*)
3. Pemeliharaan sesi aktif (*Session maintenance*)
4. Pemulihan akun (*Password reset*)
5. Penutupan sesi (*Logout*)

Celah broken authentication muncul ketika ada kelemahan pada salah satu mata rantai di atas:
- Sistem tidak membatasi tebakan password (memungkinkan brute force).
- ID sesi dapat ditebak atau dibuat berurutan (misal: ID sesi = 1001, 1002, 1003).
- Token sesi tidak pernah kadaluwarsa (*never expires*).
- Server tidak memvalidasi ulang token saat pengguna mengganti email/password.

---

## 3. Penyebab

- **Ketiadaan Rate Limiting**: Membiarkan penyerang mencoba ratusan kombinasi password per detik tanpa pemblokiran IP atau akun.
- **Pesan Galat Terlalu Rinci (Username Enumeration)**: Menampilkan pesan "Username tidak ditemukan" vs "Password salah", yang membocorkan kepada penyerang akun mana yang terdaftar.
- **Pengelolaan Sesi yang Ceroboh**: ID sesi dipaparkan di URL parameter atau tidak dimusnahkan di server saat pengguna menekan tombol "Logout".
- **Ketiadaan Multi-Factor Authentication (MFA)**: Bergantung semata-mata pada satu kata sandi tanpa lapisan keamanan kedua.

---

## 4. Skenario Serangan

1. **Enumerasi Akun**: Penyerang memasukkan email acak di form login. Respon "Email ini belum terdaftar" membantu penyerang menyaring daftar email pengguna yang valid.
2. **Uji Coba Password Lemah**: Terhadap email yang valid, penyerang mencoba password umum seperti `Password123`, `Admin2024!`, atau tanggal lahir.
3. **Penyusupan Akun**: Karena server tidak memblokir percobaan login berulang, penyerang berhasil masuk ke akun sasaran.
4. **Pemasangan Sesi Permanen**: Penyerang mengubah alamat email dan membiarkan sesi aktif tetap berjalan tanpa diputus oleh sistem.

---

## 5. Dampak

- **Pengambilalihan Akun Total (Account Takeover - ATO)**: Penyerang menguasai data pribadi, riwayat transaksi, dan wewenang pengguna.
- **Pencurian Dana & Aset**: Pada aplikasi perbankan atau e-wallet, saldo pengguna dapat dipindahkan secara ilegal.
- **Pelanggaran Kepatuhan Regulasi**: Perusahaan dapat dikenakan sanksi denda berat oleh regulator karena gagal melindungi data identitas nasabah.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint login (`/api/auth/login`).
- Endpoint penggantian email dan kata sandi (`/api/user/change-password`).
- Mekanisme logout dan manajemen refresh token.
- Integrasi Single Sign-On (SSO) atau OAuth callback.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Mobile Banking BankNusantara:**  
BankNusantara merilis fitur login API baru untuk aplikasi mobile. Endpoint login menerima nomor rekening dan PIN 6 digit angka:
`POST /api/v1/auth/mobile-pin`

Developer lupa memasang pembatas frekuensi (*rate limiting*) pada endpoint ini karena mengira aplikasi mobile tidak bisa diserang seperti browser.

Penyerang menulis skrip otomatis sederhana untuk mencoba PIN dari `000000` sampai `999999` (total 1 juta kemungkinan). Dalam waktu 4 jam menggunakan ribuan koneksi paralel, skrip berhasil menebak PIN milik seorang nasabah prioritas (`180792` - tanggal lahir nasabah). Penyerang berhasil menguras rekening korban.

---

## 8. Cara Mendeteksi

- **Pemeriksaan Respon Login**: Cek apakah respon sistem membedakan antara "Email tidak terdaftar" dan "Password salah".
- **Pengujian Batas Percobaan (Brute Force Test)**: Coba lakukan login gagal 10 kali berturut-turut. Jika akun tidak terkunci atau tidak ada jeda waktu (CAPTCHA/delay), sistem rentan.
- **Inspeksi Siklus Sesi**: Login, salin token sesi, lakukan logout di aplikasi. Jika token lama masih bisa digunakan untuk memanggil API profil, manajemen sesi rusak.

---

## 9. Pencegahan

1. **Terapkan Rate Limiting & Account Lockout**: Batasi percobaan login (misal maksimal 5 kali gagal per 15 menit per IP/akun).
2. **Gunakan Pesan Galat Generik**: Selalu gunakan kalimat seragam seperti: *"Username atau kata sandi yang Anda masukkan salah"* untuk mencegah enumerasi username.
3. **Wajibkan Multi-Factor Authentication (MFA/2FA)**: Gunakan OTP berbasis TOTP (Google Authenticator) atau push notification untuk aksi sensitif.
4. **Invaliasi Sesi di Sisi Server Saat Logout**: Jangan hanya menghapus token di peramban pengguna; cabut dan masukkan token ke daftar hitam (*blacklist/revocation store*) di database/Redis.

---

## 10. Penanganan

1. **Paksa Reset Sesi Massal**: Putuskan seluruh sesi pengguna aktif di server (*revoke all active tokens*).
2. **Aktifkan Rate Limiting Segera**: Pasang middleware rate-limiter di API gateway atau load balancer (seperti NGINX / Cloudflare).
3. **Kunci Akun yang Mencurigakan**: Identifikasi akun-akun yang mengalami lonjakan percobaan login gagal dan kirimkan tautan verifikasi ulang ke email resmi pemilik akun.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"crypto/subtle"
	"errors"
	"sync"
	"time"
	"golang.org/x/crypto/bcrypt"
)

// Struktur sederhana pelacak percobaan gagal (In-Memory Rate Limiter)
type RateLimiter struct {
	mu       sync.Mutex
	attempts map[string]int
	lockTime map[string]time.Time
}

var limiter = &RateLimiter{
	attempts: make(map[string]int),
	lockTime: make(map[string]time.Time),
}

// ❌ KODE RENTAN (Vulnerable)
// 1. Membocorkan apakah email terdaftar atau tidak
// 2. Tidak ada pembatasan jumlah percobaan login
func LoginVulnerable(email, password string, getUserFromDB func(string) (string, error)) error {
	storedHash, err := getUserFromDB(email)
	if err != nil {
		// RENTAN: Memberi tahu penyerang bahwa email ini tidak ada
		return errors.New("email tidak terdaftar")
	}

	if bcrypt.CompareHashAndPassword([]byte(storedHash), []byte(password)) != nil {
		return errors.New("password salah")
	}

	return nil
}

// ✅ KODE AMAN (Secure)
// 1. Pesan galat generik dan waktu respon konsisten
// 2. Dilengkapi proteksi penguncian sementara akun
func LoginSecure(email, password string, getUserFromDB func(string) (string, error)) error {
	limiter.mu.Lock()
	if lockUntil, ok := limiter.lockTime[email]; ok && time.Now().Before(lockUntil) {
		limiter.mu.Unlock()
		return errors.New("terlalu banyak percobaan gagal, silakan coba lagi dalam 15 menit")
	}
	limiter.mu.Unlock()

	storedHash, err := getUserFromDB(email)
	var dummyHash = "$2a$12$e8N8Vw9m6D3y.FvQ5wZzpe6k1L1H3Z4mO7lK8j9k0l1m2n3o4p5q" // Dummy hash untuk waktu komputasi konstan

	var isSuccess bool
	if err != nil {
		// Jalankan komputasi bcrypt palsu agar durasi respons sama persis
		bcrypt.CompareHashAndPassword([]byte(dummyHash), []byte(password))
		isSuccess = false
	} else {
		isSuccess = bcrypt.CompareHashAndPassword([]byte(storedHash), []byte(password)) == nil
	}

	if !isSuccess {
		limiter.mu.Lock()
		limiter.attempts[email]++
		if limiter.attempts[email] >= 5 {
			limiter.lockTime[email] = time.Now().Add(15 * time.Minute)
			limiter.attempts[email] = 0
		}
		limiter.mu.Unlock()

		// Selalu gunakan pesan seragam!
		return errors.New("email atau kata sandi yang Anda masukkan salah")
	}

	// Reset percobaan gagal jika login berhasil
	limiter.mu.Lock()
	delete(limiter.attempts, email)
	delete(limiter.lockTime, email)
	limiter.mu.Unlock()

	return nil
}
```

### B. Contoh dalam TypeScript (Express & Rate Limiting)

```typescript
import { Request, Response } from 'express';
import bcrypt from 'bcrypt';
import rateLimit from 'express-rate-limit';

// ✅ Rate Limiter Middleware: Batasi maksimal 5 request per 15 menit per IP
export const authRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { message: 'Terlalu banyak percobaan login. Silakan coba lagi 15 menit kemudian.' },
  standardHeaders: true,
  legacyHeaders: false,
});

// ❌ KODE RENTAN (Vulnerable Handler)
export async function loginVulnerable(req: Request, res: Response) {
  const { email, password } = req.body;
  const user = await findUserByEmail(email);

  if (!user) {
    // RENTAN: Membocorkan bahwa email tidak terdaftar
    return res.status(404).json({ error: 'Email tidak ditemukan' });
  }

  const isValid = await bcrypt.compare(password, user.passwordHash);
  if (!isValid) {
    return res.status(401).json({ error: 'Kata sandi salah' });
  }

  return res.json({ token: generateSessionToken(user.id) });
}

// ✅ KODE AMAN (Secure Handler)
export async function loginSecure(req: Request, res: Response) {
  const { email, password } = req.body;
  const user = await findUserByEmail(email);

  // Jika user tidak ada, tetap hitung hash palsu untuk mencegah timing attack
  const dummyHash = '$2b$10$wN3t6vB9jP4xY1zQ2uR4vOe...';
  const hashToCompare = user ? user.passwordHash : dummyHash;

  const isValid = await bcrypt.compare(password, hashToCompare);

  if (!user || !isValid) {
    // Pesan identik dengan status 401 unauthorizied generik
    return res.status(401).json({ error: 'Email atau kata sandi yang Anda masukkan salah' });
  }

  return res.json({ token: generateSessionToken(user.id) });
}

async function findUserByEmail(email: string): Promise<{ id: string; passwordHash: string } | null> {
  return null;
}
function generateSessionToken(id: string): string { return 'token_rahasia'; }
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Respon Waktu & Pesan Galat
1. Uji dengan email acak yang pasti tidak ada: `curl -d '{"email":"tidakada@test.com","password":"123"}' /api/login`
2. Uji dengan email pengguna yang sah tetapi password salah: `curl -d '{"email":"admin@test.com","password":"123"}' /api/login`
- **Verifikasi**: Keduanya harus mengembalikan status code (401) dan teks pesan yang identik seratus persen.

### Pengujian Batas Percobaan (Rate Limiting)
Kirimkan 6 permintaan login gagal secara berturut-turut dalam hitungan detik.
- **Kondisi Aman**: Pada permintaan ke-6, sistem mengembalikan status `429 Too Many Requests`.

---

## Referensi

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [MITRE CWE-287](https://cwe.mitre.org/data/definitions/287.html)
- [NIST Special Publication 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
