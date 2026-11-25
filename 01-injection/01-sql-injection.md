# SQL Injection (SQLi)

> **Metadata Keamanan:**
> - **CWE**: [CWE-89: Improper Neutralization of Special Elements used in an SQL Command](https://cwe.mitre.org/data/definitions/89.html)
> - **OWASP**: [A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 9.0 - 10.0)

---

## 1. Pengertian

**SQL Injection (SQLi)** adalah celah keamanan di mana pengguna luar dapat menyisipkan perintah SQL ilegal ke dalam kueri database yang dijalankan oleh aplikasi.

**Analogi Sederhana:**  
Bayangkan Anda mengisi formulir pengambilan barang di sebuah gudang. Di formulir tertulis:  
`Nomor Kotak: [ ... ]`

Orang biasa akan menulis: `123`. Maka petugas gudang mengambil kotak nomor 123.  
Namun, orang jahat menulis: `123 ATAU Buka semua kotak yang ada di gudang`.  
Jika petugas gudang membaca formulir itu secara mentah-mentah tanpa memverifikasi apa itu "nomor kotak", petugas akan membuka seluruh kotak gudang dan memberikannya kepada orang tersebut. Petugas gudang gagal memisahkan antara **nomor kotak** (data) dan **aturan kerja gudang** (perintah).

---

## 2. Cara Kerja

Database berkomunikasi dengan aplikasi menggunakan bahasa SQL (*Structured Query Language*). Ketika aplikasi ingin mencari data, aplikasi merangkai kalimat SQL:

```text
SELECT * FROM users WHERE email = 'input_user' AND password = 'input_password';
```

Jika aplikasi membuat kalimat SQL ini dengan cara menggabungkan teks (*string concatenation*) secara langsung tanpa pengamanan, karakter khusus SQL seperti tanda kutip satu (`'`) akan mengubah logika kalimat.

```
Pengguna mengetik: admin' --
Kalimat SQL menjadi:
SELECT * FROM users WHERE email = 'admin' --' AND password = '...';
                                          ^^
                                   Tanda -- mengabaikan sisa perintah password!
```

Database mengira kalimat tersebut adalah satu instruksi sah, mengeksekusi pencarian pengguna `admin`, dan mengabaikan pemeriksaan kata sandi.

---

## 3. Penyebab

Akar masalah SQL Injection selalu bersumber dari hal yang sama:
- **Penggabungan String Mentah**: Developer menggunakan format string (seperti `fmt.Sprintf` di Go atau *template literal* `` `...${input}...` `` di JavaScript) untuk menyusun perintah SQL.
- **Ketiadaan Parameterized Queries (Prepared Statements)**: Developer tidak menggunakan fitur bawaan database driver yang memisahkan struktur SQL dari nilai data.
- **Asumsi Kepercayaan Input**: Menganggap data dari form, dropdown, atau cookie selalu aman dan tidak bisa dimanipulasi oleh klien.

---

## 4. Skenario Serangan

1. **Observasi**: Penyerang membuka halaman pencarian produk di `https://toko.example/search?q=laptop`.
2. **Penyisipan Karakter Uji**: Penyerang menambahkan tanda petik tunggal (`'`) pada parameter: `?q=laptop'`.
3. **Analisis Respons**: Aplikasi mengembalikan pesan galat database: `Error: You have an error in your SQL syntax near ''laptop'''`. Penyerang mengetahui aplikasi menggabungkan string mentah.
4. **Eksploitasi Logika**: Penyerang mengirimkan payload `laptop' OR 1=1 --`. Aplikasi mengembalikan seluruh data produk termasuk produk yang belum dirilis atau disembunyikan.
5. **Ekstraksi Data Sensitif (UNION-based / Blind)**: Penyerang menggunakan kueri bertingkat untuk mengambil username, email, dan hash password dari tabel pengguna.

---

## 5. Dampak

- **Bagi Pengguna**: Akun dapat diambil alih, data pribadi (nama, alamat, nomor telepon, kartu identitas) bocor ke publik.
- **Bagi Perusahaan**: Kerugian reputasi masif, tuntutan hukum pelanggaran privasi data (UU PDP / GDPR), potensi kerugian finansial akibat manipulasi saldo atau pesanan.
- **Bagi Sistem**: Penyerang dapat membaca seluruh isi database, mengubah data saldo/stok, menghapus tabel penting (`DROP TABLE`), bahkan pada database tertentu dapat mengeksekusi perintah server sistem operasi.

---

## 6. Bagian Sistem yang Terdampak

- Formulir login, registrasi, dan lupa password.
- Kotak pencarian produk atau filter data (kategori, harga, tanggal).
- Parameter ID pada URL (misalnya `/api/orders?id=101`).
- Nilai di dalam HTTP Header (User-Agent, Cookie) jika dicatat langsung ke database dengan kueri mentah.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem E-Commerce TokoKeren:**  
TokoKeren memiliki fitur pelacakan pesanan publik. Pengguna memasukkan `nomor_resi` untuk melihat status pengiriman.

Kode backend TokoKeren mencari status dengan kueri:
```sql
SELECT status, alamat, total_harga FROM pesanan WHERE resi = '<input_user>';
```

Penyerang memasukkan input:
```text
NONEXISTENT' UNION SELECT username, password_hash, no_rekening FROM pengguna --
```

Kueri yang dieksekusi database menjadi gabungan dua tabel (*UNION*). Alih-alih menampilkan riwayat paket pengiriman, layar pelacakan TokoKeren menampilkan daftar akun pelanggan lengkap beserta hash password dan data perbankan mereka.

---

## 8. Cara Mendeteksi

- **Static Application Security Testing (SAST)**: Gunakan alat audit kode otomatis (seperti `gosec` untuk Go, `eslint-plugin-security` untuk TypeScript, atau SonarQube) yang mendeteksi penggunaan penggabungan string pada kueri database.
- **Dynamic Analysis (DAST) / Web Scanner**: Gunakan pemindai seperti OWASP ZAP untuk menyuntikkan payload SQL umum secara otomatis.
- **Manual Code Review**: Cari semua kemunculan fungsi eksekusi SQL (misalnya `db.Query()`, `client.query()`) dan pastikan tidak ada tanda `+`, `fmt.Sprintf`, atau template string di dalamnya.
- **Audit Database Log**: Pantau kueri dengan anomali pola sintaks seperti `UNION SELECT`, `OR 1=1`, atau pesan error sintaks berulang.

---

## 9. Pencegahan

1. **Gunakan Parameterized Queries / Prepared Statements (Wajib)**: Ini adalah standar industri mutlak. Database driver akan mengirim struktur kueri dan data secara terpisah. Database memperlakukan input semurni-murninya sebagai data teks biasa, bukan kode instruksi.
2. **Gunakan ORM (Object-Relational Mapping)**: Framework seperti GORM (Go) atau Prisma / TypeORM / Drizzle (TypeScript) secara default menggunakan parameterized queries untuk operasi standar.
3. **Prinsip Hak Akses Terkecil (Least Privilege)**: Akun database yang digunakan aplikasi web tidak boleh memiliki hak `SUPERUSER` atau `DROP DATABASE`.

---

## 10. Penanganan

Jika kerentanan SQL Injection ditemukan pada sistem produksi:
1. **Mitigasi Cepat (WAF Rule)**: Pasang aturan Web Application Firewall (WAF) darurat untuk memblokir pola SQLi umum pada endpoint yang bermasalah.
2. **Deploy Patch Segera**: Ganti kueri gabungan string dengan *prepared statement* dan lakukan rilis *hotfix*.
3. **Investigasi Log Forensik**: Periksa access log database untuk mencari tahu data apa saja yang sempat diakses atau dicuri oleh penyerang.
4. **Rotasi Kredensial**: Jika tabel kredensial atau session database sempat diakses, paksa *reset password* pengguna dan lakukan rotasi token sesi.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/lib/pq"
)

