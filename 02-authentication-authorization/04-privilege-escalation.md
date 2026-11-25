# Privilege Escalation (Eskalasi Hak Akses)

> **Metadata Keamanan:**
> - **CWE**: [CWE-269: Improper Privilege Management](https://cwe.mitre.org/data/definitions/269.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.8)

---

## 1. Pengertian

**Privilege Escalation** (Eskalasi Hak Akses) adalah situasi di mana seorang pengguna memperoleh hak akses, izin, atau wewenang yang lebih tinggi daripada yang seharusnya diberikan kepadanya oleh administrator sistem.

Terdapat dua jenis eskalasi hak akses:
1. **Vertical Privilege Escalation**: Pengguna biasa naik tingkat menjadi peran yang lebih tinggi (misalnya: pembeli biasa menjadi admin toko, atau staf magang menjadi manajer HR).
2. **Horizontal Privilege Escalation**: Pengguna mengakses data atau fitur milik pengguna lain yang berada pada tingkatan peran yang sama (misalnya: Nasabah A dapat mengedit profil Nasabah B).

**Analogi Sederhana:**  
- **Vertikal**: Seorang penonton bioskop membeli tiket kelas ekonomi biasa, tetapi berhasil menyelinap masuk ke lounge VVIP dan memesan makanan gratis tanpa membayar karena penjaga pintu lounge tertidur.
- **Horizontal**: Anda menginap di hotel di kamar 201, tetapi kunci kartu Anda ternyata bisa digunakan untuk membuka pintu kamar 202 milik tamu lain.

---

## 2. Cara Kerja

Eskalasi hak akses sering kali terjadi karena aplikasi mempercayai status peran (*role*) yang dikirimkan oleh browser atau aplikasi mobile pengguna:
1. **Parameter Tampering**: Formulir update profil mengirimkan JSON `{"username": "budi", "email": "budi@mail.com"}`. Penyerang sengaja menambahkan properti baru: `{"username": "budi", "role": "admin"}`. Jika backend langsung menyimpan objek tersebut ke database tanpa memfilter field, peran Budi langsung berubah menjadi admin (*Mass Assignment leading to Privilege Escalation*).
2. **Kelemahan Verifikasi Token / Header**: Mengubah nilai header seperti `X-User-Role: admin` atau mendekripsi/memalsukan klaim peran di dalam token JWT.
3. **Penyalahgunaan Alur Reset Password**: Mengarahkan token reset password milik akun admin ke alamat email milik penyerang.

---

## 3. Penyebab

- **Mass Assignment**: Menerima seluruh body request langsung ke dalam model database tanpa daftar putih (*whitelist*) atribut yang boleh diubah pengguna.
- **Pengecekan Akses Tidak Lengkap**: Hanya mengecek login, tanpa mengecek apakah peran pengguna memiliki wewenang untuk fungsi tertentu.
- **Penyimpanan Status Peran di Klien**: Menyimpan informasi peran di cookie yang tidak ditandatangani (*unsigned*) atau di localStorage dan mempercayainya di backend.

---

## 4. Skenario Serangan

1. **Pendaftaran Akun**: Penyerang membuat akun standar baru di aplikasi kantor `https://hrms.kantor.example`.
2. **Analisis Request**: Saat mengedit nama display di profil, penyerang menangkap request HTTP:
   ```http
   PUT /api/v1/users/me
   Content-Type: application/json

   { "displayName": "Budi Santoso" }
   ```
3. **Injeksi Parameter Peran**: Penyerang mengirim ulang request dengan menambahkan properti peran:
   ```json
   { "displayName": "Budi Santoso", "role": "superadmin", "department": "HR" }
   ```
4. **Keberhasilan Eskalasi**: Server merespons `200 OK` dengan status `role: "superadmin"`. Penyerang merefresh browser dan kini memiliki akses penuh ke menu penggajian (*payroll*) dan data rahasia seluruh karyawan perusahaan.

---

## 5. Dampak

- **Pengambilalihan Sistem Seluruhnya**: Penyerang dengan akses admin dapat mengunci pengguna lain, menghapus database, atau memasang pintu belakang (*backdoor*).
- **Spionase Perusahaan**: Mengakses laporan rahasia dewan direksi, dokumen merger bisnis, atau data gaji.
- **Penyalahgunaan Transaksi Keuangan**: Menyetujui pencairan dana atau transfer saldo yang seharusnya hanya dapat disetujui oleh level manajer.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint pembaruan profil pengguna (`/api/profile`, `/api/users/me`).
- Endpoint registrasi akun baru (jika ada field `is_admin` tersembunyi).
- Mekanisme pergantian organisasi / tenant pada aplikasi multi-tenant SaaS.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Manajemen Karyawan (HRMS) PT Maju Jaya:**  
HRMS PT Maju Jaya memiliki sistem persetujuan cuti. Karyawan mengajukan cuti, dan hanya atasan langsung (Manager) yang dapat menyetujui melalui tombol "Approve".

Seorang karyawan baru memeriksa kode JavaScript halaman web dan menemukan fungsi:
```javascript
fetch('/api/leave-requests/105/status', {
  method: 'PATCH',
  body: JSON.stringify({ status: 'APPROVED', approvedBy: 999 })
})
```

Karyawan tersebut mengirimkan request PATCH langsung dari console browser miliknya sendiri terhadap permohonan cutinya yang bernomor `105`. Server backend tidak memverifikasi apakah pemanggil API memiliki jabatan Manager, melainkan langsung mengubah status cuti menjadi "APPROVED". Karyawan berhasil meloloskan cuti panjang tanpa persetujuan atasan.

---

## 8. Cara Mendeteksi

- **Fuzzing Parameter Peran**: Uji setiap form edit profil dengan menyisipkan parameter seperti `role: "admin"`, `isAdmin: true`, `group: "administrators"`.
- **Matrix Testing Autorisasi**: Buat matriks pengujian hak akses yang memetakan setiap endpoint API dengan peran yang diizinkan (Guest, User, Manager, Admin).
- **SAST**: Cari pola pembaruan database seperti `db.Save(&user)` di mana struct model database diikat langsung dari request HTTP tanpa DTO (*Data Transfer Object*).

---

## 9. Pencegahan

1. **Gunakan Data Transfer Object (DTO) Ketat (Wajib)**: Jangan pernah mengikat request input langsung ke model database. Buat struct/tipe khusus yang hanya memuat field yang boleh diedit oleh pengguna umum (misal: hanya `displayName` dan `bio`).
2. **Terapkan Prinsip Hak Akses Terkecil (Least Privilege)**: Akun baru selalu diberikan peran terendah secara default di database.
3. **Otorisasi Server-Side yang Kuat**: Selalu ambil identitas peran dari sesi terpercaya di server (seperti isi token JWT yang telah diverifikasi tandatangannya), bukan dari body request.

---

## 10. Penanganan

1. **Audit Akun Peran Tinggi Segera**: Buat kueri database untuk memeriksa seluruh pengguna dengan peran admin:
   `SELECT * FROM users WHERE role = 'admin' ORDER BY created_at DESC`. Identifikasi apakah ada akun baru yang tiba-tiba berstatus admin.
2. **Turunkan (*Demote*) Akun Liar**: Cabut hak admin dari akun-akun yang tidak sah.
3. **Deploy Patch DTO**: Pasang validasi whitelist pada endpoint pembaruan profil.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"encoding/json"
	"net/http"
)

