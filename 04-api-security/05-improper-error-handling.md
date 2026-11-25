# Improper Error Handling (Penanganan Galat yang Tidak Tepat)

> **Metadata Keamanan:**
> - **CWE**: [CWE-209: Generation of Error Message Containing Sensitive Information](https://cwe.mitre.org/data/definitions/209.html)
> - **OWASP**: [A05:2021 – Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
> - **Tingkat Keparahan (Severity)**: Low / Medium (CVSS ~ 4.3 - 6.5)

---

## 1. Pengertian

**Improper Error Handling** adalah kelemahan di mana aplikasi web atau API menampilkan pesan kesalahan sistem secara mentah (*verbose stack trace*, pesan galat database SQL, jalur direktori internal server, atau versi library) langsung ke layar peramban atau respons JSON pengguna saat terjadi kesalahan eksekusi.

Informasi ini menjadi tambang emas bagi penyerang (*reconnaissance*) untuk memetakan arsitektur teknologi backend, nama tabel database, hingga versi pustaka yang memiliki celah keamanan publik yang belum di-patch.

**Analogi Sederhana:**  
Bayangkan sebuah mesin ATM bank yang mengalami gangguan saat menarik uang.  
- **Penanganan Aman**: Layar menampilkan: *"Maaf, terjadi gangguan teknis pada transaksi Anda. Silakan hubungi call center dengan kode referensi: ERR-991"*.
- **Penanganan Tidak Tepat**: Layar mesin ATM mencetak cetak biru struktur brankas mesin: *"Error: Kunci gembok merk XYZ tipe 201 mengalami aus pada baut nomor 4 di koordinat laci bawah, sistem operasi Windows XP build 2600"*. Seorang pencuri yang membaca layar tersebut langsung tahu kelemahan fisik mesin ATM tersebut.

---

## 2. Cara Kerja

Ketika terjadi kesalahan di kode backend (misalnya database mati, nilai `null` diakses, atau sintaks SQL salah):
1. Runtime bahasa pemrograman menghasilkan objek Exception / Stack Trace yang memuat:
   - Nama file dan nomor baris kode tempat terjadinya error (`/var/app/backend/services/payment.js:42`).
   - String kueri SQL mentah beserta nama kolom dan tabel.
   - Versi framework (Express, Spring, Go, Python).
   - Terkadang memuat nilai variabel lokal (kunci API atau password yang gagal divalidasi).
2. Jika developer tidak memasang penanganan kesalahan terpusat (*Global Error Handler*), web server secara otomatis mencetak seluruh pesan mentah tersebut ke dalam respons HTTP 500.

---

## 3. Penyebab

- **Menjalankan Server Produksi dalam Mode 'Development'**: Lupa menyetel `NODE_ENV=production` atau `DEBUG=False`.
- **Ketiadaan Global Error Handling Middleware**: Membiarkan exception yang tidak tertangkap (*uncaught exception*) mengalir bebas ke respons HTTP klien.
- **Mencetak `err.Error()` atau `err.stack` Langsung ke Respons Klien**: Menuliskan pesan error mentah ke `res.status(500).json({ error: err.message, stack: err.stack })`.

---

## 4. Skenario Serangan

1. **Pemicu Error Sengaja**: Penyerang mengirimkan input tipe data yang salah ke API: `POST /api/orders` dengan body `{"quantity": "BUKAN_ANGKA"}` atau menyisipkan karakter petik tunggal (`'`).
2. **Inspeksi Respons Galat**: Server mengembalikan status 500 Internal Server Error dengan respons JSON:
   ```json
   {
     "error": "pq: syntax error at or near \"'\"",
     "query": "SELECT * FROM orders WHERE user_id = 12 AND status = 'draft'",
     "stack": "at Query.run (/var/www/app/node_modules/pg/lib/query.js:142:12)...",
     "server": "Node v18.12.0 Linux 5.15.0-generic"
   }
   ```
3. **Pemanfaatan Informasi**:
   - Penyerang kini tahu sistem menggunakan database **PostgreSQL** (`pq:`).
   - Penyerang mengetahui nama tabel dan kolom (`orders`, `user_id`, `status`).
   - Penyerang mengetahui jalur direktori fisik server (`/var/www/app/`).
   - Penyerang mengetahui versi Node.js yang berjalan, lalu mencari celah keamanan CVE spesifik untuk versi tersebut.

---

## 5. Dampak

- **Mempermudah Eksploitasi Tingkat Lanjut**: Membantu penyerang menyempurnakan payload SQL Injection, Path Traversal, atau SSRF karena struktur internal sistem sudah terbongkar.
- **Kebocoran Kunci Rahasia**: Dalam beberapa kasus fatal, stack trace mencetak variabel lingkungan (*environment variables*) yang memuat password database atau secret token.
- **Kerugian Reputasi Profesional**: Menampilkan pesan error mentah merusak citra profesionalitas sistem di mata pengguna.

---

## 6. Bagian Sistem yang Terdampak

- Seluruh endpoint API yang menangani masukan data dari pengguna.
- Halaman galat bawaan server (Halaman 404 dan 500 standar Nginx/Apache/Tomcat).
- Logika parsing body JSON atau pengunggahan berkas.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Pelaporan Pajak KantorPajak Online:**  
Wajib pajak memasukkan nomor NPWP di kolom pencarian.

Seorang penyerang memasukkan input karakter khusus. Aplikasi backend Java Spring Boot mengalami crash dan menampilkan halaman *Whitelabel Error Page*:
```text
org.hibernate.exception.SQLGrammarException: could not extract ResultSet
Table 'dbtax_prod.tbl_wajib_pajak_2024' doesn't exist
Path: /opt/production/tax-gateway/src/main/java/...
DB Connection String: jdbc:mysql://10.0.1.50:3306/dbtax_prod?user=root
```

Pesan error tersebut membocorkan nama database produksi (`dbtax_prod`), skema tabel, dan alamat IP internal server database (`10.0.1.50`) beserta username `root`. Penyerang kini memiliki target IP dan kredensial yang jelas untuk menyerang jaringan internal kantor pajak.

---

## 8. Cara Mendeteksi

- **Pemicu Error Manual**: Kirimkan JSON tidak lengkap (malformed JSON seperti `{"name": ` tanpa penutup) atau tipe data yang salah (`id: true`).
- **Inspeksi Respons**: Amati apakah respons mengandung kata kunci error database (`syntax error`, `ORA-`, `SQLSTATE`, `Uncaught TypeError`, file path `/var/www/` atau `C:\inetpub`).
- **Pemeriksaan Header Server**: Periksa apakah header `X-Powered-By` membocorkan versi framework (misal `Express`).

---

## 9. Pencegahan

1. **Pisahkan Log Internal dan Respons Pengguna (Wajib)**:  
   Catat pesan error lengkap dan stack trace ke sistem log internal (seperti file log, Datadog, Sentry, atau Elasticsearch). Tampilkan HANYA pesan generik yang ramah ke pengguna (misal: *"Terjadi kesalahan internal pada server"*).
2. **Gunakan Nomor Tiket Pelacakan Acak (Error ID / Tracking ID)**:  
   Hasilkan kode UUID acak saat terjadi error (misal: `ERR-550e8400`). Tampilkan UUID ini ke pengguna dan simpan UUID yang sama di log server, sehingga pengguna dapat melapor ke tim support tanpa membocorkan detail teknis.
3. **Pastikan Mode Produksi Aktif**:  
   Setel `NODE_ENV=production` pada runtime Node.js untuk mematikan stack trace otomatis dari framework Express.
4. **Matikan Banner Versi Server**:  
   Hapus header `X-Powered-By` dan sembunyikan versi Nginx/Apache (`server_tokens off;`).

---

## 10. Penanganan

1. **Deploy Global Error Handler Segera**: Tambahkan middleware penangkap error di akhir alur routing backend.
2. **Aktifkan `NODE_ENV=production`**: Pastikan environment variable server disetel dengan benar.
3. **Periksa Kustomisasi Halaman Error**: Ganti halaman error default web server dengan halaman HTML statis yang bersih.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Centralized Error Handler dengan Tracking ID)

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"log"
	"net/http"
)

