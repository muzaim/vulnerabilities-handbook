# Session Management Vulnerabilities (Hijacking & Fixation)

> **Metadata Keamanan:**
> - **CWE**: [CWE-384: Session Fixation](https://cwe.mitre.org/data/definitions/384.html) & [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
> - **Tingkat Keparahan (Severity)**: High (CVSS ~ 7.5 - 8.8)

---

## 1. Pengertian

**Session Management Vulnerabilities** (Kerentanan Pengelolaan Sesi) adalah kelompok celah keamanan yang terjadi ketika identitas sesi sementara pengguna (*session token / cookie*) dapat dicuri, diprediksi, atau ditentukan oleh penyerang, sehingga penyerang dapat menggunakan sesi tersebut untuk menyamar sebagai korban tanpa perlu mengetahui kata sandi korban.

Dua bentuk serangan utama dalam kategori ini adalah:
1. **Session Hijacking (Pembajakan Sesi)**: Penyerang mencuri token sesi aktif milik korban yang sedang login (misalnya melalui jaringan Wi-Fi publik yang tidak aman atau melalui serangan XSS).
2. **Session Fixation (Fiksasi Sesi)**: Penyerang menetapkan (*mem-fiksasi*) ID sesi tertentu kepada korban sebelum korban login. Ketika korban berhasil login, sistem tetap mempertahankan ID sesi yang sudah diketahui penyerang tersebut.

**Analogi Sederhana:**  
- **Session Hijacking**: Di taman hiburan, setiap pengunjung memakai gelang tiket kertas tahan air. Penyerang menggunting gelang tiket dari pergelangan tangan Anda saat Anda lengah, lalu memasangnya di tangannya sendiri untuk naik semua wahana gratis atas nama Anda.
- **Session Fixation**: Penyerang sengaja meletakkan gelang tiket miliknya di meja loket dan menunggu Anda mengambil gelang tersebut. Begitu Anda membayar di loket dan mengaktifkan gelang itu, penyerang (yang sudah mencatat nomor seri gelang tersebut) bisa ikut masuk lewat pintu samping.

---

## 2. Cara Kerja

### A. Session Hijacking
Protokol HTTP bersifat *stateless* (tidak menyimpan memori). Agar server mengenali bahwa request berasal dari orang yang sama yang baru saja login, server memberikan **Session ID** yang disimpan di cookie browser. Jika cookie ini tidak diamankan:
- Ditransmisikan melalui HTTP biasa (bukan HTTPS), penyerang di jaringan Wi-Fi yang sama dapat menyadap (*sniffing*) cookie tersebut.
- Tidak diberi atribut `HttpOnly`, skrip XSS dapat membaca `document.cookie` dan mengirimkannya ke server penyerang.

### B. Session Fixation
1. Penyerang membuka website sasaran dan mendapatkan ID sesi mentah (belum terotentikasi), misalnya `SID=XYZ123`.
2. Penyerang mengirimkan link ke korban: `https://bank.example/login?session_id=XYZ123`.
3. Korban membuka tautan tersebut dan memasukkan username serta password miliknya.
4. **Kesalahan Fatal Sistem**: Setelah korban berhasil login, sistem **tidak membuat ID sesi baru**, melainkan meng-upgrade status `SID=XYZ123` menjadi terotentikasi.
5. Penyerang kini cukup membuka browsernya dengan `SID=XYZ123` dan langsung berada di akun korban.

---

## 3. Penyebab

- **Tidak Melakukan Regenerasi ID Sesi Saat Login**: Membiarkan session identifier yang sama tetap aktif setelah proses autentikasi berhasil.
- **Konfigurasi Cookie yang Longgar**: Tidak menyetel flag keamanan standar pada cookie (`HttpOnly`, `Secure`, `SameSite`).
- **Menerima ID Sesi dari Parameter URL**: Membiarkan session ID dikirim melalui query string (`?sid=...`) alih-alih melalui HTTP Header Cookie.
- **Ketiadaan Session Timeout**: Sesi dibiarkan aktif selamanya meskipun pengguna sudah menutup browser atau tidak ada aktivitas selama berhari-hari.

---

## 4. Skenario Serangan

1. **Persiapan Fiksasi**: Penyerang membuat tautan khusus yang menyertakan token sesi: `https://toko.example/login?JSESSIONID=ATTACKER_CHOSEN_ID`.
2. **Phishing Tautan**: Penyerang mengirim email promo palsu ke korban berisi tautan tersebut.
3. **Login Korban**: Korban mengklik tautan, lalu login dengan akun dan password miliknya.
4. **Pembajakan Akun**: Karena aplikasi Toko mempertahankan `JSESSIONID=ATTACKER_CHOSEN_ID`, penyerang yang sudah memegang ID tersebut cukup membuka situs dan langsung memiliki akses ke akun korban.

---

## 5. Dampak

- **Pengambilalihan Akun Penuh**: Penyerang bertindak atas nama korban tanpa memicu peringatan perubahan password.
- **Kebocoran Data Pribadi**: Akses langsung ke seluruh riwayat belanja, dokumen rahasia, atau data keuangan.
- **Penyalahgunaan Wewenang**: Melakukan transaksi finansial ilegal menggunakan saldo korban.

---

## 6. Bagian Sistem yang Terdampak

- Handler login dan logout aplikasi.
- Konfigurasi middleware session store (Express-session, Go gorilla/sessions).
- Endpoint yang menerima autentikasi berbasis cookie atau header authorization.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Portal Pajak Daerah KasPajak:**  
Portal KasPajak menggunakan cookie `SESSIONID` untuk melacak pengguna. Saat pengguna mengakses halaman utama, server membuat cookie `SESSIONID=ABC001`.

Ketika wajib pajak memasukkan NPWP dan password di halaman login, backend KasPajak memvalidasi kredensial, tetapi tidak memanggil fungsi `regenerateSession()`.

Seorang penyerang mengirimkan URL ke bendahara perusahaan dengan menyematkan cookie sesi yang sudah disiapkan penyerang. Ketika bendahara login untuk melaporkan SPT tahunan perusahaan bernilai miliaran rupiah, penyerang langsung merefresh browser miliknya dan mengunduh seluruh data SPT rahasia serta bukti potong pajak perusahaan tersebut.

---

## 8. Cara Mendeteksi

- **Uji Regenerasi Sesi**: Buka halaman login, catat nilai cookie sesi. Lakukan login berhasil. Periksa kembali nilai cookie sesi di browser. Jika nilainya sama persis sebelum dan sesudah login, sistem positif rentan **Session Fixation**.
- **Inspeksi Atribut Cookie**: Periksa apakah cookie sesi memiliki atribut `HttpOnly; Secure; SameSite=Lax (atau Strict)`.
- **Inspeksi URL**: Pastikan tidak ada ID sesi yang muncul di URL saat bernavigasi di situs web.

---

## 9. Pencegahan

1. **Selalu Regenerasi Session ID Saat Login (Wajib Mutlak)**: Setiap kali status autentikasi berubah (dari guest ke user login, atau dari user ke admin), sistem **wajib memusnahkan ID sesi lama dan menerbitkan ID sesi baru yang acak**.
2. **Gunakan Flag Cookie yang Aman**:
   - `HttpOnly`: Mencegah skrip JavaScript (XSS) membaca cookie.
   - `Secure`: Memastikan cookie hanya dikirim melalui koneksi terenkripsi HTTPS.
   - `SameSite=Lax / Strict`: Melindungi cookie dari serangan CSRF.
3. **Tolak ID Sesi dari URL**: Hanya terima session identifier melalui HTTP Cookie header, jangan pernah dari query string atau body.
4. **Terapkan Idle Timeout & Absolute Timeout**: Hancurkan sesi otomatis jika pengguna tidak aktif selama 15-30 menit, dan batasi usia maksimal sesi (misal 12-24 jam).

---

## 10. Penanganan

1. **Revoke Seluruh Sesi Aktif**: Segera hapus seluruh data sesi di session store (Redis / Database) untuk memaksa seluruh pengguna login ulang.
2. **Deploy Patch Regenerasi**: Pasang baris pemanggilan regenerasi sesi di handler login.
3. **Perbarui Kebijakan Cookie**: Aktifkan flag `HttpOnly` dan `Secure` pada server produksi.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Session Management)

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"net/http"
	"sync"
	"time"
)

