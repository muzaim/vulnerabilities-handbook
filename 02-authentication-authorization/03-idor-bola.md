# IDOR & BOLA (Insecure Direct Object Reference / Broken Object Level Authorization)

> **Metadata Keamanan:**
> - **CWE**: [CWE-639: Authorization Bypass Through User-Controlled Key](https://cwe.mitre.org/data/definitions/639.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) / [API1:2023 – Broken Object Level Authorization (BOLA)](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/) *(Peringkat #1 OWASP API Top 10)*
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.1)

---

## 1. Pengertian

**IDOR** (*Insecure Direct Object Reference*) dan **BOLA** (*Broken Object Level Authorization*) merujuk pada konsep celah keamanan yang sama persis: kondisi di mana aplikasi menerima pengenal objek langsung (seperti ID dokumen, ID pesanan, ID pengguna) dari permintaan pengguna dan langsung mengakses data tersebut di database **tanpa memeriksa apakah pengguna yang meminta memiliki hak atas objek tersebut**.

- **IDOR** adalah istilah klasik yang populer dalam keamanan aplikasi web tradisional.
- **BOLA** adalah istilah modern yang dipopulerkan oleh OWASP API Security Top 10 untuk menggambarkan IDOR pada endpoint API RESTful/GraphQL.

**Analogi Sederhana:**  
Bayangkan sebuah jasa laundry pakaian. Setiap pelanggan diberi nota dengan nomor antrean: `#105`.  
Ketika datang mengambil baju, Anda memberikan nota `#105` dan petugas memberikan kantong baju Anda.  
Namun, jika Anda iseng berkata: "Tolong ambilkan juga cucian nomor `#104` dan `#106`", dan petugas langsung menyerahkan pakaian milik orang lain tanpa meminta bukti KTP atau mencocokkan identitas pemilik, maka laundry tersebut memiliki celah **IDOR / BOLA**.

---

## 2. Cara Kerja

Pada arsitektur aplikasi modern, data disimpan dengan kunci unik (biasanya angka auto-increment seperti `1, 2, 3` atau UUID).

Aplikasi sering mengekspos ID ini pada URL atau body request:
```http
GET /api/documents/5820
GET /api/invoices/9912
PATCH /api/profiles/301
```

Alur kerja yang rentan terjadi ketika kode backend hanya melakukan hal ini:
```sql
SELECT * FROM documents WHERE id = <input_id>;
```
Server menemukan dokumen bernomor 5820 dan langsung mengirimkannya ke klien. Server **lupa melakukan langkah validasi kepemilikan**:
*"Apakah dokumen bernomor 5820 ini dibuat oleh atau ditujukan untuk pengguna yang sedang login saat ini?"*

---

## 3. Penyebab

- **Ketiadaan Validasi Kepemilikan (Ownership Check)**: Hanya memeriksa autentikasi (apakah user memiliki token valid), tetapi tidak memverifikasi hak akses terhadap *spesifik record* data yang diminta.
- **Penggunaan ID yang Mudah Ditebak (Sequential IDs)**: Menggunakan ID bertipe integer yang berurutan (`101, 102, 103`), sehingga penyerang sangat mudah melakukan *scraping* otomatis terhadap jutaan data pengguna lain.
- **Asumsi Kepercayaan Klien**: Menganggap bahwa karena antarmuka web hanya menampilkan tautan dokumen milik user sendiri, user tidak akan pernah mengganti angka ID di URL atau Postman.

---

## 4. Skenario Serangan

1. **Identifikasi Endpoint**: Korban dan penyerang sama-sama mendaftar di situs portal kuitansi pajak.
2. **Observasi URL**: Penyerang mengunduh kuitansinya sendiri dan melihat URL: `https://pajak.example/api/v1/receipts/45021`.
3. **Eksplorasi Angka**: Penyerang mengubah angka tersebut menjadi `45020` dan `45022` di browser atau cURL.
4. **Eksfiltrasi Massal**: Server mengembalikan kuitansi milik pembayar pajak lain lengkap dengan Nama, NIK, dan nominal penghasilan. Penyerang membuat skrip loop sederhana untuk menyedot seluruh kuitansi dari nomor `1` hingga `100000`.

