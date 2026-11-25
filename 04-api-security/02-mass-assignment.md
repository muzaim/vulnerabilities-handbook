# Mass Assignment (Autobinding Vulnerability)

> **Metadata Keamanan:**
> - **CWE**: [CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes](https://cwe.mitre.org/data/definitions/915.html)
> - **OWASP**: [API3:2023 – Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
> - **Tingkat Keparahan (Severity)**: High (CVSS ~ 7.5 - 8.5)

---

## 1. Pengertian

**Mass Assignment** (atau *Autobinding Vulnerability*) adalah celah keamanan di mana framework backend secara otomatis memetakan (*bind*) seluruh data properti JSON dari permintaan pengguna langsung ke objek model database tanpa memfilter atribut mana yang diizinkan untuk diubah oleh pengguna.

Akibatnya, penyerang dapat menyisipkan properti tersembunyi (seperti `is_admin`, `balance`, `role`, atau `verified: true`) yang seharusnya hanya boleh diubah oleh sistem atau administrator.

**Analogi Sederhana:**  
Bayangkan Anda mengisi formulir pendaftaran anggota perpustakaan umum. Di formulir ada kolom nama dan alamat.  
Di bagian bawah formulir ada bagian bertuliskan: *"Hanya untuk Petugas: Biaya Iuran Tahunan [ Rp 100.000 ]"*.  
Jika Anda mencoret angka tersebut dan menuliskan `Rp 0` dengan pulpen Anda, lalu petugas administrasi langsung memasukkan angka `Rp 0` tersebut ke komputer perpustakaan tanpa memeriksa bagian khusus petugas, perpustakaan tersebut mengalami celah *mass assignment*.

---

## 2. Cara Kerja

Banyak framework modern (Spring Boot, Ruby on Rails, Express dengan Mongoose/Prisma, Go dengan GORM) menyediakan fitur kemudahan:
```javascript
// Memetakan req.body langsung ke model database
const updatedUser = await User.update(req.body, { where: { id: userId } });
```

Jika pengguna biasa mengirimkan:
```json
{
  "name": "Budi Santoso",
  "bio": "Software developer"
}
```
Maka kolom `name` dan `bio` diperbarui (Aman).

Namun jika penyerang mengirimkan payload yang dimanipulasi:
```json
{
  "name": "Budi Santoso",
  "bio": "Software developer",
  "is_admin": true,
  "account_balance": 99999999,
  "kyc_verified": true
}
```
Framework otomatis membaca semua properti tambahan tersebut dan menulisnya ke kolom database terkait. Penyerang seketika menjadi admin dengan saldo tak terbatas dan akun yang sudah terverifikasi tanpa perlu verifikasi KTP.

---

## 3. Penyebab

- **Menerima Objek Request Mentah ke Fungsi Database**: Menyerahkan `req.body` atau `json.NewDecoder` langsung ke struct atau entitas model database.
- **Ketiadaan Data Transfer Object (DTO) / Whitelist Skema**: Tidak mendefinisikan secara eksplisit daftar field yang boleh diedit oleh klien.
- **Asumsi Bahwa Frontend Adalah Benteng Pertahanan**: Mengira bahwa karena aplikasi web hanya menampilkan field nama dan bio di formulirnya, klien tidak akan pernah mengirimkan field `is_admin`.

---

## 4. Skenario Serangan

1. **Analisis Skema Database**: Penyerang membuat akun baru dan melihat struktur respons profil:
   ```json
   { "id": 10, "username": "budi", "email": "budi@mail.com", "role": "buyer", "credits": 0, "is_verified": false }
   ```
2. **Identifikasi Properti Sensitif**: Penyerang mencatat adanya field `credits` dan `is_verified`.
3. **Penyisipan Properti pada Permintaan Update**: Penyerang membuka halaman edit profil, lalu mengirim request PUT:
   ```http
   PUT /api/v1/profile HTTP/1.1
   Content-Type: application/json

   {
     "username": "budi",
     "credits": 5000000,
     "is_verified": true
   }
   ```
4. **Keberhasilan Modifikasi**: Database memperbarui field `credits` menjadi 5 juta. Penyerang menggunakan kredit gratis tersebut untuk berbelanja di e-commerce.

---

## 5. Dampak

- **Privilege Escalation**: Mengubah status peran menjadi admin (`role: "admin"`).
- **Manipulasi Finansial Langsung**: Menambahkan saldo dompet, diskon kupon, atau nilai kredit.
- **Bypass Verifikasi Identitas**: Mengubah status KYC (*Know Your Customer*) menjadi lolos tanpa verifikasi dokumen.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint pembuatan akun (`POST /api/register`).
- Endpoint pembaruan profil pengguna (`PUT /api/profile`, `PATCH /api/users/me`).
- Formulir pembuatan pesanan (*checkout / order creation*).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Peminjaman Online DanaKilat:**  
Nasabah DanaKilat mengajukan pinjaman dengan mengisi formulir jumlah pinjaman dan jangka waktu cicilan.

Endpoint pembuat pinjaman:
`POST /api/v1/loans`

Nasabah jahat menyuntikkan properti tingkat suku bunga (*interest_rate*) pada payload request JSON:
```json
{
  "amount": 20000000,
  "tenor_months": 12,
  "interest_rate": 0.0,
  "status": "APPROVED"
}
```

Karena backend DanaKilat menggunakan autobinding ke entitas `LoanEntity` tanpa DTO, pinjaman langsung tersimpan dengan bunga 0% dan status pinjaman otomatis "APPROVED" tanpa melalui proses analisis kredit oleh tim manajemen risiko.

---

## 8. Cara Mendeteksi

- **Fuzzing Properti JSON**: Kirim request ke endpoint update dengan menyisipkan field-field umum:
  `{"role": "admin", "isAdmin": true, "status": "active", "balance": 10000, "verified": true}`.
  Periksa apakah database atau respon JSON memuat nilai baru tersebut.
- **SAST**: Pemindai kode mendeteksi binding langsung dari request ke ORM (seperti `db.Model(&user).Updates(reqBody)`).

---

## 9. Pencegahan

1. **Gunakan DTO (Data Transfer Object) Terpisah (Wajib Mutlak)**:  
   Jangan pernah menggunakan model database sebagai model penampung request input. Buat kelas/struct terpisah yang HANYA memuat properti yang sah untuk diubah oleh pengguna.
2. **Gunakan Skema Validasi Whitelist (Zod / Joi)**:  
   Gunakan validator yang secara default membuang (*strip*) atau menolak properti asing yang tidak terdaftar di skema.
3. **Kunci Kolom Sensitif pada Level ORM**:  
   Gunakan fitur atribut readonly pada framework ORM (misal `gorm:"<-:create"` atau `attr_protected`) untuk mencegah pembaruan kolom tertentu.

---

## 10. Penanganan

1. **Audit Integritas Data**: Periksa apakah ada akun non-admin yang memiliki properti `is_admin = true` atau saldo mencurigakan yang diperbarui melalui endpoint profil.
2. **Deploy Patch DTO**: Ganti seluruh binding model langsung dengan DTO ketat.
3. **Koreksi Data Anomali**: Reset status akun-akun yang terbukti memodifikasi field internal secara ilegal.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Penggunaan DTO vs Model Langsung)

```go
package main

import (
	"encoding/json"
	"net/http"
)

// Model Database
type Account struct {
	ID        int    `json:"id"`
	Email     string `json:"email"`
	FullName  string `json:"full_name"`
	Role      string `json:"role"`      // Kritis: Hanya boleh diubah admin
	Balance   int64  `json:"balance"`   // Kritis: Hanya boleh diubah modul pembayaran
}

// ❌ KODE RENTAN (Vulnerable)
// Mengurai JSON langsung ke model Account
func UpdateAccountVulnerable(w http.ResponseWriter, r *http.Request) {
	var acc Account
	// Jika klien mengirim: {"full_name": "Budi", "role": "admin", "balance": 999999}
	// Maka struct acc akan menampung nilai tersebut dan disimpan ke DB!
	json.NewDecoder(r.Body).Decode(&acc)
	// db.Save(&acc)
	w.Write([]byte("Akun diperbarui"))
}

// ✅ KODE AMAN (Secure)
// DTO khusus yang HANYA memuat kolom yang diizinkan untuk diubah oleh user
type UpdateAccountDTO struct {
	FullName string `json:"full_name"` // HANYA nama lengkap yang diterima
}

func UpdateAccountSecure(w http.ResponseWriter, r *http.Request, loggedInUserID int) {
	var dto UpdateAccountDTO
	if err := json.NewDecoder(r.Body).Decode(&dto); err != nil {
		http.Error(w, "Payload JSON tidak valid", http.StatusBadRequest)
		return
	}

	// Kueri eksplisit: HANYA update kolom full_name untuk user bersangkutan
	// query: "UPDATE accounts SET full_name = $1 WHERE id = $2"
	// Kolom 'role' dan 'balance' aman seratus persen dari modifikasi liar
	w.Write([]byte("Akun aman diperbarui"))
}
```

### B. Contoh dalam TypeScript (Express & Zod Strict Parsing)

```typescript
import { Request, Response } from 'express';
import { z } from 'zod';

// ❌ KODE RENTAN (Vulnerable)
// req.body langsung diserahkan ke Prisma ORM
export async function updateProfileVulnerable(req: Request, res: Response) {
  const userId = req.user!.id;
  // Jika req.body memuat { "role": "admin", "isVerified": true }
  const updated = await prisma.user.update({
    where: { id: userId },
    data: req.body, // RENTAN MASS ASSIGNMENT!
  });
  return res.json(updated);
}

// ✅ KODE AMAN (Secure)
// Gunakan skema Zod dengan .strict() untuk menolak properti asing
const SafeProfileUpdateSchema = z.object({
  fullName: z.string().min(1).max(100),
  bio: z.string().max(250).optional(),
}).strict(); // .strict() otomatis menolak jika ada field liar seperti role/balance

export async function updateProfileSecure(req: Request, res: Response) {
  const parseResult = SafeProfileUpdateSchema.safeParse(req.body);
  if (!parseResult.success) {
    return res.status(400).json({
      error: 'Data tidak valid atau mengandung atribut terlarang',
      details: parseResult.error.flatten(),
    });
  }

  const userId = req.user!.id;
  const { fullName, bio } = parseResult.data;

  // Hanya field tervalidasi yang dikirim ke database
  const updated = await prisma.user.update({
    where: { id: userId },
    data: {
      fullName,
      bio,
    },
  });

  return res.json(updated);
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Penolakan Properti Tambahan
Kirimkan request update dengan properti liar:
```bash
curl -X PUT http://localhost:8080/api/profile \
  -H "Authorization: Bearer <token_user>" \
  -H "Content-Type: application/json" \
  -d '{"fullName": "Budi Santoso", "role": "admin", "balance": 1000000}'
```
- **Kondisi Rentan**: Di respons atau database nilai `role` berubah menjadi `admin`.
- **Kondisi Aman**: Permintaan ditolak dengan status `400 Bad Request` ("Data mengandung atribut terlarang"), atau properti liar tersebut dibuang secara diam-diam dan kolom `role` di database tetap `user`.

---

## Referensi

- [OWASP API Security Top 10 - API3:2023 Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [MITRE CWE-915](https://cwe.mitre.org/data/definitions/915.html)
- [PortSwigger: Mass Assignment Vulnerabilities](https://portswigger.net/web-security/api-testing#mass-assignment)