type PublicErrorResponse struct {
	ErrorID string `json:"error_id"`
	Message string `json:"message"`
}

func generateErrorID() string {
	b := make([]byte, 8)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// ✅ KODE AMAN: Pembungkus handler dengan penanganan galat terpusat
func SafeHandler(w http.ResponseWriter, r *http.Request) {
	err := doDatabaseOperation()
	if err != nil {
		errorID := generateErrorID()

		// 1. Catat detail teknis error lengkap HANYA ke log internal server
		log.Printf("[ERROR-ID: %s] Gagal mengeksekusi kueri: %v | Path: %s", errorID, err, r.URL.Path)

		// 2. Kirim pesan generik yang bersih ke klien luar beserta Error ID
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusInternalServerError)

		resp := PublicErrorResponse{
			ErrorID: errorID,
			Message: "Terjadi gangguan pada sistem kami. Silakan hubungi layanan bantuan dengan menyertakan Error ID Anda.",
		}
		json.NewEncoder(w).Encode(resp)
		return
	}

	w.Write([]byte("Operasi sukses"))
}

func doDatabaseOperation() error {
	// Simulasi kegagalan database
	return http.ErrHandlerTimeout
}
```

### B. Contoh dalam TypeScript (Express Global Error Middleware)

```typescript
import express, { Request, Response, NextFunction } from 'express';
import crypto from 'crypto';

const app = express();

// Matikan header pengenal teknologi
app.disable('x-powered-by');

// Router aplikasi...
app.get('/api/data', (req, res) => {
  throw new Error('Koneksi ke database PostgreSQL di 10.0.1.50 timeout!');
});

// ✅ KODE AMAN: Global Error Handling Middleware (Diletakkan paling akhir di aplikasi Express)
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  const errorRef = crypto.randomUUID();

  // 1. Catat stack trace lengkap HANYA di server console/monitoring
  console.error(`[ERROR REF: ${errorRef}]`, err.stack);

  // 2. Kembalikan respons yang bersih tanpa rincian teknis ke pengguna
  return res.status(500).json({
    status: 'error',
    errorRef: errorRef,
    message: 'Terjadi kesalahan internal pada server kami. Silakan coba beberapa saat lagi.',
    // JANGAN PERNAH menyertakan: stack: err.stack atau rawError: err.message di environment produksi!
  });
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Respon Error
Kirimkan permintaan sengaja salah untuk memicu error 500:
```bash
curl -s http://localhost:8080/api/data | jq .
```
- **Kondisi Rentan**: Respons memuat kata `stack`, `node_modules`, `SQL`, atau file path internal.
- **Kondisi Aman**: Respons hanya memuat format bersih generik:
  ```json
  {
    "status": "error",
    "errorRef": "550e8400-e29b-41d4-a716-446655440000",
    "message": "Terjadi kesalahan internal pada server kami..."
  }
  ```

---

## Referensi

- [OWASP Error Handling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html)
- [MITRE CWE-209: Information Exposure Through an Error Message](https://cwe.mitre.org/data/definitions/209.html)
- [Express.js Error Handling Best Practices](https://expressjs.com/en/guide/error-handling.html)