type SessionStore struct {
	mu       sync.Mutex
	sessions map[string]string // sessionID -> userID
}

var store = &SessionStore{sessions: make(map[string]string)}

func generateSecureToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// ❌ KODE RENTAN (Session Fixation)
// Menggunakan cookie sesi yang sudah ada dari klien saat login berhasil
func LoginHandlerVulnerable(w http.ResponseWriter, r *http.Request) {
	// Kredensial divalidasi...
	cookie, err := r.Cookie("session_id")
	if err == nil {
		// RENTAN: Mengaitkan session_id lama bawaan penyerang dengan akun korban!
		store.sessions[cookie.Value] = "user_123"
		w.Write([]byte("Login berhasil"))
	}
}

// ✅ KODE AMAN (Secure Session Regeneration)
// Selalu musnahkan sesi lama dan terbitkan ID sesi baru yang aman
func LoginHandlerSecure(w http.ResponseWriter, r *http.Request) {
	// 1. Jika ada cookie lama, hapus dari store
	if oldCookie, err := r.Cookie("session_id"); err == nil {
		store.mu.Lock()
		delete(store.sessions, oldCookie.Value)
		store.mu.Unlock()
	}

	// 2. Terbitkan Session ID BARU yang acak dan kuat (32 bytes entropy)
	newSessionID := generateSecureToken()
	store.mu.Lock()
	store.sessions[newSessionID] = "user_123"
	store.mu.Unlock()

	// 3. Kirim cookie baru dengan flag keamanan lengkap
	http.SetCookie(w, &http.Cookie{
		Name:     "session_id",
		Value:    newSessionID,
		Path:     "/",
		Expires:  time.Now().Add(2 * time.Hour), // Timeout sesi
		HttpOnly: true,                          // Kebal XSS
		Secure:   true,                          // Wajib HTTPS
		SameSite: http.SameSiteLaxMode,          // Proteksi CSRF
	})

	w.Write([]byte("Login aman berhasil"))
}
```

### B. Contoh dalam TypeScript (Express & Express-Session)

```typescript
import express, { Request, Response } from 'express';
import session from 'express-session';

