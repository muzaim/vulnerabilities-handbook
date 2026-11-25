# Unrestricted Resource Consumption (Konsumsi Sumber Daya Tanpa Batasan)

> **Metadata Keamanan:**
> - **CWE**: [CWE-400: Uncontrolled Resource Consumption](https://cwe.mitre.org/data/definitions/400.html) & [CWE-770: Allocation of Resources Without Limits or Throttling](https://cwe.mitre.org/data/definitions/770.html)
> - **OWASP**: [API4:2023 – Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)
> - **Tingkat Keparahan (Severity)**: High (CVSS ~ 7.5 - 8.2)

---

## 1. Pengertian

**Unrestricted Resource Consumption** adalah celah keamanan pada API di mana server tidak membatasi jumlah sumber daya komputasi (memori RAM, CPU, kapasitas disk, batas request, atau ukuran query) yang dapat diminta oleh seorang klien dalam satu permintaan atau dalam rentang waktu tertentu.

Celah ini memungkinkan penyerang dengan sengaja mengirimkan permintaan yang sangat rakus sumber daya (seperti meminta 1 juta data sekaligus, atau mengunggah payload JSON raksasa berukuran 100MB), sehingga server kehabisan memori (*Out-of-Memory / OOM*), CPU mencapai 100%, dan layanan lumpuh bagi seluruh pengguna sah (*Denial of Service*).

**Analogi Sederhana:**  
Bayangkan sebuah restoran prasmanan yang bertuliskan: *"Gratis tambah nasi sepuasnya"*.  
Namun restoran tidak memiliki aturan batasan satu piring per orang. Seorang pengunjung iseng datang dan meminta kasir memasukkan 50 karung beras langsung ke dalam koper pribadinya. Akibatnya, dapur restoran kehabisan stok beras seketika dan ratusan pelanggan lain yang mengantre di belakang tidak bisa makan siang.

---

## 2. Cara Kerja

Setiap permintaan yang diproses oleh API membutuhkan alokasi sumber daya:
1. Alokasi memori RAM untuk membedah (*parse*) body JSON.
2. Waktu komputasi CPU dan koneksi database untuk mengeksekusi kueri.
3. Bandwidth jaringan untuk mengirimkan respons.

Celah ini terjadi saat developer tidak menetapkan batas atas (*upper bound*):
- **Pagination Tanpa Batas**: Mengizinkan parameter `limit` atau `size` bernilai angka sembarang: `GET /api/v1/products?limit=1000000`. Server mencoba memuat 1 juta baris dari database ke dalam memori RAM sekaligus, memicu crash proses server (*OOM Kill*).
- **Ukuran Body Request Tanpa Batas**: Membiarkan parser body menerima payload puluhan megabyte, menghabiskan memori buffer server.
- **Operasi Kriptografi / Kompresi Berat Tanpa Pembatas**: Mengizinkan klien memicu kompresi zip file raksasa atau parsing ekspresi reguler rumit (*ReDoS*) secara berulang.

---

## 3. Penyebab

- **Ketiadaan Batas Maksimal Paginasi (Pagination Max Limit)**: Menyetel `limit = req.query.limit` secara mentah tanpa validasi `Math.min(limit, 100)`.
- **Parser Body JSON Tanpa Batas Ukuran**: Menggunakan `express.json()` tanpa opsi `{ limit: '100kb' }`, sehingga secara default menerima payload besar.
- **Ketiadaan Timeout pada Kueri Database**: Membiarkan kueri berat berjalan di database tanpa batas waktu eksekusi (*statement timeout*).
- **Ketiadaan Rate Limiting pada Endpoint Berat**: Membiarkan endpoint penghasil laporan PDF dipanggil 100 kali per detik oleh satu pengguna.

---

## 4. Skenario Serangan

1. **Eksplorasi Parameter Paginasi**: Penyerang melihat endpoint katalog produk: `GET /api/products?page=1&limit=20`.
2. **Penyisipan Nilai Ekstrem**: Penyerang mengubah nilai limit menjadi angka raksasa:  
   `GET /api/products?page=1&limit=500000`.
3. **Eksekusi Kueri Masif di Database**: Server menjalankan `SELECT * FROM products LIMIT 500000`. Database memproses kueri selama 30 detik dan mengirimkan ratusan megabyte data mentah ke backend.
4. **Kehabisan Memori Server (OOM Crash)**: Proses backend kehabisan alokasi heap memori RAM saat mencoba mengonversi data tersebut ke string JSON, menyebabkan service backend mati (*crash*) dan seluruh website tumbang (Error 502 Bad Gateway).

---

## 5. Dampak

- **Lumpuhnya Layanan (Denial of Service - DoS)**: Server tidak dapat melayani pengguna sah.
- **Pembengkakan Biaya Cloud (Financial Denial of Wallet)**: Jika aplikasi berjalan di arsitektur auto-scaling (seperti AWS Lambda atau Kubernetes Auto-scaler), serangan ini akan memicu lonjakan jumlah container secara otomatis, mengakibatkan tagihan komputasi cloud membengkak ratusan juta rupiah.
- **Database Degradation**: Kueri berat mengunci tabel database penting dan memperlambat seluruh proses bisnis lainnya.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint daftar data / pencarian yang mendukung pagination (`/api/items`, `/api/logs`).
- Endpoint pengunggahan berkas atau dokumen teks.
- Endpoint ekspor laporan (CSV, Excel, PDF generation).
- Endpoint GraphQL yang mengizinkan kueri bertingkat (*deep nested queries*).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem E-Commerce TokoGrosir:**  
TokoGrosir memiliki endpoint API untuk mengambil riwayat mutasi transaksi:
`GET /api/v1/transactions?size=50`

Developer memprogram backend Go:
```go
limit, _ := strconv.Atoi(r.URL.Query().Get("size"))
db.Limit(limit).Find(&transactions)
```

Seorang penyerang mengirim 10 request bersamaan dengan parameter:
`GET /api/v1/transactions?size=2147483647`

Server mencoba mengalokasikan memori untuk menampung miliaran baris transaksi. Penggunaan RAM server melonjak dari 500MB menjadi 16GB dalam waktu 3 detik. Kernel Linux memicu mekanisme *Out-of-Memory Killer* (`oom-killer`) dan mematikan proses web server TokoGrosir. Toko online tersebut tidak dapat diakses selama 45 menit saat jam sibuk belanja.

---

## 8. Cara Mendeteksi

- **Stress Testing / Fuzzing Parameter**: Kirimkan nilai parameter batas ekstrem: `limit=9999999`, `limit=-1`, `limit=0`.
- **Uji Ukuran Payload**: Kirim payload JSON berukuran 50MB ke endpoint POST. Jika server menerima tanpa menolak dengan status `413 Payload Too Large`, parser rentan.
- **Pemantauan Metrik APM**: Amati apakah ada lonjakan tajam pemakaian RAM dan CPU saat endpoint pencarian tertentu dipanggil.

---

## 9. Pencegahan

1. **Terapkan Batas Maksimal Paginasi yang Ketat (Wajib)**:  
   Tetapkan batas atas yang tidak bisa di-override oleh klien (misal maksimal 100 data per halaman).
2. **Batasi Ukuran Maksimal Body Request (Max Payload Size)**:  
   Batasi parser JSON (misal maksimal 1MB atau 100KB) dan tolak payload lebih besar dengan status `413 Payload Too Large`.
3. **Tetapkan Statement Timeout pada Database**:  
   Batasi kueri database maksimal berjalan 3–5 detik. Jika lebih dari itu, batalkan kueri secara otomatis untuk mencegah penguncian sumber daya.
4. **Terapkan Rate Limiting Spesifik per Endpoint**:  
   Untuk endpoint yang membutuhkan komputasi berat (seperti export PDF atau kompresi zip), batasi maksimal 2-5 request per menit per akun.

---

## 10. Penanganan

1. **Deploy Pembatas di Reverse Proxy Segera**: Konfigurasikan batas ukuran body di Nginx (`client_max_body_size 1m;`) dan batas koneksi rate limiting.
2. **Kunci Nilai Limit Paginasi di Kode**: Tambahkan satu baris validasi: `if limit > 100 { limit = 100 }`.
3. **Restart Service yang Mengalami Memory Leak**: Kembalikan ketersediaan layanan sistem terlebih dahulu.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Pembatasan Ukuran Body & Paginasi Maksimal)

