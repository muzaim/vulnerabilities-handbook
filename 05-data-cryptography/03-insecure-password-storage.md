# Insecure Password Storage (Penyimpanan Kata Sandi Tidak Aman)

> **Metadata Keamanan:**
> - **CWE**: [CWE-256: Plaintext Storage of a Password](https://cwe.mitre.org/data/definitions/256.html) & [CWE-916: Use of Password Hash With Insufficient Computational Effort](https://cwe.mitre.org/data/definitions/916.html)
> - **OWASP**: [A02:2021 – Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 8.5 - 9.8)

---

## 1. Pengertian

**Insecure Password Storage** adalah kegagalan sistem dalam melindungi kata sandi pengguna di database, baik karena menyimpannya dalam bentuk teks polos (*plaintext*) maupun karena menggunakan fungsi hash yang lemah, usang, atau tanpa bumbu pengacak (*salt*), seperti MD5 atau SHA-1.

Jika database mengalami kebocoran data (*data breach*), penyimpanan yang tidak aman memungkinkan penyerang membaca atau memecahkan (*cracking*) seluruh kata sandi pengguna secara instan.

**Analogi Sederhana:**  
- **Plaintext**: Anda menyimpan kunci lemari di bawah keset pintu depan rumah dengan tulisan: *"Kunci Lemari Emas"*. Siapa saja yang mengangkat keset langsung memegang kunci Anda.
- **Weak Hashing (MD5/SHA1)**: Anda memasukkan kunci ke dalam mesin blender jus buah biasa. Potongan kuncinya sangat mudah dirangkai kembali oleh penjahat menggunakan cetakan lilin dalam 2 detik.
- **Secure Hashing (bcrypt / Argon2)**: Anda melebur kunci tersebut ke dalam kawah lahar vulkanik bersama 10 ton batu granit (*Salt unik*) dan mengaduknya selama 10 jam (*Work Factor*). Tidak ada komputer di dunia yang bisa menyatukan kembali kawah lahar tersebut menjadi bentuk kunci semula.

---

## 2. Cara Kerja

Kata sandi **TIDAK BOLEH PERNAH DIENKRIPSI (bolak-balik)** dan **TIDAK BOLEH DISIMPAN MENTAH**. Kata sandi harus diolah menggunakan **Fungsi Hash Satu Arah yang Lambat (*Slow One-Way Cryptographic Hash*)**.

### Mengapa MD5 dan SHA-256 Biasa TIDAK AMAN untuk Password?
Algoritma seperti MD5, SHA-1, dan SHA-256 dirancang untuk **kecepatan tinggi** (misal memverifikasi integritas unduhan file ISO 5GB). Komputer modern dengan kartu grafis (GPU) dapat menghitung **miliaran hash MD5 per detik**:
- **Rainbow Table**: Tabel kamus raksasa yang sudah memetakan jutaan hash MD5 ke teks aslinya (misal hash `5f4dcc3b5aa765d61d8327deb882cf99` langsung diketahui sebagai kata `password`).
- **Tanpa Salt**: Jika 10.000 pengguna menggunakan password `Rahasia2024`, hash mereka di database akan sama persis. Penyerang hanya perlu memecahkan 1 hash untuk membajak 10.000 akun sekaligus.

### Solusi Standar Industri: Adaptive KDF (Key Derivation Function)
Algoritma seperti **bcrypt**, **scrypt**, dan **Argon2id**:
1. Menghasilkan *Salt* acak unik per pengguna (sehingga hash selalu berbeda meskipun passwordnya sama).
2. Memiliki *Cost Factor / Work Factor*: sengaja dibuat membutuhkan waktu komputasi (misal ~100 milidetik per password) dan memori RAM yang besar, membuat serangan brute force GPU menjadi sangat lambat dan tidak mungkin dilakukan secara masif.

---

## 3. Penyebab

- **Penyimpanan Plaintext**: Menyimpan password langsung apa adanya di kolom `password VARCHAR(255)`.
- **Penggunaan Fungsi Hash Cepat**: Menggunakan `md5(password)`, `sha1(password)`, atau `sha256(password)`.
- **Ketiadaan Salt Unik**: Melakukan hash tanpa menyertakan string acak kriptografis unik per baris pengguna.
- **Fungsi Hash Buatan Sendiri (*Roll Your Own Crypto*)**: Mencoba membuat logika pengacakan string sendiri yang rentan.

---

## 4. Skenario Serangan

1. **Kebocoran Database (SQLi / Backup Leak)**: Penyerang berhasil mengunduh tabel `users` berisi 100.000 baris.
2. **Inspeksi Kolom Password**: Penyerang melihat hash memiliki panjang tepat 32 karakter hex (ciri khas MD5): `e10adc3949ba59abbe56e057f20f883e`.
3. **Pencarian Online**: Penyerang menempelkan hash tersebut ke mesin pencari online (seperti crackstation.net). Dalam 0,1 detik, hasil muncul: `123456`.
4. **Serangan GPU Offline Massal**: Menggunakan software `Hashcat` dan kartu grafis RTX 4090, penyerang memecahkan 85% dari seluruh password pengguna dalam waktu kurang dari 2 jam.

---

## 5. Dampak

- **Pembajakan Akun Massal**: Seluruh akun pelanggan, staf, dan administrator dapat dimasuki penyerang.
- **Efek Domino (Credential Stuffing)**: Password yang bocor digunakan penyerang untuk meretas akun email, rekening bank, dan media sosial milik korban di platform lain.
- **Keruntuhan Reputasi Bisnis Total**: Pengguna kehilangan kepercayaan sepenuhnya saat mengetahui perusahaan menyimpan kata sandi mereka secara tidak bertanggung jawab.

---

## 6. Bagian Sistem yang Terdampak

- Tabel pengguna di database (`users`, `accounts`, `admins`).
- Modul pendaftaran akun (*registration*) dan pergantian kata sandi (*change password*).
- Modul autentikasi login.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Forum Komunitas Hobi Mainan:**  
Forum memiliki 50.000 anggota aktif. Developer lama membangun sistem registrasi menggunakan PHP lawas:
`$passwordHash = md5($inputPassword);`

Pada tahun 2023, database forum tersebut dicuri melalui celah SQL Injection dan diunggah ke forum peretas. Karena menggunakan MD5 tanpa salt, komplotan penjahat berhasil memecahkan password dari 40.000 anggota dalam beberapa menit.

Salah satu anggota forum tersebut adalah seorang pejabat bank terkemuka yang menggunakan email kantor dan kata sandi yang persis sama. Penyerang menggunakan password tersebut untuk masuk ke email korporat bank dan melancarkan penipuan transfer dana antar-perusahaan.

---

## 8. Cara Mendeteksi

- **Inspeksi Kolom Database**:
  - Jika isi kolom password terbaca sebagai teks biasa (`budi123`), sistem menyimpan **Plaintext**.
  - Jika panjang string tepat 32 karakter hex, sistem kemungkinan besar menggunakan **MD5**.
  - Jika panjang string tepat 40 karakter hex, sistem menggunakan **SHA-1**.
  - Jika string diawali dengan `$2a$`, `$2b$`, atau `$2y$`, sistem menggunakan **bcrypt** (Aman).
  - Jika diawali dengan `$argon2id$`, sistem menggunakan **Argon2id** (Sangat Aman).
- **SAST**: Pemindai kode mendeteksi import `crypto/md5` atau `crypto/sha1` yang dihubungkan dengan variabel password.

---

## 9. Pencegahan

1. **Gunakan Algoritma Hash Kata Sandi Modern (Argon2id atau bcrypt) (Wajib Mutlak)**:  
   - **Argon2id**: Pemenang Password Hashing Competition (PHC), standar tertinggi saat ini (direkomendasikan oleh OWASP).
   - **bcrypt**: Standar industri yang sangat matang dan teruji selama puluhan tahun.
2. **Wajibkan Salt Unik Otomatis**:  
   Pustaka bcrypt dan Argon2 secara otomatis menghasilkan salt acak dan menyematkannya ke dalam string hasil hash.
3. **Konfigurasikan Work Factor / Cost yang Tepat**:  
   Atur cost factor bcrypt minimal **12** (atau minimal 10). Tujuannya agar komputasi 1 hash memakan waktu sekitar 100-250 milidetik di server Anda.
4. **Terapkan Password Peppering (Defense-in-Depth)**:  
   Tambahkan string rahasia global (*Pepper*) yang disimpan terpisah dari database (misal di Key Management Service) sebelum password di-hash.

---

## 10. Penanganan

1. **Migrasi Hash Bertahap Saat Login**:  
   Jika sistem saat ini masih menggunakan MD5, lakukan migrasi saat pengguna login:
   ```text
   Saat user login -> periksa apakah password cocok dengan MD5 lama -> jika cocok, hash ulang password menggunakan bcrypt -> simpan hash bcrypt baru ke DB -> hapus hash MD5.
   ```
2. **Paksa Reset Password Massal**: Jika database lama sempat bocor, kirim email ke seluruh pengguna dan wajibkan pembuatan password baru dengan sistem hash aman yang baru.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (`golang.org/x/crypto/bcrypt`)

```go
package main

import (
	"errors"
	"fmt"
	"golang.org/x/crypto/bcrypt"
)

// ❌ KODE SANGAT RENTAN (Vulnerable)
// func HashPasswordVulnerable(password string) string {
// 	  hash := md5.Sum([]byte(password))
// 	  return hex.EncodeToString(hash[:]) // RENTAN! Sangat mudah di-crack oleh GPU
// }

// ✅ KODE AMAN: Menggunakan bcrypt dengan Cost Factor 12
func HashPasswordSecure(password string) (string, error) {
	// Cost factor 12 membutuhkan waktu komputasi ~150ms per hash (sangat tahan brute force)
	hashedBytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
	if err != nil {
		return "", err
	}
	return string(hashedBytes), nil
}

// Verifikasi password saat login
func VerifyPasswordSecure(hashedPasswordFromDB, passwordInput string) error {
	// bcrypt secara otomatis membedah salt dan work factor dari string hash
	err := bcrypt.CompareHashAndPassword([]byte(hashedPasswordFromDB), []byte(passwordInput))
	if err != nil {
		return errors.New("kata sandi tidak cocok")
	}
	return nil
}

func main() {
	hash, _ := HashPasswordSecure("PasswordRahasiaKu123!")
	fmt.Println("Hash bcrypt aman:", hash)
	// Output: $2a$12$e8N8Vw9m6D3y.FvQ5wZzpe6k1L1H3Z4mO7lK8j9k0l1m2n3o4p5q
}
```

### B. Contoh dalam TypeScript (Node.js & `bcrypt`)

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // Work factor 12

// ❌ KODE SANGAT RENTAN (Vulnerable)
// export function hashVulnerable(password: string) {
//   return crypto.createHash('md5').update(password).digest('hex'); // RENTAN!
// }

// ✅ KODE AMAN (Secure): Hashing dengan bcrypt
export async function hashPasswordSecure(plainPassword: string): Promise<string> {
  // bcrypt.hash otomatis menghasilkan salt acak unik dan mengiterasi komputasi
  const hash = await bcrypt.hash(plainPassword, SALT_ROUNDS);
  return hash;
}

// ✅ Verifikasi password saat pengguna login
export async function verifyPasswordSecure(plainPassword: string, storedHash: string): Promise<boolean> {
  // Membandingkan dengan waktu komputasi konstan (kebal timing attack)
  const isMatch = await bcrypt.compare(plainPassword, storedHash);
  return isMatch;
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Keunikan Hash (Uniqueness Check)
Lakukan hash terhadap kata sandi yang sama dua kali:
```go
hash1, _ := HashPasswordSecure("Testing123")
hash2, _ := HashPasswordSecure("Testing123")
```
- **Kondisi Rentan (MD5/Plaintext)**: `hash1` dan `hash2` bernilai sama persis.
- **Kondisi Aman (bcrypt)**: `hash1` dan `hash2` menghasilkan string yang **berbeda seratus persen** (karena salt yang dibuat otomatis selalu acak dan unik), namun keduanya tetap valid saat diuji menggunakan `bcrypt.CompareHashAndPassword`.

---

## Referensi

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [MITRE CWE-256: Plaintext Storage of a Password](https://cwe.mitre.org/data/definitions/256.html)
- [MITRE CWE-916: Insufficient Computational Effort](https://cwe.mitre.org/data/definitions/916.html)
