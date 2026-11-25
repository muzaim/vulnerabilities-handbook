# Cross-Site Request Forgery (CSRF)

> **Metadata Keamanan:**
> - **CWE**: [CWE-352: Cross-Site Request Forgery (CSRF)](https://cwe.mitre.org/data/definitions/352.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 6.5 - 8.8)

---

## 1. Pengertian

**Cross-Site Request Forgery (CSRF)** adalah celah keamanan di mana situs web jahat pihak ketiga menjebak peramban (browser) pengguna yang sedang login di suatu situs resmi untuk secara diam-diam mengirimkan permintaan HTTP berbahaya atas nama pengguna tersebut tanpa persetujuan atau sepengetahuannya.

**Analogi Sederhana:**  
Bayangkan Anda memiliki stempel tanda tangan resmi di kantor pos yang menyatakan: "Segala surat yang dibubuhi stempel ini wajib dikirimkan". Anda meninggalkan stempel tersebut di atas meja loket kantor pos.  
Lalu ada orang asing menyodorkan selembar surat transfer dana palsu ke petugas pos sambil menunjuk stempel Anda di atas meja. Karena petugas pos hanya melihat stempel resmi Anda (dan bukan siapa yang menyodorkan surat), petugas langsung mengeksekusi transfer dana tersebut.

---

## 2. Cara Kerja

Perilaku dasar browser secara default adalah: **ketika browser mengirim permintaan HTTP ke suatu domain, browser akan otomatis melampirkan seluruh cookie milik domain tersebut** (termasuk cookie sesi login), bahkan jika permintaan itu dipicu oleh tautan atau skrip dari website lain!

```
[ Korban Sedang Login di bank.example ]
           │
           ▼ (Korban membuka tab lain: website-jahat.example)
[ Website Jahat memuat tag tersembunyi: ]
<img src="https://bank.example/transfer?to=penyerang&amount=1000000" />
           │
           ▼
[ Browser Korban otomatis melampirkan Cookie Sesi bank.example! ]
           │
           ▼
[ Server Bank mengira korban yang sengaja meminta transfer dana ]
```

Jika server bank hanya mengandalkan cookie untuk mengotentikasi permintaan dan tidak memeriksa asal permintaan (*origin/referer*) atau tidak meminta token anti-CSRF rahasia, transfer dana akan berhasil diproses.

---

## 3. Penyebab

- **Bergantung Semata-mata pada Cookie Otomatis**: Mempercayai cookie sesi yang otomatis dilampirkan browser tanpa bukti bahwa aksi tersebut benar-benar diinginkan pengguna.
- **Ketiadaan Token Anti-CSRF (Synchronizer Token)**: Tidak menyertakan token acak rahasia yang unik per sesi pada form atau header HTTP.
- **Konfigurasi Cookie `SameSite` yang Longgar**: Tidak menyetel atribut `SameSite=Lax` atau `SameSite=Strict` pada cookie autentikasi.
- **Menggunakan Metode HTTP GET untuk Aksi yang Mengubah Status**: Menggunakan `GET /delete?id=1` alih-alih `POST / DELETE` yang dilindungi.

---

## 4. Skenario Serangan

1. **Korban Login**: Korban login ke portal jejaring sosial `sosial.example`. Cookie sesi tersimpan di browser.
2. **Kunjungan Situs Berbahaya**: Tanpa logout, korban membuka email promo dan mengklik tautan ke `hadiah-gratis.example`.
3. **Pemicu Otomatis (Hidden Form)**: Halaman `hadiah-gratis.example` memuat form HTML tersembunyi dengan script auto-submit:
   ```html
   <form id="f" action="https://sosial.example/api/change-email" method="POST">
     <input type="hidden" name="email" value="hacker@attacker.com" />
   </form>
   <script>document.getElementById('f').submit();</script>
   ```
4. **Eksekusi di Latar Belakang**: Browser korban mengirimkan request POST ke `sosial.example` lengkap dengan cookie sesi sah milik korban.
5. **Pergantian Email Akun**: Email akun korban berubah menjadi email penyerang. Penyerang tinggal meminta lupa password untuk mengambil alih akun.

---

## 5. Dampak

- **Perubahan Data Sensitif Tanpa Izin**: Mengubah alamat email, kata sandi, atau nomor telepon akun.
- **Transaksi Finansial Ilegal**: Mengirim uang, membeli barang, atau memindahkan saldo.
- **Aksi Sosial Liar**: Mengirim pesan massal (spam), mengikuti akun lain, atau menghapus postingan korban.

---

## 6. Bagian Sistem yang Terdampak

- Formulir perubahan kata sandi dan email.
- Endpoint transfer dana atau pembayaran instan.
- Operasi penghapusan data atau perubahan pengaturan profil.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Router Wi-Fi Rumahan SmartNet:**  
Router SmartNet memiliki panel kontrol web di `http://192.168.1.1`. Admin login untuk mengonfigurasi internet.

Endpoint perubahan password router adalah:
`GET http://192.168.1.1/set_password?new_pass=admin123`

Seorang penyerang membuat forum online dan memposting gambar dengan URL:
`<img src="http://192.168.1.1/set_password?new_pass=hacked123" width="0" height="0">`

Ketika seorang pengguna yang sedang login di panel admin routernya membuka postingan forum tersebut, browser pengguna diam-diam memanggil IP lokal `192.168.1.1`. Password router seketika diubah menjadi `hacked123`, dan penyerang kini dapat mengendalikan router korban dari jaringan publik.

---

## 8. Cara Mendeteksi

- **Inspeksi Form & Permintaan State-Changing**: Periksa apakah form POST memiliki field token tersembunyi (`_csrf` atau `csrf_token`).
- **Inspeksi Atribut Cookie Sesi**: Periksa apakah cookie memiliki atribut `SameSite=Lax` atau `SameSite=Strict`.
- **Pengujian dengan Burp Suite**: Gunakan fitur *Generate CSRF PoC*. Buka file HTML hasil generate di browser yang sudah login. Jika aksi berhasil dieksekusi tanpa error, sistem rentan CSRF.

---

## 9. Pencegahan

1. **Gunakan Atribut Cookie `SameSite=Lax` atau `SameSite=Strict` (Standar Modern)**: Atribut ini memerintahkan browser untuk TIDAK mengirimkan cookie sesi jika permintaan berasal dari situs pihak ketiga (*cross-site*).
2. **Terapkan Anti-CSRF Token (Synchronizer Token Pattern)**: Server menyematkan token acak yang unik per sesi pada setiap form. Saat form dikirim, server memvalidasi apakah token yang dikirim cocok dengan token sesi di server.
3. **Gunakan Custom Request Header (Untuk API Modern)**: Untuk SPA / Mobile API, gunakan header seperti `X-Requested-With: XMLHttpRequest` atau `Authorization: Bearer <token>`. Situs pihak ketiga tidak bisa mengirimkan custom header sembarangan karena dibatasi oleh aturan CORS browser.
4. **Verifikasi Header `Origin` dan `Referer`**: Tolak permintaan jika nilai header `Origin` atau `Referer` tidak cocok dengan domain aplikasi Anda.

---

## 10. Penanganan

1. **Aktifkan SameSite Cookie Global**: Segera setel atribut `SameSite=Lax` pada seluruh cookie sesi yang diterbitkan server.
2. **Pasang Middleware CSRF**: Pasang pustaka pelindung CSRF (seperti `csurf` di Node.js atau middleware CSRF di framework Go/Gin).
3. **Konversikan Seluruh Aksi GET Menjadi POST/PUT/DELETE**: Pastikan tidak ada aksi modifikasi data yang berjalan menggunakan metode HTTP GET.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (CSRF Middleware)

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"net/http"
)