```go
package main

import (
	"encoding/json"
	"net/http"
	"strconv"
)

const (
	DefaultPageSize = 20
	MaxPageSize     = 100              // Batas maksimal yang diizinkan sistem
	MaxBodyBytes    = 1024 * 1024      // Maksimal payload 1 Megabyte
)

// ✅ KODE AMAN: Paginasi dengan Batas Maksimal Mutlak
func GetProductsSecure(w http.ResponseWriter, r *http.Request) {
	pageSizeStr := r.URL.Query().Get("size")
	pageSize, err := strconv.Atoi(pageSizeStr)

	// Jika tidak diisi atau error, gunakan default
	if err != nil || pageSize <= 0 {
		pageSize = DefaultPageSize
	}

	// KUNCI KEAMANAN: Paksa batas maksimal agar klien tidak bisa meminta jutaan data!
	if pageSize > MaxPageSize {
		pageSize = MaxPageSize
	}

	// Lakukan kueri database dengan batas aman: db.Limit(pageSize)...
	w.Write([]byte("Menampilkan data dengan limit aman: " + strconv.Itoa(pageSize)))
}

// ✅ KODE AMAN: Membatasi ukuran body request untuk mencegah OOM
func CreateProductSecure(w http.ResponseWriter, r *http.Request) {
	// Batasi pembacaan body request maksimal 1MB menggunakan http.MaxBytesReader
	r.Body = http.MaxBytesReader(w, r.Body, MaxBodyBytes)

	var payload struct {
		Name  string  `json:"name"`
		Price float64 `json:"price"`
	}

	if err := json.NewDecoder(r.Body).Decode(&payload); err != nil {
		http.Error(w, "Payload terlalu besar (maksimal 1MB) atau format tidak valid", http.StatusRequestEntityTooLarge)
		return
	}

	w.Write([]byte("Produk berhasil dibuat"))
}
```

