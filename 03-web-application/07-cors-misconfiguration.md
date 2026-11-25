# CORS Misconfiguration (Kesalahan Konfigurasi CORS)

> **Metadata Keamanan:**
> - **CWE**: [CWE-942: Permissive Cross-domain Policy with Untrusted Domains](https://cwe.mitre.org/data/definitions/942.html)
> - **OWASP**: [A05:2021 – Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 6.5 - 8.5)

---

## 1. Pengertian

**CORS Misconfiguration** adalah kesalahan pengaturan kebijakan *Cross-Origin Resource Sharing* pada server web atau API, di mana server secara tidak sengaja mengizinkan situs web asing yang tidak dipercaya untuk membaca data rahasia pengguna yang dikembalikan oleh API pribadi.

**Konsep Penting:**  
Secara default, peramban memiliki aturan **SOP (Same-Origin Policy)** yang melarang situs `jahat.example` membaca respon data dari `bank.example`. CORS adalah mekanisme resmi bagi server untuk memberikan "pengecualian". Jika pengecualian ini disetel sembarangan, SOP menjadi tidak berguna.

**Analogi Sederhana:**  
Bayangkan brankas dokumen rahasia perusahaan yang dijaga resepsionis.  
Aturan SOP standar: "Hanya karyawan dengan seragam perusahaan ini yang boleh membaca isi dokumen".  
Lalu manajemen memasang papan CORS: *"Siapa pun yang datang dan meminta dokumen ini, tolong berikan fotokopinya, termasuk orang asing di jalanan yang membawa kartu nama acak"*. Resepsionis pun menyerahkan laporan rahasia ke orang asing tersebut.

---

## 2. Cara Kerja

Ketika JavaScript di browser pada domain `https://penyerang.example` melakukan `fetch('https://api.bank.example/me')`:
1. Browser mengirimkan header: `Origin: https://penyerang.example`.
2. Jika server mengalami salah konfigurasi, server membalas dengan header:
   ```http
   Access-Control-Allow-Origin: https://penyerang.example
   Access-Control-Allow-Credentials: true
   ```
3. Browser korban melihat bahwa server bank secara eksplisit mengizinkan `https://penyerang.example` dan mengizinkan kredensial/cookie.
4. Browser membiarkan skrip di situs penyerang membaca seluruh data sensitif (nama, saldo, nomor rekening) yang dikembalikan oleh API bank!

### Pola Kesalahan Konfigurasi Umum:
- **Refleksi Asal Tanpa Batasan (Arbitrary Origin Reflection)**: Server secara otomatis membaca header `Origin` dari klien dan menaruhnya kembali di `Access-Control-Allow-Origin`.
- **Mempercayai Origin `null`**: Membolehkan `Access-Control-Allow-Origin: null`. Penyerang bisa memicu origin `null` menggunakan iframe sandbox lokal.
- **Kesalahan Regex Domain**: Menggunakan regex longgar seperti `api.bank.example` yang bisa di-bypass oleh penyerang dengan mendaftarkan domain `api.bank.example.evil.com` atau `notapi.bank.example`.

---

## 3. Penyebab

- **Jalan Pintas Developer saat Menemukan Error CORS**: Developer pemula sering kali malas mengonfigurasi domain dan memilih menyalin konfigurasi dari internet: menyetel `origin: true` atau merefleksikan seluruh origin demi menghilangkan pesan error di konsol browser.
- **Kombinasi Berbahaya Wildcard & Credentials**: Mencoba memberikan akses publik tetapi tetap menyalakan `Allow-Credentials`.

---

## 4. Skenario Serangan

1. **Analisis Konfigurasi**: Penyerang mengirim request uji coba ke `https://api.toko.example/user/profile` dengan header palsu: `Origin: https://attacker.com`.
2. **Konfirmasi Kerentanan**: Server membalas:  
   `Access-Control-Allow-Origin: https://attacker.com` dan `Access-Control-Allow-Credentials: true`.
3. **Penyebaran Jebakan**: Penyerang membuat halaman web phishing `https://attacker.com/promo`.
4. **Pencurian Data Korban**: Korban yang sedang login di `toko.example` membuka tautan promo tersebut. Skrip JavaScript di situs penyerang memanggil `fetch('https://api.toko.example/user/profile', {credentials: 'include'})`, membaca seluruh data pribadi korban, dan mengirimkannya ke server penyerang.

---

## 5. Dampak

- **Kebocoran Data Pribadi Skala Penuh**: Penyerang dapat menyedot data profil, nomor kartu kredit, riwayat transaksi, atau pesan pribadi korban.
- **Pencurian API Keys & Token Rahasia**: Membaca token sesi atau kunci API yang dikembalikan oleh endpoint profil.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint RESTful API yang mengembalikan data pribadi pengguna (`/api/me`, `/api/orders`).
- Server Single Sign-On (SSO) dan endpoint otentikasi.
- Konfigurasi middleware CORS di Express (Node.js), Gin / Chi (Go), atau API Gateway.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem FinTech DompetKeluarga:**  
DompetKeluarga memiliki endpoint API internal:
`GET https://api.dompetkeluarga.example/v1/wallet/balance`

Agar aplikasi mobile dan web internal bisa berkomunikasi tanpa ribet, developer menyetel middleware backend:
```javascript
res.setHeader('Access-Control-Allow-Origin', req.headers.origin || '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

Penyerang membuat situs web forum gosip. Saat seorang pengguna DompetKeluarga membaca forum tersebut, skrip tersembunyi langsung memanggil API saldo dan mutasi rekening korban. Data seluruh tabungan, nomor rekening bank, dan daftar transaksi terakhir korban otomatis terkirim ke database penyerang tanpa disadari oleh korban.

---

## 8. Cara Mendeteksi

- **Uji Coba dengan cURL (Origin Reflection Check)**:
  ```bash
  curl -H "Origin: https://attacker.com" -I https://api.anda.example/me
  ```
  Periksa apakah `Access-Control-Allow-Origin` di respons bernilai `https://attacker.com`. Jika ya dan `Allow-Credentials: true`, sistem positif rentan.
- **Uji Coba Origin Null**:
  ```bash
  curl -H "Origin: null" -I https://api.anda.example/me
  ```

---

## 9. Pencegahan

1. **Gunakan Whitelist Asal yang Sangat Ketat (Strict Whitelist)**:  
   Hanya izinkan domain resmi milik Anda (misal: HANYA `https://toko.example.com` dan `https://admin.toko.example.com`).
2. **Jangan Pernah Merefleksikan Header `Origin` Secara Otomatis**:  
   Periksa apakah domain pemanggil ada dalam daftar whitelist sebelum menetapkan header `Access-Control-Allow-Origin`.
3. **Tolak Origin `null`**:  
   Jangan pernah menyertakan `null` di dalam daftar domain yang diizinkan.
4. **Jika API Bersifat Publik, Matikan Kredensial**:  
   Jika API memang ditujukan untuk publik bebas (seperti data cuaca atau kurs mata uang), gunakan `Access-Control-Allow-Origin: *` dan **JANGAN PERNAH** menyalakan `Access-Control-Allow-Credentials: true`.

---

## 10. Penanganan

1. **Deploy Patch Konfigurasi CORS**: Hapus konfigurasi refleksi liar dan ganti dengan array domain tetap.
2. **Periksa Penggunaan Cookie SameSite**: Pastikan cookie menggunakan `SameSite=Lax` atau `Strict`, yang memberikan perlindungan tambahan dari pembacaan lintas domain.
3. **Audit Log Akses CORS**: Analisis log server untuk melihat apakah ada domain asing yang sering memanggil API dengan kredensial.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Strict CORS Middleware)

