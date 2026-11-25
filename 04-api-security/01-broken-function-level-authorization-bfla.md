# Broken Function Level Authorization (BFLA)

> **Metadata Keamanan:**
> - **CWE**: [CWE-285: Improper Authorization](https://cwe.mitre.org/data/definitions/285.html)
> - **OWASP**: [API5:2023 – Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.1)

---

## 1. Pengertian

**Broken Function Level Authorization (BFLA)** adalah celah keamanan pada API di mana pengguna biasa yang terautentikasi dapat menjalankan fungsi administratif atau fungsi tingkat tinggi yang seharusnya dibatasi hanya untuk kelompok pengguna berwenang tertentu (seperti manajer, admin, atau auditor).

Jika **BOLA (Broken Object Level Authorization)** berkaitan dengan *"mengakses data milik orang lain"*, maka **BFLA** berkaitan dengan *"menjalankan fungsi / aksi yang bukan haknya"* (misalnya menghapus pengguna, mematikan sistem, atau mengubah konfigurasi global).

**Analogi Sederhana:**  
Bayangkan sebuah restoran cepat saji.  
Pelanggan dipersilakan memesan di layar sentuh kasir (*fungsi publik*).  
Namun tombol *"Cetak Laporan Keuangan Harian"* dan *"Buka Laci Kasir"* ternyata diletakkan di layar sentuh yang sama, hanya saja tombol itu sengaja diwarnai abu-abu transparan. Jika ada pelanggan iseng menekan area abu-abu tersebut dan laci kasir terbuka seketika, restoran tersebut memiliki celah BFLA.

---

## 2. Cara Kerja

Pada arsitektur API RESTful, fungsi-fungsi sering dikelompokkan berdasarkan nama entitas dan metode HTTP:
- `GET /api/v1/users` (Melihat profil)
- `POST /api/v1/users` (Mendaftar)
- `DELETE /api/v1/users/:id` (Menghapus akun - Khusus Admin)

Celah BFLA terjadi saat developer memasang middleware otentikasi login secara global (`checkLogin`), tetapi **lupa memasang verifikasi peran spesifik** pada metode sensitif:
- Mengizinkan pengguna biasa memanggil `DELETE /api/v1/users/50`.
- Membiarkan pengguna menebak rute admin: mengganti `/api/v1/users/me` menjadi `/api/v1/admin/users`.
- Membiarkan pengguna mengubah metode HTTP: `GET /api/v1/orders/10` diubah menjadi `DELETE /api/v1/orders/10`.

---

## 3. Penyebab

- **Penyembunyian Tombol di Sisi Klien (*Security Through Obscurity*)**: Berasumsi bahwa jika aplikasi mobile atau React tidak menampilkan menu admin, pengguna tidak akan pernah memanggil endpoint API admin secara langsung.
- **Ketiadaan Kebijakan Otorisasi Terpusat**: Pemeriksaan peran ditulis sporadis (*ad-hoc*) di beberapa controller dan terlewat pada controller lainnya.
- **Kesalahan Konfigurasi Routing**: Router hanya memeriksa izin pada path induk (misal: `/api/v1/products`), tetapi rute anak (`/api/v1/products/purge`) tidak mewarisi proteksi izin admin.

---

## 4. Skenario Serangan

1. **Eksplorasi Dokumentasi / File JS**: Penyerang membuka file JavaScript frontend atau Swagger UI publik (`/api/swagger.json`) dan menemukan daftar endpoint admin yang tersembunyi: `POST /api/v1/admin/export-all-data`.
2. **Pengujian Token Biasa**: Penyerang login dengan akun pengguna gratis biasa dan mendapatkan token JWT.
3. **Pemanggilan Endpoint Admin**: Penyerang mengirimkan request cURL ke `POST /api/v1/admin/export-all-data` dengan menyertakan token JWT pengguna biasa.
4. **Respon Sukses**: Server memverifikasi bahwa token valid (pengguna login), tetapi tidak memeriksa apakah pengguna adalah admin. Seluruh arsip data perusahaan berhasil diunduh.

---

## 5. Dampak

- **Vertical Privilege Escalation**: Pengguna biasa bertindak selayaknya super admin.
- **Perusakan Data Sistem**: Menghapus data inventaris, membatalkan pesanan massal, atau mematikan layanan.
- **Kebocoran Data Rahasia Perusahaan**: Mengakses laporan audit, mutasi bank, atau histori transaksi seluruh pengguna.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint administratif (`/api/admin/*`, `/api/management/*`).
- Operasi HTTP `DELETE` dan `PUT/PATCH` pada resource penting.
- Endpoint ekspor data, konfigurasi integrasi sistem, dan audit log.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Penjualan Tiket Bioskop TiketAsik:**  
TiketAsik memiliki aplikasi mobile untuk pemesanan tiket bioskop.

Saat meneliti API, penyerang menemukan endpoint pembatalan tiket:
`DELETE /api/v1/bookings/8812`

Endpoint ini dibuat untuk staf kasir bioskop jika ada pelanggan yang membatalkan pesanan di loket. Namun, developer lupa memasang middleware pemeriksaan jabatan kasir.

Penyerang mengirimkan ribuan request `DELETE` untuk nomor pesanan acak dari `1` sampai `5000` menggunakan token akun penonton biasa miliknya. Seluruh tiket penonton lain yang sudah dipesan untuk jadwal tayang film perdana malam itu terhapus dari sistem, menyebabkan kekacauan besar di bioskop.

---

## 8. Cara Mendeteksi

- **Matrix Testing Autorisasi API**: Buat skrip otomatis yang mencoba mengakses seluruh daftar endpoint API menggunakan token dari berbagai level peran (Unauthenticated, User Biasa, Manager, Superadmin).
- **Audit File OpenAPI / Postman Collection**: Tinjau setiap endpoint yang memiliki awalan `/admin/` atau aksi berbahaya dan verifikasi middleware proteksinya.

---

## 9. Pencegahan

1. **Terapkan Prinsip "Deny by Default"**:  
   Secara default, seluruh endpoint API harus menolak akses kecuali dinyatakan secara eksplisit peran mana yang diizinkan (*allow-list*).
2. **Gunakan Middleware Verifikasi Peran Terpusat**:  
   Setiap route handler fungsi administratif wajib dibungkus dengan middleware pengecekan izin (misal: `requireRole(['ADMIN', 'SUPERADMIN'])`).
3. **Validasi Metode HTTP Secara Tepat**:  
   Pastikan aturan otorisasi membedakan antara aksi baca (`GET`) dan aksi modifikasi/penghapusan (`POST`, `PUT`, `DELETE`).

---

## 10. Penanganan

1. **Pasang Proteksi di API Gateway**: Terapkan filter penolakan di API Gateway (Kong, AWS API Gateway, NGINX) untuk seluruh rute `/api/admin/*` bagi token yang tidak memiliki klaim peran admin.
2. **Audit Aksi Terakhir**: Periksa log sistem untuk mendeteksi apakah ada fungsi admin yang dipanggil oleh ID pengguna non-admin dalam beberapa minggu terakhir.
3. **Perbarui Router Backend**: Pasang middleware otorisasi pada seluruh route sensitif.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Middleware Berbasis Peran & Izin)