---

## 5. Dampak

- **Kebocoran Data Pribadi Massal (Mass Data Leak)**: Pencurian identitas, riwayat transaksi, atau data rekam medis.
- **Manipulasi / Penghapusan Data Orang Lain**: Mengubah alamat pengiriman pesanan orang lain (`PUT /api/orders/501`) atau membatalkan tiket milik orang lain (`DELETE /api/tickets/88`).
- **Pelanggaran Regulasi Privasi**: Pelanggaran berat terhadap standar kepatuhan privasi (UU PDP di Indonesia atau GDPR di Eropa).

---

## 6. Bagian Sistem yang Terdampak

- Endpoint REST API dengan parameter ID path: `/api/users/:id`, `/api/orders/:id`.
- Parameter Query URL: `/download?file_id=123`.
- Payload body JSON pada operasi PUT / PATCH / DELETE: `{"order_id": 999, "status": "cancelled"}`.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Rekam Medis Elektronik KlinikSejahtera:**  
Pasien klinik dapat mengunduh surat rujukan laboratorium melalui aplikasi web. Ketika pasien mengklik tombol "Unduh PDF", aplikasi memanggil:
`GET /api/v1/lab-results/download?result_id=8841`

Seorang penyerang mengganti parameter `result_id` menjadi `8840`. Sistem backend KlinikSejahtera langsung men-generate PDF hasil laboratorium milik pasien lain tanpa memeriksa apakah sesi login yang aktif adalah pasien yang bersangkutan atau dokter penanggung jawabnya. Dalam hitungan jam, ribuan data diagnosis rahasia (seperti status HIV, hasil tes darah, dan penyakit kronis) berhasil diunduh secara ilegal.

---

## 8. Cara Mendeteksi

- **Uji Coba Pengguna Ganda (Multi-Account Testing)**: Buat Akun A (pemilik data) dan Akun B (penyerang). Buat data di Akun A, catat ID objeknya. Gunakan token sesi Akun B untuk meminta ID objek Akun A. Jika berhasil (status 200 OK), sistem positif rentan IDOR/BOLA.
- **Audit Kueri Database**: Periksa seluruh kueri pencarian data spesifik di repository backend. Jika kueri hanya memfilter berdasarkan `WHERE id = ?` tanpa melibatkan `AND user_id = ?`, patut diduga kuat terdapat celah BOLA.

---

## 9. Pencegahan

1. **Selalu Validasi Kepemilikan Data di Kueri Database (Wajib)**:  
   Ikat pencarian data dengan identitas pengguna yang terautentikasi:
   ```sql
   SELECT * FROM orders WHERE id = $order_id AND user_id = $logged_in_user_id;
   ```
2. **Gunakan GUID / UUID v4 yang Acak (Defense-in-Depth)**:  
   Ganti ID angka sekuensial (`1, 2, 3`) dengan format acak 128-bit seperti UUID v4 (`550e8400-e29b-41d4-a716-446655440000`). Hal ini mencegah penyerang menebak atau menyapu (*brute-force enumerating*) ID data lain. *(Ingat: UUID saja tidak cukup jika tidak ada pengecekan kepemilikan!)*
3. **Pemeriksaan Otorisasi Berbasis Kebijakan (Policy-Based Authorization)**:  
   Gunakan fungsi otorisasi objek: `canUserAccessObject(currentUser, requestedObject)`.

---

## 10. Penanganan

1. **Terapkan Filter Kepemilikan Global**: Perbarui lapisan akses data (Data Access Layer / Repository) agar setiap pemanggilan data otomatis menyertakan filter kepemilikan tenant / user ID.
2. **Audit Log URL Berurutan**: Analisis access log web server untuk mencari alamat IP yang melakukan ribuan pemanggilan dengan ID sekuensial berturut-turut.
3. **Blokir Pola Scraping**: Terapkan rate-limiting ketat pada endpoint pengunduhan dokumen.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"database/sql"
	"errors"
	"net/http"
)

type Document struct {
	ID      int
	OwnerID int
	Title   string
	Content string
}

