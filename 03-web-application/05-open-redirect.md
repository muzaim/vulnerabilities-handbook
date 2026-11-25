# Open Redirect

> **Metadata Keamanan:**
> - **CWE**: [CWE-601: URL Redirection to Untrusted Site ('Open Redirect')](https://cwe.mitre.org/data/definitions/601.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
> - **Tingkat Keparahan (Severity)**: Medium (CVSS ~ 6.1 - 7.4)

---

## 1. Pengertian

**Open Redirect** adalah celah keamanan di mana aplikasi web mengizinkan parameter URL menentukan ke mana pengguna akan dialihkan (*redirect*) tanpa memvalidasi apakah domain tujuan tersebut aman atau berada di bawah kendali aplikasi resmi.

Akibatnya, penyerang dapat menyusun tautan yang berawalan domain resmi perusahaan Anda, tetapi secara otomatis mengarahkan korban ke situs web tiruan penipu (*phishing*).

**Analogi Sederhana:**  
Bayangkan sebuah rambu penunjuk jalan resmi yang dipasang oleh dinas perhubungan di pintu keluar tol: *"Gerbang Resmi Bank Mandiri: Ikuti tanda panah ini"*.  
Namun rambu tersebut memiliki engsel longgar yang bisa diputar oleh siapa saja. Seorang penjahat memutar panah penunjuk jalan ke arah gang buntu sepi di mana komplotan begal sudah menunggu. Pengendara mobil percaya karena mereka melihat rambu resmi di awal jalan.

---

## 2. Cara Kerja

Aplikasi web sering kali memiliki parameter untuk mengembalikan pengguna ke halaman asal setelah tindakan tertentu, misalnya setelah login atau logout:
```http
https://bank.example/login?returnUrl=/dashboard
```
Setelah login berhasil, backend mengeksekusi HTTP response header:
```http
HTTP/1.1 302 Found
Location: /dashboard
```

Celah terjadi ketika backend langsung mengambil nilai parameter tersebut dan menaruhnya di header `Location`:
```http
https://bank.example/login?returnUrl=https://phishing-bank.evil
```
Server mengembalikan:
```http
HTTP/1.1 302 Found
Location: https://phishing-bank.evil
```
Peramban pengguna akan otomatis berpindah ke situs `phishing-bank.evil`. Karena korban awalnya mengklik tautan resmi `https://bank.example`, korban tidak curiga dan langsung memasukkan kredensialnya di situs tiruan tersebut.

---

## 3. Penyebab

- **Mempercayai Input URL Eksternal Mentah**: Meneruskan nilai parameter `next`, `returnUrl`, `redirect_to` langsung ke fungsi redirect tanpa validasi.
- **Validasi Domain yang Keliru (Regex Flaw)**: Menggunakan pemeriksaan substring sederhana, misalnya memeriksa apakah URL mengandung `bank.example`. Penyerang bisa mengakalinya dengan:
  - `https://bank.example.evil.com`
  - `https://evil.com?bank.example`
  - `https://evil.com#bank.example`
- **Ketiadaan Whitelist Domain Tujuan**: Tidak membatasi pengalihan hanya ke jalur relatif lokal (*relative path* seperti `/home`).

---

## 4. Skenario Serangan

1. **Penyusunan Tautan Jebakan**: Penyerang membuat tautan phishing berpenampilan meyakinkan:
   `https://portal.kampus.ac.id/auth/login?redirect=https://login-portal-palsu.com`.
2. **Penyebaran**: Penyerang mengirimkan email kepada mahasiswa: *"Pembaruan Jadwal Ujian: Silakan login ke portal resmi kampus melalui tautan ini"*.
3. **Pemberian Kepercayaan**: Mahasiswa melihat bahwa domain berawalan `https://portal.kampus.ac.id`, sehingga merasa aman untuk mengkliknya.
4. **Pengalihan Otomatis**: Saat tautan dibuka, server kampus langsung mengalihkan browser mahasiswa ke `https://login-portal-palsu.com` yang tampilannya 100% mirip portal kampus. Mahasiswa memasukkan NIM dan password, dan akun mahasiswa berhasil dicuri.

---

## 5. Dampak

- **Meningkatkan Efektivitas Serangan Phishing**: Penipuan tampak sah karena menggunakan domain asli perusahaan/kampus di awal tautan.
- **Pencurian Kredensial dan Token OAuth**: Pada alur OAuth / SSO, open redirect dapat digunakan untuk mencuri token otentikasi (`authorization code`) ke server penyerang jika parameter `redirect_uri` tidak divalidasi ketat.

---

## 6. Bagian Sistem yang Terdampak

- Alur login, logout, dan pendaftaran akun.
- Alur pergantian bahasa situs (`/lang?set=id&return=...`).
- Alur konfirmasi transaksi perbankan / payment gateway callback.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem OAuth SSO Perusahaan Global BUMN:**  
Karyawan menggunakan akun SSO tunggal untuk masuk ke puluhan aplikasi internal. Endpoint login SSO adalah:
`https://sso.bumn.example/login?service=https://mail.bumn.example`

Developer tidak memvalidasi parameter `service`. Penyerang mengirimkan tautan:
`https://sso.bumn.example/login?service=https://attacker-logger.com`

Ketika korban berhasil login di portal SSO asli, server BUMN melampirkan kode otorisasi (*auth code*) rahasia di URL redirect:  
`Location: https://attacker-logger.com?code=AUTH_SECRET_CODE`

Server penyerang menangkap kode otorisasi tersebut dan langsung menukarkannya dengan access token untuk mengakses akun email dan dokumen internal korban.

---

## 8. Cara Mendeteksi

- **Uji Coba Pengalihan Domain Luar**: Masukkan domain luar pada parameter redirect:
  `?redirect=https://google.com`, `?next=//google.com`, `?return=/\google.com`.
- **Uji Coba Bypass Regex**: Uji domain dengan subdomain manipulatif: `?next=https://toko.example.attacker.com`.
- **Inspeksi Response Header**: Amati apakah ada header `Location: https://google.com` pada respons HTTP 302/301.

---

## 9. Pencegahan

1. **Wajibkan Jalur Relatif Saja (Relative Path Only)**:  
   Pastikan URL tujuan selalu diawali dengan satu karakter garis miring (`/`) dan BUKAN diawali dengan garis miring ganda (`//` yang diartikan browser sebagai protokol independen).
2. **Gunakan Whitelist Domain yang Ketat**:  
   Jika memang harus mengalihkan ke domain eksternal, gunakan daftar putih (*strict whitelist*) domain yang diperbolehkan.
3. **Tampilkan Halaman Peringatan Konfirmasi Pengalihan (Exit Notice)**:  
   Jika pengguna dialihkan ke luar situs aplikasi, tampilkan halaman perantara: *"Anda akan meninggalkan situs kami menuju [domain luar]. Apakah Anda yakin?"*.

---

## 10. Penanganan

1. **Deploy Validasi URL Segera**: Pasang fungsi validator pada seluruh fungsi redirect.
2. **Kunci Nilai Default**: Jika parameter redirect kosong atau mencurigakan, selalu alihkan pengguna ke halaman dashboard utama (`/dashboard`).
3. **Audit OAuth Whitelist**: Pada penyedia identitas (OAuth IdP), pastikan seluruh `redirect_uri` klien terdaftar secara statis dan cocok seratus persen (*exact matching*).

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"net/http"
	"net/url"
	"strings"
)