// ❌ KODE RENTAN (Vulnerable)
// String input pengguna digabungkan langsung dengan fmt.Sprintf.
func GetUserVulnerable(db *sql.DB, email string) (*sql.Rows, error) {
	query := fmt.Sprintf("SELECT id, name, email FROM users WHERE email = '%s'", email)
	// Jika email = "admin@example.com' OR '1'='1", seluruh user akan terbaca!
	return db.Query(query)
}

// ✅ KODE AMAN (Secure)
// Menggunakan placeholder ($1 pada PostgreSQL atau ? pada MySQL).
// Database membedakan dengan tegas antara perintah SQL dan nilai data email.
func GetUserSecure(db *sql.DB, email string) (*sql.Row, error) {
	query := "SELECT id, name, email FROM users WHERE email = $1"
	row := db.QueryRow(query, email)
	return row, nil
}
```

### B. Contoh dalam TypeScript / JavaScript (Node.js pg / postgres)

```typescript
import { Client } from 'pg';

const client = new Client();

// ❌ KODE RENTAN (Vulnerable)
// Menggunakan template literal JavaScript `${...}`.
async function searchProductVulnerable(keyword: string) {
  const query = `SELECT id, name, price FROM products WHERE name LIKE '%${keyword}%'`;
  // Jika keyword = "baju%' UNION SELECT id, password, 0 FROM users --"
  const result = await client.query(query);
  return result.rows;
}