### B. Contoh dalam TypeScript (Express Middleware & Query Limiter)

```typescript
import express, { Request, Response } from 'express';

const app = express();

// ✅ KODE AMAN: Batasi ukuran JSON parser maksimal 500KB
app.use(express.json({ limit: '500kb' }));

const MAX_PAGE_SIZE = 100;
const DEFAULT_PAGE_SIZE = 20;

// ✅ KODE AMAN: Paginasi dengan pembatas nilai maksimal
app.get('/api/transactions', async (req: Request, res: Response) => {
  let size = parseInt(req.query.size as string, 10);

  if (isNaN(size) || size <= 0) {
    size = DEFAULT_PAGE_SIZE;
  }

  // Paksa agar tidak melebihi batas maksimal yang aman bagi memori server
  const safeLimit = Math.min(size, MAX_PAGE_SIZE);

  const transactions = await prisma.transaction.findMany({
    take: safeLimit, // Mengambil maksimal 100 baris saja
  });

  return res.json({ count: transactions.length, limit: safeLimit, data: transactions });
});
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Parameter Paginasi Maksimal
Kirim request dengan nilai limit 999999:
```bash
curl "http://localhost:8080/api/products?size=999999"
```
- **Kondisi Rentan**: Server memproses kueri selama bermenit-menit atau crash dengan error `Out of Memory`.
- **Kondisi Aman**: Respons kembali seketika dengan `limit: 100` (sistem secara otomatis memotong angka ekstrem menjadi batas maksimal 100).

---

## Referensi

- [OWASP API Security Top 10 - API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)
- [MITRE CWE-400: Uncontrolled Resource Consumption](https://cwe.mitre.org/data/definitions/400.html)
- [Express.js Body-Parser Limits Documentation](https://expressjs.com/en/resources/middleware/body-parser.html)