// ❌ KODE RENTAN (Vulnerable)
// Mengambil parameter target langsung dan dialihkan
func RedirectVulnerable(w http.ResponseWriter, r *http.Request) {
	target := r.URL.Query().Get("next")
	// Jika target = "https://evil.com", browser langsung diarahkan ke sana!
	http.Redirect(w, r, target, http.StatusFound)
}

// ✅ KODE AMAN (Secure)
// 1. Memaksa target hanya berupa jalur relatif lokal
// 2. Mencegah manipulasi garis miring ganda "//evil.com"
func RedirectSecure(w http.ResponseWriter, r *http.Request) {
	target := r.URL.Query().Get("next")

	if !isSafeLocalURL(target) {
		// Jika target tidak aman atau mencurigakan, alihkan ke dashboard default
		target = "/dashboard"
	}

	http.Redirect(w, r, target, http.StatusFound)
}

func isSafeLocalURL(target string) bool {
	if target == "" {
		return false
	}

	parsed, err := url.Parse(target)
	if err != nil {
		return false
	}

	// 1. Tolak jika ada host atau skema (berarti domain eksternal)
	if parsed.Scheme != "" || parsed.Host != "" {
		return false
	}

	// 2. Wajib diawali dengan "/" dan TIDAK boleh diawali "//" atau "/\"
	if !strings.HasPrefix(target, "/") || strings.HasPrefix(target, "//") || strings.HasPrefix(target, "/\\") {
		return false
	}

	return true
}
```

### B. Contoh dalam TypeScript (Express)

```typescript
import { Request, Response } from 'express';

const ALLOWED_EXTERNAL_DOMAINS = ['payment.partner.com', 'helpdesk.mycompany.com'];

// ✅ KODE AMAN: Validator redirect yang ketat
export function handleLoginRedirect(req: Request, res: Response) {
  const nextUrl = req.query.next as string;
  const safeDestination = getSafeRedirectUrl(nextUrl, '/home');

  return res.redirect(safeDestination);
}

function getSafeRedirectUrl(targetUrl: string | undefined, defaultFallback: string): string {
  if (!targetUrl) return defaultFallback;

  // 1. Izinkan jalur relatif lokal murni (contoh: "/profile", "/orders/12")
  // Cegah "//evil.com" atau "/\evil.com" yang diartikan browser sebagai domain eksternal
  if (targetUrl.startsWith('/') && !targetUrl.startsWith('//') && !targetUrl.startsWith('/\\')) {
    return targetUrl;
  }

  // 2. Jika berupa URL absolut, validasi dengan daftar putih domain terpercaya
  try {
    const parsed = new URL(targetUrl);
    if (ALLOWED_EXTERNAL_DOMAINS.includes(parsed.hostname)) {
      return parsed.toString();
    }
  } catch {
    // String bukan format URL yang valid
  }

  return defaultFallback;
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Otomatis Parameter Redirect
Kirimkan permintaan dengan URL eksternal berbahaya:
```bash
curl -I "http://localhost:8080/auth/login?next=https://evil-hacker.com"
curl -I "http://localhost:8080/auth/login?next=//evil-hacker.com"
```
- **Kondisi Rentan**: Header respons memuat `Location: https://evil-hacker.com`.
- **Kondisi Aman**: Header respons memuat `Location: /dashboard` atau `Location: /home`.

---

## Referensi

- [OWASP Unvalidated Redirects and Forwards Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [MITRE CWE-601](https://cwe.mitre.org/data/definitions/601.html)
- [PortSwigger: Open Redirection](https://portswigger.net/web-security/open-redirection)