```go
package main

import (
	"net/http"
)

var allowedOrigins = map[string]bool{
	"https://app.example.com":       true,
	"https://dashboard.example.com": true,
}

// ✅ KODE AMAN: Middleware CORS dengan Whitelist Ketat
func SecureCORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")

		// 1. Cek apakah origin ada di dalam daftar whitelist resmi
		if allowedOrigins[origin] {
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Set("Access-Control-Allow-Credentials", "true")
			w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
		}

		// 2. Tangani Preflight request (OPTIONS)
		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusNoContent)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

### B. Contoh dalam TypeScript (Express & `cors` package)

```typescript
import express from 'express';
import cors from 'cors';

const app = express();

const ALLOWED_ORIGINS = [
  'https://www.tokoku.com',
  'https://admin.tokoku.com',
];

// ✅ KODE AMAN: Konfigurasi CORS yang benar menggunakan validator fungsi
app.use(
  cors({
    origin: (origin, callback) => {
      // Izinkan permintaan tanpa origin (seperti aplikasi mobile native atau curl)
      if (!origin) return callback(null, true);

      if (ALLOWED_ORIGINS.includes(origin)) {
        return callback(null, true);
      } else {
        return callback(new Error('Diblokir oleh kebijakan keamanan CORS'));
      }
    },
    credentials: true, // Hanya diizinkan untuk domain yang lolos whitelist di atas!
  })
);

app.get('/api/me', (req, res) => {
  res.json({ name: 'Budi', balance: 5000000 });
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Otomatis dengan Domain Asing
Kirimkan permintaan dengan memalsukan header origin:
```bash
curl -I -H "Origin: https://evil-attacker.com" http://localhost:8080/api/me
```
- **Kondisi Rentan**: Header `Access-Control-Allow-Origin: https://evil-attacker.com` muncul di respons.
- **Kondisi Aman**: Header `Access-Control-Allow-Origin` sama sekali tidak dicantumkan di respons, atau sistem mengembalikan error penolakan CORS.

---

## Referensi

- [PortSwigger: Cross-Origin Resource Sharing (CORS)](https://portswigger.net/web-security/cors)
- [OWASP CORS Origin Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Origin_Resource_Sharing_Cheat_Sheet.html)
- [MITRE CWE-942](https://cwe.mitre.org/data/definitions/942.html)
