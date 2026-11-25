# Broken Access Control

> **Metadata Keamanan:**
> - **CWE**: [CWE-284: Improper Access Control](https://cwe.mitre.org/data/definitions/284.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) *(Peringkat #1 OWASP Top 10)*
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 8.0 - 9.8)

---

## 1. Pengertian

**Broken Access Control** (Kegagalan Kontrol Akses) adalah kondisi di mana sistem gagal membatasi tindakan yang boleh dilakukan oleh pengguna, sehingga pengguna dapat melihat, mengubah, atau menghapus data milik orang lain, ataupun menjalankan fungsi administratif yang bukan haknya.

**Analogi Sederhana:**  
Bayangkan sebuah rumah sakit di mana setiap pengunjung diberikan lencana nama tamu.  
Seharusnya lencana tamu hanya mengizinkan pengunjung berada di ruang tunggu. Namun, karena tidak ada pintu yang terkunci dan tidak ada perawat yang memeriksa lencana, pengunjung tersebut bebas masuk ke ruang operasi, membuka lemari obat bius, dan mengutak-atik rekam medis pasien lain.

---

## 2. Cara Kerja

Kontrol akses harus ditegakkan secara konsisten pada setiap titik akhir (endpoint) di sisi server (*server-side enforcement*).

Celah ini terjadi saat developer hanya mengandalkan penyembunyian antarmuka di sisi depan (*security through obscurity*):
- Menyembunyikan tombol "Hapus Akun" di frontend bagi pengguna biasa, tetapi lupa memeriksa peran pengguna di endpoint backend `/api/admin/delete-user`.
- Mengizinkan pengguna mengubah ID di URL: `GET /invoice/1001` diubah menjadi `GET /invoice/1002` tanpa memeriksa apakah faktur nomor 1002 tersebut milik pengguna yang sedang login.

---

## 3. Penyebab

- **Pengecekan Akses Hanya di Sisi Klien (Frontend)**: Berasumsi bahwa jika tombol dinonaktifkan di layar React/Vue, pengguna tidak bisa memanggil API backend.
- **Konfigurasi Default yang Longgar (CORS / Whitelist)**: Membiarkan rute API terbuka untuk publik secara tidak sengaja.
- **Ketiadaan Validasi Kepemilikan Data (Ownership Validation)**: Hanya memeriksa "apakah pengguna sudah login?", tetapi lupa memeriksa "apakah data yang diminta adalah milik pengguna ini?".
- **Manipulasi Parameter Peran (Role Tampering)**: Mempercayai parameter `role: "admin"` yang dikirim langsung dari permintaan JSON pengguna.

---

## 4. Skenario Serangan

1. **Eksplorasi Endpoint**: Pengguna biasa yang sudah login membuka alat inspeksi browser (Network Tab) dan mencatat URL API: `GET /api/user/profile`.
2. **Uji Coba Rute Administratif**: Pengguna mencoba mengakses URL umum admin secara manual: `GET /api/admin/users` atau `POST /api/admin/settings`.
3. **Penyalahgunaan Hak Akses**: Karena server tidak memverifikasi apakah token sesi memiliki peran `admin`, server langsung mengembalikan seluruh daftar pengguna atau menyimpan pengaturan baru.

---

## 5. Dampak

- **Kebocoran Data Massal**: Akses tidak sah ke seluruh database pelanggan, laporan keuangan, atau data transaksi rahasia.
- **Perusakan Data Sistem**: Pengguna biasa dapat menghapus produk, membatalkan pesanan orang lain, atau mematikan layanan.
- **Pengambilalihan Sistem**: Menjadikan akun pribadi sebagai akun administrator (*vertical privilege escalation*).

---

## 6. Bagian Sistem yang Terdampak

- Endpoint API manajemen pengguna dan master data.
- URL pengunduhan dokumen, laporan, atau faktur.
- Modul administrasi dan pengaturan global aplikasi.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Akademik Universitas Maju (SIAKAD):**  
Mahasiswa memiliki portal untuk melihat nilai semester. Di sisi frontend, mahasiswa hanya melihat tombol "Cetak KHS".

Seorang mahasiswa iseng melihat bahwa tombol cetak memanggil:
`GET /api/grades/export?student_id=2021001`

Mahasiswa tersebut mengganti `student_id` menjadi `2021002` (nomor teman sekelasnya). Server langsung mengunduh KHS milik temannya tanpa memeriksa apakah pemanggil request adalah pemilik ID tersebut. Lebih jauh lagi, saat mahasiswa mencoba memanggil:
`POST /api/grades/update` dengan body `{"student_id": 2021001, "course_id": "CS101", "grade": "A"}`, server menerima pembaruan tersebut karena tidak memvalidasi bahwa hanya akun Dosen yang berhak memanggil endpoint pembaruan nilai.

---

## 8. Cara Mendeteksi