// Generate token anti-CSRF acak
func generateCSRFToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// ❌ KODE RENTAN (Vulnerable)
// Hanya memeriksa cookie tanpa memvalidasi token CSRF
func ChangeEmailVulnerable(w http.ResponseWriter, r *http.Request) {
	if r.Method == http.MethodPost {
		newEmail := r.FormValue("email")
		// Langsung update email user di DB...
		w.Write([]byte("Email diubah menjadi: " + newEmail))
	}
}

// ✅ KODE AMAN (Secure)
// Memvalidasi token CSRF yang dikirimkan bersama form atau header
func ChangeEmailSecure(w http.ResponseWriter, r *http.Request) {
	if r.Method == http.MethodPost {
		submittedToken := r.Header.Get("X-CSRF-Token")
		if submittedToken == "" {
			submittedToken = r.FormValue("csrf_token")
		}

		sessionToken := getCSRFTokenFromSession(r)

		// Tolak jika token tidak ada atau tidak cocok
		if submittedToken == "" || submittedToken != sessionToken {
			http.Error(w, "Validasi CSRF gagal: Token tidak sah", http.StatusForbidden)
			return
		}

		newEmail := r.FormValue("email")
		// Eksekusi update aman...
		w.Write([]byte("Email aman diubah menjadi: " + newEmail))
	}
}

func getCSRFTokenFromSession(r *http.Request) string { return "expected_token_123" }
```

### B. Contoh dalam TypeScript (Express & SameSite Cookie)

```typescript
import express, { Request, Response } from 'express';
import session from 'express-session';

const app = express();

// ✅ KODE AMAN: Melindungi sesi dengan atribut SameSite
app.use(session({
  name: 'session_id',
  secret: 'kunci_rahasia_super_kuat',
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'lax', // Browser TIDAK AKAN mengirim cookie ini dari situs ketiga pada request POST!
  }
}));

// Middleware verifikasi Custom Header untuk API JSON
export function verifyCustomHeader(req: Request, res: Response, next: Function) {
  // Situs jahat pihak ketiga tidak bisa mengirim header kustom tanpa lolos CORS Preflight
  const requestedWith = req.headers['x-requested-with'];
  if (req.method !== 'GET' && requestedWith !== 'XMLHttpRequest') {
    return res.status(403).json({ error: 'Permintaan ditolak: Header CSRF hilang' });
  }
  next();
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian CSRF Proof of Concept (PoC)
1. Buat file `csrf_test.html` lokal:
   ```html
   <html>
     <body>
       <form action="http://localhost:8080/api/change-email" method="POST">
         <input type="hidden" name="email" value="palsu@test.com" />
         <input type="submit" value="Klik Hadiah" />
       </form>
     </body>
   </html>
   ```
2. Buka aplikasi resmi di tab 1 dan lakukan login.
3. Buka file `csrf_test.html` di tab 2 dan klik tombol submit.
- **Kondisi Rentan**: Email di profil tab 1 berubah menjadi `palsu@test.com`.
- **Kondisi Aman**: Permintaan ditolak dengan status `403 Forbidden` (Token CSRF tidak ditemukan atau cookie sesi diblokir oleh SameSite).

---

## Referensi

- [OWASP Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger: Cross-site Request Forgery (CSRF)](https://portswigger.net/web-security/csrf)
- [MITRE CWE-352](https://cwe.mitre.org/data/definitions/352.html)
