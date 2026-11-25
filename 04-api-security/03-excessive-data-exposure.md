# Excessive Data Exposure (Paparan Data Berlebih pada API)

> **Metadata Keamanan:**
> - **CWE**: [CWE-213: Exposure of Sensitive Information Due to Incompatible Policies](https://cwe.mitre.org/data/definitions/213.html)
> - **OWASP**: [API3:2023 – Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/) / *(Sebelumnya API3:2019 – Excessive Data Exposure)*
> - **Tingkat Keparahan (Severity)**: Medium / High (CVSS ~ 6.5 - 7.5)

---

## 1. Pengertian

**Excessive Data Exposure** adalah celah keamanan pada API di mana server backend mengembalikan seluruh properti objek data secara lengkap (termasuk informasi sensitif seperti nomor KTP, koordinat GPS, alamat rumah, atau hash password) ke sisi klien, dengan asumsi yang salah bahwa aplikasi mobile atau browser frontend akan menyaring dan hanya menampilkan informasi yang pantas di layar pengguna.

**Konsep Penting:**  
Apapun yang dikembalikan oleh respons API dapat dilihat secara gamblang oleh siapa saja yang membuka alat inspeksi jaringan (*Network Tab / DevTools / Charles Proxy*), terlepas dari apakah data itu dirender di layar tampilan atau tidak.

**Analogi Sederhana:**  
Bayangkan Anda meminta struk belanjaan di kasir minimarket.  
Petugas kasir menyerahkan selembar kertas yang di bagian atasnya tercetak daftar belanja Anda, namun di bagian belakang kertas tersebut tercetak seluruh riwayat kartu kredit, pin ATM, dan alamat rumah dari manajer toko. Petugas kasir berpesan: *"Tolong jangan dibalik kertasnya ya, baca depannya saja"*.

---

## 2. Cara Kerja

Developer sering kali mengambil jalan pintas saat membuat endpoint API:
```javascript
// Mengambil seluruh baris tabel pengguna dan langsung menjadikannya respons JSON
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
res.json(user);
```

Objek JSON yang dikirimkan ke browser/mobile memuat:
```json
{
  "id": 105,
  "name": "Siti Nurhaliza",
  "avatar": "https://cdn.example/siti.jpg",
  "email": "siti@private.com",
  "phone": "+6281234567890",
  "home_address": "Jl. Mawar No. 12 Jakarta",
  "nik_ktp": "3171012345670001",
  "password_hash": "$2b$12$K1dJ...",
  "role": "user"
}
```

Aplikasi frontend (misal aplikasi kencan atau direktori komunitas) di layar hanya menampilkan nama dan foto profil Siti. Namun, pengguna lain cukup menekan tombol `F12` -> tab `Network`, dan seluruh informasi rahasia Siti (NIK, alamat rumah, dan nomor telepon pribadi) dapat dibaca secara telanjang.

---

## 3. Penyebab

- **Pengembalian Objek Model Mentah (*Direct Model Serialization*)**: Mengubah model database langsung menjadi respons JSON tanpa lapisan serializer atau DTO.
- **Ketergantungan pada Filter Sisi Klien**: Mengandalkan kode JavaScript frontend (seperti `user.name` di React) untuk menyembunyikan data sensitif.
- **Ketiadaan API Contract yang Spesifik**: Satu endpoint serbaguna (seperti `/api/users/:id`) digunakan untuk profil publik sekaligus profil akun pribadi.

---

## 4. Skenario Serangan

1. **Observasi Trafik API**: Pengguna membuka aplikasi transportasi online dan mencari pengemudi terdekat.
2. **Inspeksi Jaringan**: Melalui proxy lalu lintas (seperti HTTP Toolkit), penyerang melihat respons API endpoint pencarian driver:
   `GET /api/v1/drivers/nearby`.
3. **Penemuan Data Sensitif**: Alih-alih hanya mengembalikan data posisi mobil dan nama driver, respons JSON memuat data lengkap driver:
   Nomor KTP, nomor HP pribadi, alamat rumah lengkap, hingga foto SIM pengemudi.
4. **Penyalahgunaan Data**: Penyerang mengumpulkan (*scraping*) data pribadi ribuan mitra pengemudi dan menjual database tersebut untuk keperluan telemarketing ilegal atau penipuan.

---

## 5. Dampak

- **Kebocoran Data Pribadi (Doxing & Pelanggaran Privasi)**: Publikasi identitas pribadi pengguna tanpa persetujuan.
- **Pencurian Kredensial**: Terpaparnya hash password yang dapat di-crack secara offline.
- **Pelanggaran Kepatuhan UU Privasi**: Risiko tuntutan hukum dan denda administratif atas kegagalan melindungi data identitas sensitif (PII).

---

## 6. Bagian Sistem yang Terdampak

- Endpoint profil publik pengguna (forum, direktori karyawan, aplikasi sosial).
- Endpoint daftar produk atau vendor (membocorkan harga modal supplier atau margin keuntungan).
- Endpoint pelacakan kurir atau pemesanan transportasi.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Aplikasi Kencan CariJodoh:**  
CariJodoh adalah aplikasi kencan seluler. Pengguna dapat melihat kartu profil pengguna lain di sekitarnya. Di antarmuka aplikasi, pengguna hanya melihat foto, nama panggilan, hobi, dan jarak perkiraan (misal: "3 km dari Anda").

Seorang penyerang memeriksa lalu lintas API `GET /api/v1/matches`. Ternyata backend CariJodoh mengirimkan objek database lengkap:
```json
{
  "nickname": "Dinda",
  "exact_latitude": -6.208812,
  "exact_longitude": 106.845601,
  "phone": "0811223344",
  "medical_conditions": "Tidak ada",
  "instagram_token": "IG_SECRET_123"
}
```

Dengan data koordinat GPS yang sangat presisi hingga 6 angka di belakang koma, penyerang dapat melacak lokasi rumah fisik korban secara real-time dan mengetahui nomor telepon pribadinya, menciptakan ancaman penguntitan (*stalking*) fisik yang sangat berbahaya bagi keselamatan korban.

---

## 8. Cara Mendeteksi

- **Inspeksi Respons API (Network Tab)**: Buka seluruh fitur aplikasi dan periksa isi respon JSON di tab Network. Bandingkan data yang tampak di layar dengan data yang ada di dalam respon JSON. Jika respon memuat properti yang tidak digunakan oleh UI atau bersifat rahasia, sistem mengalami **Excessive Data Exposure**.
- **Audit Skema OpenAPI / Swagger**: Tinjau skema respons endpoint dan pastikan tidak ada field rahasia yang tercantum.

---

## 9. Pencegahan

1. **Gunakan Response DTO / Serializer (Wajib Mutlak)**:  
   Definisikan secara eksplisit properti apa saja yang boleh dikembalikan ke klien. Hanya petakan properti yang aman ke dalam struct/class respons.
2. **Keluarkan (*Exclude*) Field Sensitif di Level Model ORM**:  
   Gunakan decorator atau tag penolak serialisasi (misalnya `json:"-"` di Go, `@Exclude()` di NestJS/TypeORM, atau transform filter).
3. **Pisahkan Endpoint Publik dan Privat**:  
   Gunakan endpoint berbeda untuk tampilan publik (`/api/users/:id/public-profile`) yang hanya memuat nama & avatar, dan tampilan privat pemilik akun (`/api/users/me`) yang memuat data akun lengkap.

---

## 10. Penanganan

1. **Pasang Serializer Filter Segera**: Terapkan fungsi pemetaan respons (response transformer) yang memotong field terlarang sebelum dikirim ke klien.
2. **Audit Dampak Kebocoran**: Periksa apakah field kritis seperti token rahasia atau password hash sempat terekspos. Jika ya, lakukan rotasi token.
3. **Pembaruan Aplikasi Frontend**: Sinkronkan antarmuka agar hanya mengandalkan data yang telah disaring.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Response DTO & Tag `json:"-"`)

