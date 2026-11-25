# NoSQL Injection

> **Metadata Keamanan:**
> - **CWE**: [CWE-943: Improper Neutralization of Special Elements in Data Query Logic](https://cwe.mitre.org/data/definitions/943.html)
> - **OWASP**: [A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 8.0 - 9.8)

---

## 1. Pengertian

**NoSQL Injection** adalah celah keamanan pada aplikasi yang menggunakan database NoSQL (seperti MongoDB, CouchDB, DynamoDB), di mana penyerang dapat memanipulasi logika kueri data dengan menyisipkan objek atau operator kueri tak terduga (misalnya `$ne`, `$gt`, atau `$regex`).

**Analogi Sederhana:**  
Bayangkan sebuah kantor pelayanan dokumen yang meminta kartu identitas Anda.  
Petugas: "Sebutkan nama lengkap Anda."  
Pemohon jahat: "Siapa saja yang namanya BUKAN kosong."  
Jika petugas menuruti logika tersebut secara harfiah tanpa memvalidasi bahwa input harus berupa satu nama orang, petugas akan mengeluarkan seluruh berkas milik semua orang di lemari arsip.

---

## 2. Cara Kerja

Tidak seperti database SQL yang menggunakan string teks perintah, database NoSQL (seperti MongoDB) sering kali menerima kueri dalam bentuk objek data BSON/JSON:

```javascript
db.users.find({ username: req.body.username, password: req.body.password })
```

Ketika aplikasi backend (terutama Node.js / Express dengan parser JSON) menerima input dari pengguna:
Jika klien mengirimkan:
```json
{
  "username": "admin",
  "password": { "$ne": "" }
}
```
Aplikasi yang meneruskan `req.body.password` secara langsung ke driver database akan mengeksekusi kueri:
"Cari pengguna dengan username 'admin' DAN password yang **TIDAK SAMA DENGAN ($ne)** string kosong".  
Karena password admin pasti memiliki nilai (tidak kosong), kondisi ini bernilai benar (`true`), dan penyerang berhasil masuk sebagai admin tanpa perlu tahu kata sandinya!

---

## 3. Penyebab

- **Meneruskan Objek Input Mentah ke Database Driver**: Menggunakan `req.body` langsung ke dalam fungsi pencarian tanpa validasi tipe data (membiarkan objek masuk padahal seharusnya string).
- **Kurangnya Schema Validation**: Tidak memvalidasi bahwa field input wajib bertipe primitif `string` sebelum kueri dieksekusi.
- **Penggunaan `$where` atau Evaluasi JavaScript Server-Side**: Mengizinkan kueri berbasis fungsi JavaScript string di sisi server MongoDB.

---

## 4. Skenario Serangan

1. **Observasi**: Penyerang melihat endpoint API login menerima payload JSON: `POST /api/v1/auth/login` dengan body `{"username": "user", "password": "pass"}`.
2. **Uji Coba Operator JSON**: Alih-alih mengirim string password biasa, penyerang mengirimkan objek: `{"username": "admin", "password": {"$gt": ""}}`.
3. **Bypass Autentikasi**: Server memeriksa database dengan operator `$gt` (lebih besar dari). Operator ini cocok dengan password string admin.
4. **Ekstraksi Data Sensitif (Blind Extraction)**: Dengan operator `$regex`, penyerang dapat menebak password huruf demi huruf: `{"password": {"$regex": "^a"}}`, `{"password": {"$regex": "^ab"}}`, dan seterusnya.

---

## 5. Dampak

- **Bypass Autentikasi Total**: Penyerang dapat masuk ke akun pengguna mana pun, termasuk super administrator, tanpa kredensial valid.
- **Kebocoran Data Sensitif**: Melalui ekstraksi regex, data rahasia dalam dokumen dapat disadap.
- **Kerusakan Integritas Data**: Penyerang dapat memperbarui dokumen pengguna lain jika operator disuntikkan ke kueri `update`.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint login dan registrasi berbasis JSON API.
- Filter pencarian atau katalog produk berbasis MongoDB.
- Pengaturan profil pengguna dan reset kata sandi.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Manajemen Rekam Medis Rumah Sakit Sehat:**  
Sebuah aplikasi web rekam medis menyimpan data pasien di MongoDB. Dokter mencari riwayat pasien berdasarkan nomor rekam medis (`medical_record_no`).

Endpoint backend:
```javascript
const patient = await Patient.findOne({ record_no: req.query.record_no });
```

Penyerang mengirim kueri HTTP:
`GET /api/patients?record_no[$ne]=null`

Parser Express mengubah query string tersebut menjadi objek `{ record_no: { "$ne": null } }`. Database MongoDB mengembalikan dokumen pasien pertama di database (yang kebetulan adalah pejabat publik). Penyerang berhasil membaca data riwayat penyakit rahasia tanpa izin.

---

## 8. Cara Mendeteksi

- **SAST (Static Analysis)**: Mengidentifikasi pemanggilan kueri NoSQL yang langsung menggunakan properti `req.body` atau `req.query` tanpa sanitasi schema.
- **DAST / Pentest API**: Mengirim payload JSON dengan key berawalan tanda dolar (`$ne`, `$gt`, `$where`) dan memeriksa respons anomali HTTP status 200.
- **Code Review**: Cari kueri MongoDB yang menerima input langsung dan pastikan ada sanitasi tipe atau library sanitasi (seperti `mongo-sanitize`).

---

## 9. Pencegahan

1. **Validasi Tipe Data Skema yang Ketat (Zod, Joi, Validator)**: Pastikan setiap input yang diharapkan berupa string divalidasi secara eksplisit sebelum menyentuh logika database. Tolak jika input berupa objek.
2. **Sanitasi Operator MongoDB**: Gunakan library seperti `mongo-sanitize` yang secara otomatis menghapus karakter berawalan `$` dari input pengguna.
3. **Hindari Evaluasi JavaScript (`$where`)**: Jangan pernah menggunakan opsi kueri `$where` atau `mapReduce` dengan string yang dapat dikendalikan pengguna.

---

## 10. Penanganan

1. **Deploy Middleware Sanitasi Darurat**: Pasang middleware sanitasi global yang membersihkan seluruh `req.body`, `req.query`, dan `req.params` dari key yang diawali simbol `$`.
2. **Audit Sesi Login**: Putuskan seluruh sesi aktif jika terindikasi ada login tanpa kata sandi yang valid melalui log query.
3. **Terapkan Validasi Schema**: Perbarui kode dengan validasi skema ketat (misalnya menggunakan Zod).

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Official MongoDB Go Driver)

