# Brute Force Attack

> **Metadata Keamanan:**
> - **CWE**: [CWE-307: Improper Restriction of Excessive Authentication Attempts](https://cwe.mitre.org/data/definitions/307.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 5.3 - 7.5)

---

## 1. Pengertian

**Brute Force Attack** (Serangan Tebakan Membabi Buta) adalah metode penyerangan coba-coba (*trial-and-error*) otomatis di mana penyerang secara sistematis mencoba ribuan atau jutaan kombinasi kata sandi, kode PIN, atau kunci enkripsi hingga menemukan kombinasi yang benar.

**Analogi Sederhana:**  
Bayangkan sebuah koper dengan gembok angka 3 digit (dari `000` hingga `999`).  
Jika pemilik koper tidak memasang alarm yang berbunyi saat salah 3 kali, seseorang yang berniat jahat dapat duduk tenang dan mencoba memutar angka satu per satu: `000`, `001`, `002`... Dalam waktu kurang dari 15 menit, koper tersebut pasti akan terbuka.

---

## 2. Cara Kerja

Komputer modern dapat mengirim ratusan atau ribuan permintaan HTTP per detik jika tidak ada pembatas di sisi server.

Penyerang menggunakan alat otomatisasi (seperti Hydra, Burp Suite Intruder, atau skrip Python khusus):
1. Mengidentifikasi akun target (misal: `admin@example.com`).
2. Menyiapkan kamus kata sandi populer (*wordlist* seperti `rockyou.txt` yang memuat jutaan kata sandi umum seperti `123456`, `admin123`, `qwerty`).
3. Mengirimkan request login secara paralel tanpa henti.
4. Memeriksa respons server: jika status berubah dari `401 Unauthorized` menjadi `200 OK`, penyerang telah menemukan password yang tepat.

---

## 3. Penyebab

- **Ketiadaan Rate Limiting**: Tidak ada pembatasan berapa kali sebuah IP atau akun boleh mencoba login dalam rentang waktu tertentu.
- **Ketiadaan CAPTCHA**: Tidak ada mekanisme verifikasi kemanusiaan yang mempersulit bot otomatis.
- **Kebijakan Kata Sandi yang Terlalu Lemah**: Mengizinkan pengguna membuat password yang sangat pendek atau umum (misal: password 4 karakter atau hanya angka).
- **Ketiadaan Multi-Factor Authentication (MFA)**: Mengandalkan satu-satunya benteng pertahanan pada kata sandi.

---

## 4. Skenario Serangan

1. **Targeting**: Penyerang menargetkan halaman admin portal berita `https://berita.example/wp-login.php`.
2. **Eksekusi Botnet**: Penyerang menjalankan alat otomatis yang mengirimkan 500 percobaan login per detik menggunakan daftar kata sandi terpopuler.
3. **Penyusupan Berhasil**: Pada percobaan ke-8.412, password `admin2023!` berhasil login.
4. **Pengambilalihan Portal**: Penyerang masuk ke dashboard dan mengubah isi berita portal.

---

## 5. Dampak

- **Account Takeover**: Akun pengguna atau admin berhasil dibajak.
- **Beban Server Berlebih (Denial of Service)**: Jutaan request login yang memicu komputasi hashing password (seperti bcrypt/argon2 yang berat) dapat membuat CPU server mencapai 100% dan melumpuhkan layanan bagi pengguna sah.
- **Kerugian Biaya Infrastruktur**: Lonjakan trafik bandwidth dan komputasi cloud yang membengkak.

---

## 6. Bagian Sistem yang Terdampak

- Formulir login pengguna dan admin.
- Endpoint verifikasi PIN / OTP SMS.
- Endpoint verifikasi kode kupon diskon atau voucher hadiah.
- Akses SSH, FTP, atau Database yang terekspos ke internet publik.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Dompet Digital PayFast:**  
PayFast mengizinkan otorisasi transfer saldo menggunakan PIN 6 digit angka. Saat pengguna salah memasukkan PIN, aplikasi menampilkan pesan "PIN salah" dan mengizinkan pengguna mencoba lagi seketika.

Seorang penyerang yang berhasil mencuri HP korban yang layarnya terkunci (tetapi aplikasinya belum logout) menggunakan skrip USB otomatis untuk menembakkan input PIN dari `000000` hingga `999999`. Karena tidak ada jeda waktu dan tidak ada penguncian akun setelah 3 kali salah, PIN berhasil ditebak dalam 12 menit dan saldo korban ditransfer keluar.

---

## 8. Cara Mendeteksi

- **Monitoring Log Server**: Amati lonjakan jumlah status HTTP `401 Unauthorized` yang tidak normal dari alamat IP tertentu.
- **Pemeriksaan Beban CPU**: Lonjakan penggunaan CPU yang tinggi pada proses backend yang menangani hashing autentikasi.
- **Peringatan Login Anomali**: Sistem SIEM mendeteksi ribuan request dalam beberapa menit menuju URL `/login`.

---

## 9. Pencegahan

1. **Terapkan Rate Limiting Ketat (Wajib)**: Batasi maksimal 5 percobaan login gagal per akun/IP dalam jendela waktu 15 menit.
2. **Kunci Akun Sementara (Account Lockout)**: Kunci akun selama 15–30 menit jika terjadi 5 kali kegagalan berturut-turut.
3. **Gunakan Tantangan Adaptif (CAPTCHA / Cloudflare Turnstile)**: Munculkan tantangan verifikasi bot jika terdeteksi 3 kali kegagalan login.
4. **Wajibkan Multi-Factor Authentication (MFA)**: Sekalipun password berhasil ditebak penyerang, mereka tetap tidak bisa masuk tanpa kode OTP dari perangkat fisik korban.
5. **Kebijakan Kata Sandi Kuat**: Wajibkan minimal 8-12 karakter dengan kombinasi huruf, angka, dan simbol.

---

## 10. Penanganan

1. **Blokir IP Penyerang di Tingkat Firewall**: Masukkan IP sumber serangan ke dalam daftar blokir WAF / iptables.
2. **Kunci Akun yang Menjadi Sasaran Serangan**: Kirimkan email peringatan ke pemilik akun asli bahwa akunnya sedang diserang, dan minta mereka mengubah password melalui alur pemulihan.
3. **Aktifkan Proteksi DDoS / Rate Limit Global**: Pasang batas frekuensi ketat di level reverse-proxy (Nginx / Caddy / Cloudflare).

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Rate Limiter dengan Exponential Backoff)

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
	"time"
)