```go
package main

import (
	"encoding/json"
	"net/http"
)

// Model Database Lengkap
type UserRecord struct {
	ID           int    `json:"id"`
	Username     string `json:"username"`
	Email        string `json:"email"`
	PasswordHash string `json:"-"` // Tag "-" mencegah field ini pernah keluar ke JSON!
	NIK          string `json:"nik"`
	HomeAddress  string `json:"home_address"`
}

// ❌ KODE RENTAN (Vulnerable)
// Mengembalikan seluruh struct database langsung ke klien
func GetPublicProfileVulnerable(w http.ResponseWriter, r *http.Request) {
	user := UserRecord{
		ID: 1, Username: "siti", Email: "siti@secret.com",
		PasswordHash: "$2b$12$...", NIK: "317101...", HomeAddress: "Jl. Rahasia No. 1",
	}
	// Seluruh data pribadi Siti terkirim ke klien!
	json.NewEncoder(w).Encode(user)
}

// ✅ KODE AMAN (Secure)
// Definisikan Response DTO yang HANYA memuat informasi publik yang aman
type PublicProfileResponse struct {
	ID       int    `json:"id"`
	Username string `json:"username"`
}

func GetPublicProfileSecure(w http.ResponseWriter, r *http.Request) {
	user := UserRecord{
		ID: 1, Username: "siti", Email: "siti@secret.com",
		PasswordHash: "$2b$12$...", NIK: "317101...", HomeAddress: "Jl. Rahasia No. 1",
	}

	// Petakan HANYA data yang relevan dan aman untuk konsumsi publik
	responseDTO := PublicProfileResponse{
		ID:       user.ID,
		Username: user.Username,
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(responseDTO)
}
```

### B. Contoh dalam TypeScript (Express & Selector Prisma/Lodash)

```typescript
import { Request, Response } from 'express';

// ❌ KODE RENTAN (Vulnerable)
export async function getAuthorProfileVulnerable(req: Request, res: Response) {
  // Mengambil semua kolom tabel user (termasuk password, no_telp, alamat)
  const author = await prisma.user.findUnique({
    where: { id: req.params.authorId },
  });
  // Mengembalikan semuanya ke browser
  return res.json(author);
}

// ✅ KODE AMAN (Secure)
export async function getAuthorProfileSecure(req: Request, res: Response) {
  // Gunakan 'select' pada ORM untuk HANYA menarik kolom yang dibutuhkan publik
  const author = await prisma.user.findUnique({
    where: { id: req.params.authorId },
    select: {
      id: true,
      username: true,
      avatarUrl: true,
      bio: true,
      // Field sensitif seperti email, passwordHash, phoneNumber SENGAJA TIDAK DIPILIH
    },
  });

  if (!author) {
    return res.status(404).json({ error: 'Penulis tidak ditemukan' });
  }

  return res.json(author);
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Respon JSON
Kirimkan permintaan ke endpoint profil publik:
```bash
curl -s http://localhost:8080/api/users/1/profile | jq .
```
- **Kondisi Rentan**: Output JSON memuat key sensitif seperti `"nik"`, `"password_hash"`, atau `"home_address"`.
- **Kondisi Aman**: Output JSON hanya memuat field publik yang telah ditentukan (`"id"`, `"username"`).

---

## Referensi

- [OWASP API Security Top 10 - API3:2023](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [MITRE CWE-213: Exposure of Sensitive Information](https://cwe.mitre.org/data/definitions/213.html)