```go
package main

import (
	"context"
	"errors"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
)

type LoginRequest struct {
	Username string      `json:"username"`
	Password interface{} `json:"password"` // ❌ RENTAN jika menerima tipe sembarang (interface{})
}

type User struct {
	Username     string `bson:"username"`
	PasswordHash string `bson:"password_hash"`
}

// ❌ KODE RENTAN: Jika request password di-unmarshal ke map atau interface{},
// penyerang bisa mengirim map: bson.M{"$ne": ""}
func AuthenticateVulnerable(ctx context.Context, coll *mongo.Collection, username string, rawPassword interface{}) (*User, error) {
	filter := bson.M{
		"username": username,
		"password": rawPassword, // Nilai bisa berupa objek kueri!
	}
	var user User
	err := coll.FindOne(ctx, filter).Decode(&user)
	return &user, err
}

// ✅ KODE AMAN: Validasi tipe data ketat dan selalu bandingkan hash password di aplikasi
type SafeLoginRequest struct {
	Username string `json:"username"`
	Password string `json:"password"` // Wajib string konkret!
}

func AuthenticateSecure(ctx context.Context, coll *mongo.Collection, username, password string) (*User, error) {
	if username == "" || password == "" {
		return nil, errors.New("kredensial tidak valid")
	}

	// Cari user HANYA berdasarkan username (string murni)
	var user User
	err := coll.FindOne(ctx, bson.M{"username": username}).Decode(&user)
	if err != nil {
		return nil, errors.New("pengguna tidak ditemukan")
	}

	// Verifikasi kata sandi menggunakan fungsi hash yang aman di kode aplikasi (misal: bcrypt)
	// bukan membandingkan password mentah di dalam kueri database!
	return &user, nil
}
```

### B. Contoh dalam TypeScript (Express & Mongoose/MongoDB)

```typescript
import { Request, Response } from 'express';
import mongoose from 'express';
import { z } from 'zod';

const User = mongoose.model('User', new mongoose.Schema({
  username: String,
  passwordHash: String,
}));

// ❌ KODE RENTAN (Vulnerable)
// req.body langsung dimasukkan ke kueri findOne tanpa pengecekan tipe
export async function loginVulnerable(req: Request, res: Response) {
  // Jika body: { "username": "admin", "password": { "$ne": "dummy" } }
  const user = await User.findOne({
    username: req.body.username,
    password: req.body.password,
  });

  if (!user) {
    return res.status(401).json({ message: 'Login gagal' });
  }
  return res.json({ message: 'Login berhasil', user });
}

// ✅ KODE AMAN (Secure)
// Menggunakan skema validasi Zod untuk memastikan input HANYA berupa string biasa
const LoginSchema = z.object({
  username: z.string().min(1).max(100),
  password: z.string().min(1).max(100), // Menolak jika input berupa object { $ne: ... }
});

export async function loginSecure(req: Request, res: Response) {
  const parseResult = LoginSchema.safeParse(req.body);
  if (!parseResult.success) {
    return res.status(400).json({ message: 'Format input tidak valid' });
  }

  const { username, password } = parseResult.data;

  // Cari user hanya dengan username
  const user = await User.findOne({ username });
  if (!user) {
    return res.status(401).json({ message: 'Username atau password salah' });
  }

  // Bandingkan password secara terpisah dengan bcrypt, BUKAN di kueri database
  return res.json({ message: 'Login berhasil' });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan cURL
Kirimkan body JSON berbahaya ke endpoint login lokal:
```bash
curl -X POST http://localhost:3000/api/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": {"$ne": "salah"}}'
```
- **Kondisi Rentan**: Mendapatkan status 200 OK dan token akses admin.
- **Kondisi Aman**: Mendapatkan status 400 Bad Request karena parser validasi skema mendeteksi bahwa field `password` bukan berupa string murni.

---

## Referensi

- [OWASP Testing Guide: Testing for NoSQL Injection](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection)
- [MITRE CWE-943](https://cwe.mitre.org/data/definitions/943.html)
- [PortSwigger: NoSQL Injection Vulnerabilities](https://portswigger.net/web-security/nosql-injection)
