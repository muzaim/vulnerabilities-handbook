# Hardcoded Secrets (Kunci Rahasia Tertanam di Kode)

> **Metadata Keamanan:**
> - **CWE**: [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)
> - **OWASP**: [A05:2021 – Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/) / [A02:2021 – Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.8)

---

## 1. Pengertian

**Hardcoded Secrets** adalah praktik buruk di mana kunci rahasia kriptografi, kata sandi database, API key layanan pihak ketiga (seperti OpenAI, AWS, Stripe), atau token autentikasi ditulis langsung (*hardcoded*) sebagai teks mentah di dalam file kode sumber (*source code*).

Ketika kode tersebut diunggah ke repositori (seperti GitHub atau GitLab), dikompilasi ke dalam aplikasi mobile (Android APK / iOS IPA), atau dibagikan ke anggota tim lain, kunci rahasia tersebut ikut tersebar dan dapat dibaca oleh siapa saja.

**Analogi Sederhana:**  
Bayangkan Anda memasang brankas besi paling canggih di dunia di kamar tidur Anda.  
Namun, karena Anda takut lupa nomor kombinasi brankasnya, Anda menuliskan nomor kombinasi tersebut menggunakan spidol permanen hitam tebal tepat di atas pintu depan rumah Anda. Siapa pun yang lewat di depan rumah Anda dapat membaca kombinasi tersebut dan membuka brankas Anda.

---

## 2. Cara Kerja

Developer sering kali menaruh kredensial langsung saat tahap uji coba (*prototyping*):
```javascript
const dbPassword = "SuperSecretPassword2024!";
const stripeApiKey = "sk_test_dummy_key_12345";
```

Bagaimana kunci ini bisa jatuh ke tangan penyerang?
1. **Kebocoran Repositori Git**: Developer tidak sengaja melakukan `git push` ke repositori publik di GitHub. Bot otomatis di internet memindai repositori GitHub setiap detik dan menyedot kunci AWS/OpenAI dalam waktu kurang dari 3 menit setelah diunggah!
2. **Riwayat Commit Git (*Git History*)**: Sekalipun developer sudah menghapus kunci di commit terbaru, kunci tersebut masih tersimpan abadi di dalam riwayat commit Git sebelumnya (`git log`).
3. **Decompiling Aplikasi Mobile (APK)**: Kunci yang ditulis di aplikasi Android Java/Kotlin dapat dibaca dengan sangat mudah hanya dengan mengekstrak file APK menggunakan alat gratis seperti `jadx`.
4. **Terbawa ke File Bundle Frontend**: Kunci backend yang ditaruh di React/Vue ikut terkompilasi ke dalam file JavaScript publik (`main.bundle.js`).

---

## 3. Penyebab

- **Ketiadaan Penggunaan Environment Variables**: Malas membuat file `.env` dan memanggil `process.env` atau `os.Getenv()`.
- **Lupa Menambahkan `.env` ke `.gitignore`**: Mengunggah file `.env` lokal ke sistem kontrol versi.
- **Ketiadaan Alat Pemindai Otomatis (*Pre-commit Hook*)**: Tidak ada sistem linter yang otomatis mencegah commit jika terdeteksi ada pola string rahasia.

---

## 4. Skenario Serangan

1. **Pemindaian Publik GitHub**: Penyerang menggunakan bot pencari (seperti TruffleHog atau GitHub Search API) untuk mencari string kunci API sensitif (seperti token Google Cloud atau Stripe).
2. **Penemuan Repositori Karyawan**: Penyerang menemukan repositori tugas akhir atau skripsi seorang developer magang yang memuat potongan kode perusahaan.
3. **Penyedotan Kunci Cloud**: Di dalam file `config.go`, penyerang menemukan:
   `AWS_ACCESS_KEY_ID = "AKIA_CONTOH_ID_PALSU"` dan `AWS_SECRET_ACCESS_KEY = "CONTOH_SECRET_ACCESS_KEY_PALSU"`.
4. **Pembajakan Cloud**: Penyerang menggunakan AWS CLI untuk membuat puluhan server komputasi GPU berbiaya mahal untuk menambang cryptocurrency (*crypto mining*). Tagihan cloud perusahaan membengkak hingga ratusan juta rupiah dalam semalam.

---

## 5. Dampak

- **Pengambilalihan Akun Cloud Penuh**: Penyerang dapat menghapus infrastruktur, mencuri snapshot database, atau mematikan seluruh sistem.
- **Kerugian Finansial Akibat Tagihan API**: API Key pihak ketiga (OpenAI, Twilio, Google Maps) disedot penyerang untuk keperluan pribadi.
- **Pencurian Data Pengguna Masif**: Penyerang langsung terhubung ke database produksi menggunakan password yang bocor.

---

## 6. Bagian Sistem yang Terdampak

- Seluruh kode sumber aplikasi (backend, frontend, mobile).
- Repositori Git publik maupun privat.
- File konfigurasi deployment dan Dockerfile.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Pembayaran E-Commerce BelanjaPuas:**  
BelanjaPuas mengintegrasikan payment gateway Stripe untuk memproses kartu kredit.

Seorang developer frontend ingin menguji pembayaran, lalu menulis di file konfigurasi:
```javascript
export const STRIPE_SECRET_KEY = "sk_test_dummy_key_12345";
```
File ini dibundle oleh Webpack ke dalam file `app.js` yang diunduh oleh setiap pengunjung website.