type LoginTracker struct {
	mu           sync.Mutex
	failedCounts map[string]int
	blockedUntil map[string]time.Time
}

var tracker = &LoginTracker{
	failedCounts: make(map[string]int),
	blockedUntil: make(map[string]time.Time),
}

// ✅ KODE AMAN: Melacak percobaan gagal dan mengunci akun secara bertingkat
func AuthenticateWithRateLimit(username, password string, verifyCreds func(u, p string) bool) (bool, error) {
	tracker.mu.Lock()
	defer tracker.mu.Unlock()

	// 1. Cek apakah akun sedang dalam masa pemblokiran
	if unlockTime, blocked := tracker.blockedUntil[username]; blocked {
		if time.Now().Before(unlockTime) {
			remaining := time.Until(unlockTime).Round(time.Second)
			return false, fmt.Errorf("terlalu banyak percobaan gagal. Akun dikunci, coba lagi dalam %v", remaining)
		}
		// Masa blokir selesai
		delete(tracker.blockedUntil, username)
	}

	// 2. Verifikasi kredensial
	success := verifyCreds(username, password)
	if !success {
		tracker.failedCounts[username]++

		// Jika salah 5 kali berturut-turut, kunci akun selama 15 menit
		if tracker.failedCounts[username] >= 5 {
			tracker.blockedUntil[username] = time.Now().Add(15 * time.Minute)
			tracker.failedCounts[username] = 0
			return false, fmt.Errorf("akun dikunci selama 15 menit karena 5 kali percobaan gagal")
		}

		return false, fmt.Errorf("kredensial salah. Sisa percobaan: %d", 5-tracker.failedCounts[username])
	}

	// 3. Reset jika login berhasil
	delete(tracker.failedCounts, username)
	delete(tracker.blockedUntil, username)
	return true, nil
}
```

### B. Contoh dalam TypeScript (Express & Redis Rate Limiter)

```typescript
import { Request, Response } from 'express';
import rateLimit from 'express-rate-limit';

// ✅ KODE AMAN: Middleware Express Rate Limit
export const loginRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // Periode evaluasi: 15 menit
  max: 5,                   // Maksimal 5 permintaan per IP dalam 15 menit
  standardHeaders: true,    // Mengembalikan info batas di header `RateLimit-*`
  legacyHeaders: false,
  message: {
    status: 429,
    error: 'Terlalu banyak percobaan login dari IP ini. Silakan coba lagi setelah 15 menit.',
  },
});

// Pasang middleware langsung di route login:
// app.post('/api/auth/login', loginRateLimiter, handleLogin);
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Otomatis dengan Shell Loop
Jalankan loop cepat 10 permintaan login dengan kata sandi salah:
```bash
for i in {1..10}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/login \
    -H "Content-Type: application/json" \
    -d '{"username":"admin","password":"salah"}'
done
```
- **Kondisi Rentan**: Seluruh 10 permintaan mengembalikan status `401 Unauthorized` tanpa hambatan.
- **Kondisi Aman**: Permintaan 1 s.d. 5 mengembalikan `401`, dan mulai permintaan ke-6 mengembalikan status `429 Too Many Requests`.

---

## Referensi

- [OWASP Blocking Brute Force Attacks](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html#protect-against-automated-attacks)
- [MITRE CWE-307](https://cwe.mitre.org/data/definitions/307.html)
- [NIST SP 800-63B Section 5.2.2 (Rate Limiting)](https://pages.nist.gov/800-63-3/sp800-63b.html)