```go
package main

import (
	"net/http"
)

type Claims struct {
	UserID string
	Role   string // "USER", "STAFF", "ADMIN"
}

// ✅ KODE AMAN: Middleware Otorisasi Level Fungsi (BFLA Protection)
func AuthorizeFunction(allowedRoles ...string) func(http.HandlerFunc) http.HandlerFunc {
	return func(next http.HandlerFunc) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			// Ambil klaim pengguna yang sudah diverifikasi dari konteks sesi
			user := getCurrentUserClaims(r)
			if user == nil {
				http.Error(w, "Autentikasi diperlukan", http.StatusUnauthorized)
				return
			}

			// Periksa apakah peran pengguna ada di daftar peran yang diizinkan
			isAllowed := false
			for _, role := range allowedRoles {
				if user.Role == role {
					isAllowed = true
					break
				}
			}

			if !isAllowed {
				http.Error(w, "Akses fungsi dilarang: Peran Anda tidak memenuhi syarat wewenang", http.StatusForbidden)
				return
			}

			next(w, r)
		}
	}
}

// Handler fungsi admin
func PurgeCacheHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Seluruh cache sistem berhasil dibersihkan"))
}

// Pemasangan rute yang AMAN:
// http.HandleFunc("/api/v1/system/purge-cache", AuthorizeFunction("ADMIN")(PurgeCacheHandler))

func getCurrentUserClaims(r *http.Request) *Claims { return nil }
```

### B. Contoh dalam TypeScript (Express & Role-Based Guard)

```typescript
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user?: { id: string; role: 'CUSTOMER' | 'SUPPORT' | 'ADMIN' };
}

// ✅ KODE AMAN: Guard Level Fungsi untuk API
export function requireFunctionAccess(allowedRoles: Array<'CUSTOMER' | 'SUPPORT' | 'ADMIN'>) {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Pengguna belum terotentikasi' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        error: 'Forbidden: Anda tidak memiliki wewenang untuk mengeksekusi fungsi ini',
      });
    }

    next();
  };
}

// ❌ KODE RENTAN (Vulnerable): Hanya mengecek apakah user login
// app.delete('/api/v1/users/:id', authenticateToken, deleteUserHandler);

// ✅ KODE AMAN (Secure): Membatasi eksekusi penghapusan HANYA untuk peran ADMIN
// app.delete('/api/v1/users/:id', authenticateToken, requireFunctionAccess(['ADMIN']), deleteUserHandler);
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Penolakan Akses Fungsi Non-Admin
Panggil fungsi administratif menggunakan token pengguna bertingkat 'CUSTOMER':
```bash
curl -X DELETE http://localhost:8080/api/v1/users/105 \
  -H "Authorization: Bearer <token_customer>"
```
- **Kondisi Rentan**: Sistem merespons `200 OK` ("Pengguna berhasil dihapus").
- **Kondisi Aman**: Sistem merespons `403 Forbidden` ("Forbidden: Anda tidak memiliki wewenang untuk mengeksekusi fungsi ini").

---

## Referensi

- [OWASP API Security Top 10 - API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [MITRE CWE-285: Improper Authorization](https://cwe.mitre.org/data/definitions/285.html)