const app = express();

// ✅ Konfigurasi Cookie Sesi yang Aman
app.use(session({
  name: '__Host-session', // Prefiks cookie aman
  secret: process.env.SESSION_SECRET || 'kunci_rahasia_yang_sangat_panjang_dan_acak_32_karakter',
  resave: false,
  saveUninitialized: false, // Jangan buat sesi sebelum ada data
  cookie: {
    httpOnly: true,        // Mencegah pencurian melalui XSS
    secure: true,          // Hanya dikirim lewat HTTPS
    sameSite: 'lax',       // Mencegah CSRF
    maxAge: 30 * 60 * 1000 // 30 Menit Idle Timeout
  }
}));

// ❌ KODE RENTAN (Vulnerable)
app.post('/api/login-vulnerable', (req: Request, res: Response) => {
  // Hanya menempelkan data user ke sesi yang sudah ada tanpa regenerasi ID
  req.session.userId = 'user_123';
  res.json({ message: 'Login sukses' });
});

// ✅ KODE AMAN (Secure)
app.post('/api/login-secure', (req: Request, res: Response) => {
  // 1. Validasi kredensial...
  const authenticatedUserId = 'user_123';

  // 2. REGENERASI ID SESI SECARA EKSPLISIT!
  req.session.regenerate((err) => {
    if (err) {
      return res.status(500).json({ error: 'Gagal meregenerasi sesi' });
    }

    // 3. Simpan data pengguna ke ID sesi baru yang masih bersih
    req.session.userId = authenticatedUserId;
    res.json({ message: 'Login berhasil dengan sesi baru' });
  });
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Session Fixation di Browser
1. Buka browser dalam mode Incognito, catat nilai cookie sesi pada form login.
2. Lakukan login berhasil.
3. Buka tab *Application* -> *Cookies* pada DevTools browser.
- **Kondisi Rentan**: Nilai cookie sesi sama persis dengan angka sebelum login.
- **Kondisi Aman**: Nilai cookie sesi berubah menjadi string acak baru setelah login berhasil.

---

## Referensi

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [MITRE CWE-384: Session Fixation](https://cwe.mitre.org/data/definitions/384.html)
- [PortSwigger: Session Hijacking and Fixation](https://portswigger.net/web-security/authentication/other-mechanisms)