- **Manual Testing dengan Multi-Role**: Buat dua akun dengan peran berbeda (User A dan User B, serta Admin). Coba gunakan sesi User A untuk mengakses atau memanipulasi data User B.
- **Automated Integration Testing**: Buat pengujian otomatis yang secara eksplisit memverifikasi bahwa akun tanpa hak akses menerima kode `403 Forbidden`.
- **Code Review**: Periksa apakah ada rute API yang tidak dilindungi oleh middleware autorisasi (*RBAC/ABAC middleware*).

---

## 9. Pencegahan

1. **Terapkan Kebijakan Default Deny (Tolak Semua Secara Default)**: Kecuali jika sebuah rute dinyatakan eksplisit sebagai publik (seperti halaman katalog atau pendaftaran), seluruh rute wajib menolak akses tanpa token dan peran yang sesuai.
2. **Tegakkan Kontrol Akses di Server**: Jangan pernah mengandalkan validasi frontend. Server harus selalu memverifikasi hak akses pengguna pada setiap request.
3. **Gunakan Framework Role-Based Access Control (RBAC) atau Attribute-Based (ABAC)**: Terapkan sistem pengecekan peran yang terpusat dan teruji.
4. **Verifikasi Kepemilikan Objek (Ownership Check)**: Pada setiap kueri pencarian data, sertakan ID pengguna yang login ke dalam filter database (misal: `WHERE id = $1 AND user_id = $current_user`).

---

## 10. Penanganan

1. **Tutup Rute Admin**: Blokir rute yang bocor melalui konfigurasi gateway/WAF.
2. **Audit Log Akses**: Telusuri siapa saja yang pernah mengakses endpoint administratif tersebut untuk mendeteksi potensi penyusupan.
3. **Pasang Middleware Autorisasi Terpusat**: Tambahkan middleware proteksi peran pada seluruh handler rute yang belum terlindungi.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Middleware RBAC)

```go
package main

import (
	"net/http"
)

type UserContext struct {
	UserID string
	Role   string // "user", "manager", "admin"
}

// ❌ KODE RENTAN (Vulnerable)
// Hanya memeriksa apakah user sudah login, tetapi tidak memeriksa apakah user adalah admin
func DeleteUserHandlerVulnerable(w http.ResponseWriter, r *http.Request) {
	// Siapapun yang punya token login biasa bisa menghapus user lain!
	targetID := r.URL.Query().Get("id")
	// Lakukan penghapusan di database...
	w.Write([]byte("User " + targetID + " berhasil dihapus"))
}

// ✅ KODE AMAN (Secure)
// Middleware pemverifikasi peran (Role Authorization Middleware)
func RequireRole(requiredRole string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		currentUser := getUserFromContext(r) // Ambil user dari token JWT/sesi yang sudah diverifikasi

		if currentUser == nil {
			http.Error(w, "Harus login terlebih dahulu", http.StatusUnauthorized) // 401
			return
		}

		if currentUser.Role != requiredRole {
			http.Error(w, "Akses ditolak: Anda tidak memiliki wewenang untuk tindakan ini", http.StatusForbidden) // 403
			return
		}

		next(w, r)
	}
}

func DeleteUserHandlerSecure(w http.ResponseWriter, r *http.Request) {
	targetID := r.URL.Query().Get("id")
	// Eksekusi penghapusan aman oleh admin...
	w.Write([]byte("User " + targetID + " berhasil dihapus oleh admin"))
}

// Pemasangan rute:
// http.HandleFunc("/api/admin/users/delete", RequireRole("admin", DeleteUserHandlerSecure))

func getUserFromContext(r *http.Request) *UserContext { return nil }
```

### B. Contoh dalam TypeScript (Express Middleware)

```typescript
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user?: { id: string; role: 'user' | 'admin' };
}

// ❌ KODE RENTAN (Vulnerable)
// Route handler langsung mengeksekusi tanpa memeriksa role
export async function updateSystemConfigVulnerable(req: AuthenticatedRequest, res: Response) {
  // Pengguna biasa bisa mengubah konfigurasi sistem!
  return res.json({ message: 'Konfigurasi sistem berhasil diperbarui' });
}

// ✅ KODE AMAN (Secure Middleware)
export function authorizeRole(allowedRoles: string[]) {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Autentikasi diperlukan' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Akses dilarang: Peran Anda tidak mencukupi' });
    }

    next();
  };
}

// Penggunaan pada router:
// app.post('/api/admin/settings', authorizeRole(['admin']), updateSystemConfigSecure);
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Integrasi Otomatis (Testing 403 Forbidden)
```typescript
it('harus menolak pengguna ber-role "user" saat mengakses endpoint admin', async () => {
  const userToken = generateToken({ id: 'user-1', role: 'user' });

  const response = await request(app)
    .post('/api/admin/settings')
    .set('Authorization', `Bearer ${userToken}`)
    .send({ maintenanceMode: true });

  expect(response.status).toBe(403);
  expect(response.body.error).toContain('Akses dilarang');
});
```

---

## Referensi

- [OWASP Top 10:2021 - A01 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [PortSwigger: Access Control Vulnerabilities](https://portswigger.net/web-security/access-control)
- [MITRE CWE-284: Improper Access Control](https://cwe.mitre.org/data/definitions/284.html)