type User struct {
	ID       int    `json:"id"`
	Email    string `json:"email"`
	Name     string `json:"name"`
	Role     string `json:"role"`     // "user" atau "admin"
	IsActive bool   `json:"is_active"`
}

// ❌ KODE RENTAN (Vulnerable)
// Menerima langsung model User dari request body
func UpdateProfileVulnerable(w http.ResponseWriter, r *http.Request) {
	var user User
	// Jika body JSON berisi: {"name": "Budi", "role": "admin"}
	// Struct user akan terisi role="admin" dan langsung disimpan ke DB!
	json.NewDecoder(r.Body).Decode(&user)
	
	// db.Save(&user) -> RENTAN Mass Assignment / Privilege Escalation!
	w.Write([]byte("Profil diperbarui"))
}

// ✅ KODE AMAN (Secure)
// Gunakan DTO (Data Transfer Object) terpisah yang HANYA memuat field yang boleh diubah user
type UpdateProfileDTO struct {
	Name string `json:"name"` // Hanya Name yang bisa diubah
}

func UpdateProfileSecure(w http.ResponseWriter, r *http.Request, loggedInUserID int) {
	var dto UpdateProfileDTO
	if err := json.NewDecoder(r.Body).Decode(&dto); err != nil {
		http.Error(w, "Format data tidak valid", http.StatusBadRequest)
		return
	}

	// Lakukan update HANYA pada kolom nama untuk ID user yang sedang login
	// query: "UPDATE users SET name = $1 WHERE id = $2"
	// Kolom 'role' sama sekali tidak mungkin disentuh oleh handler ini
	w.Write([]byte("Profil aman diperbarui"))
}
```

### B. Contoh dalam TypeScript (Express & Zod)

```typescript
import { Request, Response } from 'express';
import { z } from 'zod';

// ❌ KODE RENTAN (Vulnerable)
// Mengambil seluruh req.body dan meneruskannya ke update database
export async function updateUserVulnerable(req: Request, res: Response) {
  const userId = req.params.id;
  // Jika penyerang menyisipkan { "role": "admin" }, Prisma akan mengupdate field role!
  const updated = await prisma.user.update({
    where: { id: userId },
    data: req.body,
  });
  return res.json(updated);
}

// ✅ KODE AMAN (Secure)
// Skema whitelist yang tegas: HANYA izinkan pengubahan field tertentu
const UpdateProfileSchema = z.object({
  fullName: z.string().min(2).max(50),
  avatarUrl: z.string().url().optional(),
  // role, isAdmin, dsb TIDAK ADA di sini dan otomatis diabaikan/ditolak!
});

export async function updateUserSecure(req: Request, res: Response) {
  const parseResult = UpdateProfileSchema.safeParse(req.body);
  if (!parseResult.success) {
    return res.status(400).json({ error: parseResult.error.flatten() });
  }

  const userId = req.user!.id; // Menggunakan ID dari token sesi login
  const { fullName, avatarUrl } = parseResult.data;

  const updated = await prisma.user.update({
    where: { id: userId },
    data: {
      fullName,
      avatarUrl,
    },
  });

  return res.json(updated);
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Penolakan Parameter Liar
Kirim request update profil dengan atribut eskalasi peran:
```bash
curl -X PUT http://localhost:8080/api/profile \
  -H "Authorization: Bearer <token_user_biasa>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Uji Coba", "role": "admin", "isAdmin": true}'
```
- **Kondisi Rentan**: Di database kolom `role` akun berubah menjadi `admin`.
- **Kondisi Aman**: Hanya kolom `name` yang berubah; kolom `role` di database tetap bernilai `user`.

---

## Referensi

- [OWASP Testing Guide: Testing for Privilege Escalation](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/03-Testing_for_Privilege_Escalation)
- [MITRE CWE-269](https://cwe.mitre.org/data/definitions/269.html)
- [PortSwigger: Privilege Escalation](https://portswigger.net/web-security/access-control#vertical-privilege-escalation)
