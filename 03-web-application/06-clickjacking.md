# Clickjacking (UI Redressing)

> **Metadata Keamanan:**
> - **CWE**: [CWE-1021: Improper Restriction of Rendered UI Layers or Frames](https://cwe.mitre.org/data/definitions/1021.html)
> - **OWASP**: [A05:2021 – Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
> - **Tingkat Keparahan (Severity)**: Medium (CVSS ~ 4.3 - 6.5)

---

## 1. Pengertian

**Clickjacking** (atau *UI Redressing*) adalah teknik manipulasi visual di mana penyerang menjebak pengguna untuk mengklik tombol atau tautan yang tidak terlihat (*invisible*), padahal pengguna mengira mereka sedang mengklik elemen lain di halaman web yang tampak tidak berbahaya (seperti tombol game atau video).

**Analogi Sederhana:**  
Bayangkan selembar mika plastik bening yang diletakkan di atas dokumen penyerahan hak milik rumah Anda. Di atas mika plastik tersebut ditempelkan tombol bertuliskan: *"Tekan di sini untuk memenangkan hadiah mobil!"*.  
Ketika Anda menekan tombol stiker mobil tersebut dengan jempol, sebenarnya jempol Anda sedang menekan stempel tanda tangan di dokumen penyerahan rumah yang berada persis di lapisan bawah mika bening tersebut.

---

## 2. Cara Kerja

Penyerang membangun situs web berbahaya (misal `game-hadiah.example`) yang memuat situs web target sah Anda (misal `bank.example/transfer`) di dalam sebuah bingkai iframe tersembunyi:

```html
<!-- Situs Penyerang -->
<div style="position: absolute; top: 100px; left: 100px; z-index: 1;">
  <button>KLIK DI SINI UNTUK MAINKAN GAME</button>
</div>

<!-- Iframe situs target diletakkan tepat di atas tombol game, tapi DIBUAT TRANSPARAN (opacity: 0) -->
<iframe src="https://bank.example/transfer-confirm" 
        style="position: absolute; top: 90px; left: 90px; opacity: 0; z-index: 2;">
</iframe>
```

Ketika korban yang sedang login di `bank.example` mengklik tombol game tersebut, kursor korban sebenarnya mengklik tombol "Konfirmasi Transfer" milik bank yang tidak terlihat di lapisan iframe atas.

---

## 3. Penyebab

- **Tidak Memasang Header Proteksi Framing**: Server web tidak menyetel HTTP Header `X-Frame-Options` atau `Content-Security-Policy: frame-ancestors`.
- **Membiarkan Aplikasi Dimuat di Sembarang Situs**: Secara default, peramban mengizinkan situs web mana pun untuk memuat halaman web lain di dalam tag `<iframe>` kecuali dilarang secara eksplisit oleh server.

---

## 4. Skenario Serangan

1. **Pembuatan Halaman Umpan**: Penyerang membuat halaman web game interaktif sederhana: "Klik balon untuk mendapatkan poin".
2. **Penyusunan Lapisan Iframe**: Di bawah posisi balon, penyerang meletakkan iframe akun media sosial target: `https://medsos.example/settings/delete-account` dengan transparansi 100% (`opacity: 0`).
3. **Pancingan Korban**: Korban membuka situs game dan asyik mengklik balon.
4. **Penghapusan Akun Tanpa Sadar**: Salah satu klik korban tepat mengenai tombol "Hapus Akun Saya" di dalam iframe media sosial. Akun korban langsung terhapus seketika.

---

## 5. Dampak

- **Aksi Finansial Tanpa Sadar**: Mentransfer dana atau menyetujui transaksi belanja satu-klik (*1-click purchase*).
- **Pengubahan Pengaturan Keamanan**: Mematikan 2FA atau mengizinkan akses aplikasi pihak ketiga.
- **Pemberian Izin Perangkat**: Menipu pengguna untuk menyetujui akses kamera atau mikrofon laptop.

---

## 6. Bagian Sistem yang Terdampak

- Halaman pengaturan akun dan profil.
- Tombol transfer uang, pembayaran, atau checkout pesanan.
- Formulir persetujuan otorisasi OAuth ("Izinkan aplikasi ini mengakses data Anda?").

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Voting Pemilihan Ketua Organisasi VoteNow:**  
VoteNow adalah platform pemungutan suara online mahasiswa. Setiap mahasiswa hanya memiliki 1 hak suara.

Penyerang membuat situs blog meme lucu yang viral. Di atas tombol "Lihat Meme Berikutnya", penyerang menumpuk iframe transparan tombol "Pilih Kandidat Nomor 2" dari situs VoteNow. Ribuan mahasiswa yang membuka blog meme tersebut dan mengklik tombol "Next Meme" secara tidak sadar telah mengirimkan suara mereka untuk Kandidat Nomor 2. Kandidat tersebut menang mutlak akibat manipulasi suara via clickjacking.

---

## 8. Cara Mendeteksi

- **Uji Coba Iframe Sederhana**: Buat file HTML lokal:
  ```html
  <iframe src="https://aplikasi-anda.example"></iframe>
  ```
  Buka file tersebut di browser. Jika halaman aplikasi Anda berhasil muncul di dalam bingkai, aplikasi Anda **rentan Clickjacking**.
- **Inspeksi Header Keamanan**: Periksa apakah respons HTTP memuat `X-Frame-Options: DENY` atau `SAMEORIGIN`, serta directive `frame-ancestors` pada `Content-Security-Policy`.

---

## 9. Pencegahan

1. **Gunakan Header `Content-Security-Policy: frame-ancestors 'none'` (Standar Modern Utama)**:  
   Header ini melarang browser memuat halaman Anda di dalam iframe di situs mana pun:
   ```http
   Content-Security-Policy: frame-ancestors 'none';
   ```
   Atau jika hanya ingin diizinkan pada domain Anda sendiri:
   ```http
   Content-Security-Policy: frame-ancestors 'self';
   ```
2. **Gunakan Header Legacy `X-Frame-Options: DENY` / `SAMEORIGIN` (Untuk Kompatibilitas Browser Lama)**:  
   ```http
   X-Frame-Options: DENY
   ```
3. **Minta Konfirmasi Ulang / Re-Authentication untuk Aksi Kritis**:  
   Untuk aksi penting (seperti transfer uang atau hapus akun), selalu minta konfirmasi password atau kode OTP.

---

## 10. Penanganan

1. **Pasang Header di Reverse Proxy (Nginx / Cloudflare)**: Tambahkan baris `add_header X-Frame-Options "SAMEORIGIN" always;` pada konfigurasi Nginx global.
2. **Pasang Middleware Keamanan di Backend**: Gunakan library keamanan standar (seperti `helmet` di Node.js).

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Security Headers Middleware)

```go
package main

import (
	"net/http"
)

// ✅ KODE AMAN: Middleware yang menyuntikkan header anti-clickjacking
func AntiClickjackingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 1. Standar Modern: Content-Security-Policy frame-ancestors
		// 'self' berarti hanya domain yang sama yang boleh memuat di iframe
		// 'none' jika sama sekali dilarang dimuat dalam iframe
		w.Header().Set("Content-Security-Policy", "frame-ancestors 'self'")

		// 2. Standar Legacy: X-Frame-Options untuk browser yang belum mendukung CSP level 2
		w.Header().Set("X-Frame-Options", "SAMEORIGIN")

		next.ServeHTTP(w, r)
	})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/dashboard", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Dashboard Pengguna"))
	})

	// Bungkus router dengan middleware keamanan
	http.ListenAndServe(":8080", AntiClickjackingMiddleware(mux))
}
```

### B. Contoh dalam TypeScript (Express & Helmet)

```typescript
import express from 'express';
import helmet from 'helmet';

const app = express();

// ✅ KODE AMAN: Menggunakan Helmet untuk proteksi otomatis
app.use(
  helmet({
    // Konfigurasi X-Frame-Options
    xFrameOptions: { action: 'deny' }, // Tolak semua framing (bahkan dari domain sendiri)

    // Konfigurasi CSP frame-ancestors
    contentSecurityPolicy: {
      directives: {
        ...helmet.contentSecurityPolicy.getDefaultDirectives(),
        'frame-ancestors': ["'none'"], // Tidak boleh dimuat di iframe mana pun
      },
    },
  })
);

app.get('/settings/delete-account', (req, res) => {
  res.send('<h1>Halaman Hapus Akun</h1>');
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan File HTML
1. Buat file `test_frame.html` di komputer lokal:
   ```html
   <!DOCTYPE html>
   <html>
     <body>
       <h2>Pengujian Clickjacking</h2>
       <iframe src="http://localhost:8080/dashboard" width="500" height="400"></iframe>
     </body>
   </html>
   ```
2. Buka `test_frame.html` di browser (Chrome / Firefox).
- **Kondisi Rentan**: Halaman dashboard aplikasi muncul di dalam kotak frame.
- **Kondisi Aman**: Kotak frame kosong/abu-abu, dan konsol browser menampilkan pesan error:  
  `Refused to display 'http://localhost:8080/dashboard' in a frame because it set 'X-Frame-Options' to 'SAMEORIGIN'`.

---

## Referensi

- [OWASP Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html)
- [PortSwigger: Clickjacking](https://portswigger.net/web-security/clickjacking)
- [MDN Web Docs: X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)