Seorang penyerang membuka tab DevTools, mencari kata `STRIPE_SECRET_KEY`, dan menemukan kunci privat Stripe toko tersebut. Dengan kunci privat tersebut, penyerang dapat memanggil API pengembalian dana (*refund*) secara massal ke rekening penyerang sendiri, menguras saldo merchant BelanjaPuas hingga ratusan juta rupiah.

---

## 8. Cara Mendeteksi

- **Alat Pemindai Rahasia Otomatis (Secret Scanner)**:
  - **Gitleaks**: `gitleaks detect --source . -v`
  - **TruffleHog**: `trufflehog git file://. --since-commit HEAD~50`
  - **GitHub Secret Scanning**: Fitur bawaan GitHub yang otomatis memperingatkan jika ada token yang bocor.
- **Audit Manual Regex**: Cari kata kunci `password = "`, `secret = "`, `api_key = "` di codebase.

---

## 9. Pencegahan

1. **Gunakan Environment Variables (Wajib Mutlak)**:  
   Selalu baca kredensial dari lingkungan sistem (`process.env.DB_PASSWORD` atau `os.Getenv("DB_PASSWORD")`).
2. **Daftarkan File Sensitif ke `.gitignore` Sejak Hari Pertama**:  
   Pastikan file `.env`, `.env.local`, file sertifikat `.pem`, dan kunci `.key` tercantum di `.gitignore`.
3. **Gunakan Secret Manager Resmi di Tingkat Produksi**:  
   Gunakan layanan brankas rahasia terpusat seperti **HashiCorp Vault**, **AWS Secrets Manager**, atau **Google Secret Manager**. Kunci disuntikkan secara dinamis saat aplikasi dijalankan di cloud.
4. **Pasang Pre-commit Hook (Gitleaks / Husky)**:  
   Cegah proses `git commit` di komputer lokal developer jika terdapat string yang menyerupai format kunci rahasia.

---

## 10. Penanganan

1. **Anggap Kunci Sudah Terkompromi (Rotasi Segera!)**:  
   Sekali sebuah kunci masuk ke Git, **jangan hanya menghapusnya dari kode**. Kunci tersebut **WAJIB DIREVOKE DAN DIGANTI DENGAN KUNCI BARU** di dashboard penyedia layanan (AWS, Stripe, Database).
2. **Hapus Riwayat Git Secara Permanen**:  
   Gunakan alat seperti `git-filter-repo` atau BFG Repo-Cleaner untuk membersihkan string dari seluruh riwayat git masa lalu.
3. **Audit Log Penggunaan Kunci**:  
   Periksa audit log di penyedia cloud untuk melihat apakah kunci tersebut sempat digunakan untuk memanggil aksi mencurigakan.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"os"
	_ "github.com/lib/pq"
)

// ❌ KODE SANGAT RENTAN (Vulnerable)
// Password dan kredensial database ditulis langsung di dalam kode!
const (
	HardcodedDBUser = "postgres"
	HardcodedDBPass = "PasswordRahasia12345!" // Bocor ke siapapun yang membaca kode!
	HardcodedDBHost = "db.produksi.perusahaan.com"
)

// ✅ KODE AMAN (Secure)
// Membaca kredensial HANYA dari environment variables
func ConnectDatabaseSecure() (*sql.DB, error) {
	dbHost := os.Getenv("DB_HOST")
	dbUser := os.Getenv("DB_USER")
	dbPass := os.Getenv("DB_PASSWORD")
	dbName := os.Getenv("DB_NAME")

	if dbPass == "" || dbUser == "" {
		return nil, fmt.Errorf("kredensial database di environment variable belum dikonfigurasi")
	}

	connStr := fmt.Sprintf("host=%s user=%s password=%s dbname=%s sslmode=verify-full",
		dbHost, dbUser, dbPass, dbName)

	return sql.Open("postgres", connStr)
}
```

### B. Contoh dalam TypeScript (Node.js & `dotenv`)

```typescript
import dotenv from 'dotenv';
// Muat variabel dari file .env lokal (file .env WAJIB ada di .gitignore!)
dotenv.config();

// ❌ KODE SANGAT RENTAN (Vulnerable)
const STRIPE_SECRET = 'sk_test_dummy_key_12345'; // RENTAN!

// ✅ KODE AMAN (Secure)
// Ambil nilai dari Environment Variable dengan pengecekan keberadaan
const stripeApiKey = process.env.STRIPE_SECRET_KEY;

if (!stripeApiKey) {
  throw new Error('FATAL: Variabel STRIPE_SECRET_KEY belum diatur di environment server!');
}

export const stripeClient = new Stripe(stripeApiKey, {
  apiVersion: '2023-10-16',
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian dengan Gitleaks Lokal
Jalankan pemindaian otomatis di terminal repositori:
```bash
gitleaks detect --source . -v
```
- **Kondisi Rentan**: Gitleaks menemukan string secret dengan pesan `Finding: sk_test_... RuleID: stripe-api-key`.
- **Kondisi Aman**: Gitleaks melaporkan `No leaks found`.

---

## Referensi

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [MITRE CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)
- [Gitleaks: Automated Secret Scanner](https://github.com/gitleaks/gitleaks)