// ✅ KODE AMAN (Secure)
// Menggunakan parameterized query dengan array parameter $1.
async function searchProductSecure(keyword: string) {
  const query = 'SELECT id, name, price FROM products WHERE name LIKE $1';
  // Nilai dibungkus murni sebagai data parameter
  const searchPattern = `%${keyword}%`;
  const result = await client.query(query, [searchPattern]);
  return result.rows;
}
```

**Perbedaan Utama:**  
Pada kode rentan, input pengguna diperlakukan sebagai bagian dari instruksi bahasa SQL. Pada kode aman, string kueri dikirim terlebih dahulu ke database untuk dikompilasi strukturnya, baru kemudian nilai input dikirim sebagai nilai variabel semata.

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan Unit Test (Go)
```go
func TestSQLInjectionProtection(t *testing.T) {
	db := setupTestDB(t)
	maliciousInput := "nobody@example.com' OR 1=1 --"

	row, err := GetUserSecure(db, maliciousInput)
	if err != nil && err != sql.ErrNoRows {
		t.Fatalf("Unexpected error: %v", err)
	}

	var id int
	var name, email string
	scanErr := row.Scan(&id, &name, &email)

	// Harusnya tidak ada baris yang cocok, karena tidak ada user dengan email persis seperti payload
	if scanErr != sql.ErrNoRows {
		t.Errorf("Rentang SQL Injection! Ditemukan user: %s padahal input adalah payload berbahaya", email)
	}
}
```

### Verifikasi Manual Menggunakan cURL
Kirimkan karakter khusus SQL pada endpoint pencarian lokal:
```bash
curl -X GET "http://localhost:8080/api/products?search=tas%27%20OR%201=1%20--"
```
- **Kondisi Rentan**: Sistem mengembalikan 200 OK dengan seluruh daftar produk, atau 500 Internal Server Error dengan bocoran error SQL syntax.
- **Kondisi Aman**: Sistem mengembalikan 200 OK dengan hasil array kosong `[]` (karena tidak ada produk bernama tepat seperti string tersebut) tanpa error database.

---

## Referensi

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [MITRE CWE-89 Definition](https://cwe.mitre.org/data/definitions/89.html)
- [PortSwigger Web Security Academy: SQL Injection](https://portswigger.net/web-security/sql-injection)