// ❌ KODE RENTAN (Vulnerable)
// Hanya mencari berdasarkan document_id tanpa memeriksa siapa yang meminta
func GetDocumentVulnerable(db *sql.DB, docID int) (*Document, error) {
	query := "SELECT id, owner_id, title, content FROM documents WHERE id = $1"
	var doc Document
	err := db.QueryRow(query, docID).Scan(&doc.ID, &doc.OwnerID, &doc.Title, &doc.Content)
	return &doc, err
}

// ✅ KODE AMAN (Secure)
// 1. Mengikat kueri dengan loggedInUserID
// 2. Jika dokumen bukan milik user tersebut, kueri otomatis mengembalikan sql.ErrNoRows
func GetDocumentSecure(db *sql.DB, docID int, loggedInUserID int, userRole string) (*Document, error) {
	var query string
	var row *sql.Row

	// Jika admin, izinkan akses; jika pengguna biasa, WAJIB cocokkan owner_id
	if userRole == "admin" {
		query = "SELECT id, owner_id, title, content FROM documents WHERE id = $1"
		row = db.QueryRow(query, docID)
	} else {
		query = "SELECT id, owner_id, title, content FROM documents WHERE id = $1 AND owner_id = $2"
		row = db.QueryRow(query, docID, loggedInUserID)
	}

	var doc Document
	err := row.Scan(&doc.ID, &doc.OwnerID, &doc.Title, &doc.Content)
	if err == sql.ErrNoRows {
		// Tolak dengan pesan tidak ditemukan atau dilarang
		return nil, errors.New("dokumen tidak ditemukan atau Anda tidak memiliki akses")
	}

	return &doc, err
}
```

### B. Contoh dalam TypeScript (Express & Prisma / ORM)

```typescript
import { Request, Response } from 'express';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

interface AuthRequest extends Request {
  user?: { id: string; role: string };
}

// ❌ KODE RENTAN (Vulnerable)
// Mengambil invoice berdasarkan parameter ID saja
export async function getInvoiceVulnerable(req: AuthRequest, res: Response) {
  const invoiceId = req.params.id;

  const invoice = await prisma.invoice.findUnique({
    where: { id: invoiceId },
  });

  if (!invoice) {
    return res.status(404).json({ error: 'Invoice tidak ditemukan' });
  }

  // Siapapun yang login bisa melihat invoice milik siapapun!
  return res.json(invoice);
}

// ✅ KODE AMAN (Secure)
// Memasukkan userId milik pengguna yang sedang login ke klausa 'where'
export async function getInvoiceSecure(req: AuthRequest, res: Response) {
  const invoiceId = req.params.id;
  const loggedInUserId = req.user!.id;
  const isAdmin = req.user!.role === 'admin';

  const invoice = await prisma.invoice.findFirst({
    where: {
      id: invoiceId,
      // Jika bukan admin, WAJIB membatasi hanya invoice milik userId sendiri
      ...(isAdmin ? {} : { userId: loggedInUserId }),
    },
  });

  if (!invoice) {
    // Kembalikan 404 agar penyerang tidak tahu apakah ID tersebut ada milik orang lain
    return res.status(404).json({ error: 'Invoice tidak ditemukan' });
  }

  return res.json(invoice);
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Otomatis Antar-Pengguna
```typescript
it('harus menolak User B yang mencoba mengakses dokumen milik User A', async () => {
  const docUserA = await createDocument({ ownerId: 'user-a', title: 'Data Pribadi A' });
  const tokenUserB = generateToken({ id: 'user-b' });

  const response = await request(app)
    .get(`/api/documents/${docUserA.id}`)
    .set('Authorization', `Bearer ${tokenUserB}`);

  // Harus merespons 404 Not Found atau 403 Forbidden
  expect([403, 404]).toContain(response.status);
  expect(response.body.title).toBeUndefined();
});
```

---

## Referensi

- [OWASP API Security Top 10 - API1:2023 BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [PortSwigger: Insecure Direct Object References (IDOR)](https://portswigger.net/web-security/access-control/idor)
- [MITRE CWE-639](https://cwe.mitre.org/data/definitions/639.html)
