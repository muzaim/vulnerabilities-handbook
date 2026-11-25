# Credential Stuffing

> **Metadata Keamanan:**
> - **CWE**: [CWE-307: Improper Restriction of Excessive Authentication Attempts](https://cwe.mitre.org/data/definitions/307.html) & [CWE-799: Improper Control of Interaction Frequency](https://cwe.mitre.org/data/definitions/799.html)
> - **OWASP**: [A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/) / [OAT-008: Credential Stuffing](https://owasp.org/www-project-automated-threats-to-web-applications/assets/files/oat-008/)
> - **Tingkat Keparahan (Severity)**: High (CVSS ~ 7.5 - 8.5)

---

## 1. Pengertian

**Credential Stuffing** adalah serangan otomatisasi di mana penyerang menggunakan kumpulan jutaan pasangan username/email dan kata sandi yang bocor dari situs web lain (*data breach leak*) untuk mencoba masuk secara massal ke dalam sistem aplikasi target.

**Perbedaan Kunci dengan Brute Force Tradisional:**
- **Brute Force**: Menargetkan **1 akun spesifik** dengan mencoba **ribuan kemungkinan kata sandi**.
- **Credential Stuffing**: Menggunakan **jutaan kombinasi username & kata sandi yang valid** dari kebocoran data lain, lalu mencobanya **sekali saja per akun** pada sistem target dengan asumsi bahwa banyak pengguna menggunakan kata sandi yang sama di berbagai situs.

**Analogi Sederhana:**  
Bayangkan seorang pencuri menemukan satu karung berisi ribuan anak kunci yang tercecer dari sebuah apartemen di seberang jalan.  
Pencuri itu berjalan di kompleks perumahan Anda dan mencoba memasukkan anak kunci tersebut satu per satu ke setiap pintu rumah warga. Jika ada warga yang malas dan menggunakan gembok pintu dengan kunci duplikat yang sama seperti apartemen seberang, rumah tersebut akan langsung terbuka.

---

## 2. Cara Kerja

Kebiasaan buruk manusia adalah menggunakan kata sandi yang sama persis (*password reuse*) untuk email, media sosial, e-commerce, hingga aplikasi perbankan.

1. Situs web X mengalami kebocoran data (*data breach*), dan database berisi 10 juta email serta password bocor di dark web.
2. Penyerang mengunduh database bocoran tersebut (*combo list*).
3. Penyerang memprogram bot otomatis yang menggunakan ribuan alamat IP perantara (proxy / botnet) untuk menyamarkan asal serangan.
4. Bot mencoba login ke situs web Y (situs Anda) dengan kecepatan tinggi. Setiap username hanya dicoba 1 kali, sehingga tidak memicu batasan "5 kali gagal per akun".
5. Jika ada 1% saja pengguna yang menggunakan password yang sama, penyerang berhasil membajak 100.000 akun secara instan.

---

## 3. Penyebab

- **Kebiasaan Pengguna Mendaur Ulang Kata Sandi**: Pengguna menggunakan kombinasi email dan password yang identik di banyak platform.
- **Ketiadaan Pengecekan Kata Sandi yang Bocor (Pwned Passwords Check)**: Sistem mengizinkan pengguna mendaftar atau menggunakan password yang sudah diketahui umum pernah bocor.
- **Ketiadaan Deteksi Botnet Terdistribusi**: Rate limiting hanya didasarkan pada alamat IP tunggal, sehingga mudah diakali oleh penyerang yang merotasi ribuan alamat IP proxy.
- **Ketiadaan Multi-Factor Authentication (MFA)**: Sistem mengizinkan login penuh hanya dengan modal password.

---

## 4. Skenario Serangan

1. **Persiapan Data**: Penyerang mengunduh file `combo_list.txt` berisi 500.000 pasang kredensial.
2. **Konfigurasi Proxy**: Penyerang menyewa jaringan *residential proxy* sehingga setiap request login berasal dari alamat IP dan negara yang berbeda-beda.
3. **Eksekusi Penembakan**: Bot menembakkan request ke endpoint `POST /api/login`.
4. **Validasi Sukses**: Bot memilah akun yang berhasil login (status 200 OK), mencatat saldo akun atau kartu kredit tersimpan di akun tersebut, lalu menjual akun-akun tersebut di pasar gelap (*black market*).

---

## 5. Dampak

- **Pembajakan Akun Massal (Mass Account Takeover)**: Ribuan akun pelanggan dapat dikuasai dalam hitungan jam.
- **Pencurian Poin Loyalitas & Saldo**: Penggunaan saldo e-wallet atau penukaran poin reward tanpa sepengetahuan pemilik asli.
- **Beban Infrastruktur Luar Biasa**: Jutaan request bot membanjiri server autentikasi, menyebabkan kelambatan (*latency*) bagi pengguna sah.
- **Denda Regulasi & Hilangnya Kepercayaan**: Publik mengira database sistem Anda yang telah diretas, padahal kebocoran aslinya berasal dari platform lain.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint login web dan mobile API.
- Endpoint Single Sign-On (SSO).
- Halaman validasi akun atau reward point.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Layanan Streaming Film NontonYuk:**  
NontonYuk adalah platform streaming video berlangganan bulanan. Penyerang melancarkan serangan credential stuffing menggunakan 2 juta kredensial hasil kebocoran situs forum game lama.

Karena NontonYuk tidak menerapkan proteksi bot dan tidak menggunakan MFA, sebanyak 15.000 akun pelanggan yang menggunakan password yang sama berhasil ditembus. Penyerang mengumpulkan akun-akun aktif tersebut dan menjualnya di media sosial seharga Rp10.000 per akun sebagai "Akun Premium Murah". Pelanggan asli komplain karena riwayat tontonan mereka berubah dan akun mereka sering terputus (*kicked*) karena batas kuota streaming tercapai.

---

## 8. Cara Mendeteksi

- **Pola Sukses Login yang Sangat Rendah**: Rasio login berhasil turun drastis (misal dari normal 80% menjadi hanya 1-2%), sementara volume trafik login melonjak ribuan persen.
- **Trafik Login Terdistribusi**: Ribuan alamat IP unik yang masing-masing hanya mencoba login 1 atau 2 kali dengan username yang berbeda-beda.
- **User-Agent Header yang Tidak Wajar**: Permintaan datang dari pustaka otomatisasi (Python `requests`, `curl`, Go-http-client) atau User-Agent browser usang.

---

## 9. Pencegahan

1. **Integrasi Pemeriksaan Kata Sandi Bocor (Have I Been Pwned / k-Anonymity API)**: Saat pengguna mendaftar atau mengganti kata sandi, periksa apakah kata sandi tersebut ada dalam database kebocoran publik. Jika ada, tolak dan wajibkan pengguna memilih kata sandi lain.
2. **Wajibkan Multi-Factor Authentication (MFA)**: Meskipun penyerang memiliki pasangan email dan password yang tepat, mereka tidak bisa masuk tanpa kode OTP dari perangkat ponsel pengguna.
3. **Gunakan Proteksi Bot Cerdas (Behavioral Analysis / Turnstile / reCAPTCHA v3)**: Deteksi anomali perilaku browser, pergerakan kursor, dan sidik jari perangkat (*device fingerprinting*).
4. **Kirimkan Peringatan Login Baru**: Kirimkan email notifikasi: *"Login baru terdeteksi dari perangkat/lokasi yang tidak biasa"* disertai tombol untuk mengamankan akun seketika.

---

## 10. Penanganan

1. **Paksa Reset Password untuk Akun Terdampak**: Jika terdeteksi sekelompok akun berhasil dimasuki melalui serangan credential stuffing, paksa akun-akun tersebut untuk menyetel ulang password melalui email (*forced password reset*).
2. **Aktifkan Mode Proteksi Tinggi di WAF / Cloudflare**: Pasang tantangan interaktif (Managed Challenge) untuk seluruh trafik yang menuju endpoint `/login`.
3. **Putuskan Sesi Aktif**: Batalkan seluruh token sesi akun-akun yang terindikasi disusupi.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Integrasi Pwned Passwords API dengan k-Anonymity)

```go
package main

import (
	"crypto/sha1"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// ✅ KODE AMAN: Memeriksa apakah password pernah bocor di internet menggunakan k-Anonymity API (Have I Been Pwned)
// Model k-Anonymity sangat aman karena password tidak pernah dikirimkan ke pihak ketiga secara utuh!
func IsPasswordBreached(password string) (bool, error) {
	// 1. Hitung SHA-1 hash dari password
	hasher := sha1.New()
	hasher.Write([]byte(password))
	hash := strings.ToUpper(hex.EncodeToString(hasher.Sum(nil)))

	prefix := hash[:5]  // 5 karakter pertama
	suffix := hash[5:]  // Sisa karakter

	// 2. Kirim HANYA 5 karakter pertama ke API HIBP
	resp, err := http.Get(fmt.Sprintf("https://api.pwnedpasswords.com/range/%s", prefix))
	if err != nil {
		return false, err
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return false, err
	}

	// 3. Cari suffix di dalam respons lokal
	lines := strings.Split(string(body), "\r\n")
	for _, line := range lines {
		parts := strings.Split(line, ":")
		if len(parts) == 2 && parts[0] == suffix {
			// Password ditemukan dalam daftar kebocoran publik!
			return true, nil
		}
	}

	return false, nil
}
```

### B. Contoh dalam TypeScript (Express & Cloudflare Turnstile Verification)

```typescript
import { Request, Response, NextFunction } from 'express';

// ✅ KODE AMAN: Middleware Verifikasi Bot Token (Cloudflare Turnstile)
export async function verifyTurnstileToken(req: Request, res: Response, next: NextFunction) {
  const token = req.body['cf-turnstile-response'];
  const ip = req.headers['cf-connecting-ip'] || req.socket.remoteAddress;

  if (!token) {
    return res.status(400).json({ error: 'Tantangan keamanan bot wajib diselesaikan' });
  }

  // Verifikasi token ke API resmi Cloudflare
  const formData = new URLSearchParams();
  formData.append('secret', process.env.TURNSTILE_SECRET_KEY || '');
  formData.append('response', token);
  formData.append('remoteip', ip as string);

  try {
    const result = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
      body: formData,
      method: 'POST',
    });

    const outcome = await result.json();
    if (!outcome.success) {
      return res.status(403).json({ error: 'Validasi bot gagal, permintaan Anda ditolak' });
    }

    next();
  } catch (error) {
    return res.status(500).json({ error: 'Gagal memverifikasi proteksi keamanan' });
  }
}

// Router login dilindungi proteksi bot sebelum menyentuh verifikasi password:
// app.post('/api/login', verifyTurnstileToken, loginHandler);
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Integrasi k-Anonymity
Jalankan fungsi pengecekan password bocor dengan password umum:
```go
breached, _ := IsPasswordBreached("Password123!")
if breached {
    fmt.Println("Sistem berhasil mendeteksi dan menolak password yang pernah bocor!")
}
```
- **Verifikasi**: Sistem harus mengembalikan `true` untuk kata sandi populer dan menolak pendaftaran akun atau pembaruan kata sandi tersebut.

---

## Referensi

- [OWASP Automated Threats to Web Applications: OAT-008 Credential Stuffing](https://owasp.org/www-project-automated-threats-to-web-applications/assets/files/oat-008/)
- [Troy Hunt: Pwned Passwords (k-Anonymity)](https://www.troyhunt.com/ive-just-launched-pwned-passwords-v2/)
- [NIST Special Publication 800-63B: Verifying Against Compromised Credentials](https://pages.nist.gov/800-63-3/sp800-63b.html)
