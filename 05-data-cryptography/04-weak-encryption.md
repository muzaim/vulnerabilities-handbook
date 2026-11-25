# Weak Encryption (Enkripsi yang Lemah)

> **Metadata Keamanan:**
> - **CWE**: [CWE-327: Use of a Broken or Risky Cryptographic Algorithm](https://cwe.mitre.org/data/definitions/327.html) & [CWE-326: Inadequate Encryption Strength](https://cwe.mitre.org/data/definitions/326.html)
> - **OWASP**: [A02:2021 – Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
> - **Tingkat Keparahan (Severity)**: High (CVSS ~ 7.5 - 8.5)

---

## 1. Pengertian

**Weak Encryption** adalah penggunaan algoritma enkripsi yang sudah usang, mode operasi cipher yang cacat secara matematis, panjang kunci yang terlalu pendek, atau penggunaan vektor inisialisasi (*Initialization Vector / IV*) yang statis atau berulang.

Meskipun data secara visual tampak teracak (*ciphertext*), kelemahan kriptografis ini memungkinkan penyerang mendekripsi data rahasia tanpa memiliki kunci enkripsi resmi.

**Analogi Sederhana:**  
Bayangkan Anda mengunci pagar rumah dengan gembok kuningan murah tahun 1950 yang kuncinya hanya memiliki 3 gerigi kecil.  
Setiap pencuri dengan kawat jepit rambut dapat membukanya dalam 5 detik. Menggunakan algoritma enkripsi kuno (seperti DES atau RC4) sama saja dengan memasang gembok berkarat tersebut di brankas digital Anda.

---

## 2. Cara Kerja

### A. Algoritma Kriptografi yang Sudah Patah (*Broken Algorithms*)
- **DES (Data Encryption Standard)**: Hanya memiliki panjang kunci 56-bit. Komputer modern dapat mencoba seluruh $2^{56}$ kemungkinan kunci dalam waktu beberapa jam saja.
- **RC4**: Cipher aliran (*stream cipher*) yang memiliki bias statistik pada byte-byte awal, memungkinkan rekonstruksi plaintext dari penyadapan lalu lintas jaringan.

### B. Mode Operasi Cacat: AES-ECB (Electronic Codebook)
AES (*Advanced Encryption Standard*) adalah algoritma yang sangat kuat. Namun, jika digunakan dengan mode **ECB**:
- Setiap blok 16-byte plaintext yang identik akan menghasilkan ciphertext yang identik pula.
- **Pola data tidak tersembunyi**: Contoh terkenalnya adalah gambar "ECB Penguin" (gambar maskot Linux Tux yang dienkripsi dengan AES-ECB tetap memperlihatkan siluet pinguin secara jelas!).

```
[ Plaintext: Blok A ] [ Plaintext: Blok A ]
         │                     │
   (Enkripsi ECB)        (Enkripsi ECB)
         ▼                     ▼
[ Cipher:  Blok X ] [ Cipher:  Blok X ] ===> Pola data sama persis!
```

### C. Penggunaan Ulang IV (Initialization Vector Reuse)
Pada mode enkripsi modern seperti AES-GCM atau AES-CBC, IV wajib diacak dan **TIDAK BOLEH PERNAH DIGUNAKAN ULANG** dengan kunci yang sama. Jika IV bersifat statis, penyerang dapat melakukan operasi XOR matematis untuk membongkar isi pesan asli.

---

## 3. Penyebab

- **Penggunaan Mode Default Library Kripto yang Usang**: Beberapa library Java atau C++ lawas secara default menggunakan mode `AES/ECB/PKCS5Padding`.
- **Penggunaan IV Statis (Hardcoded IV)**: Menggunakan `IV = "0000000000000000"` agar kode tidak error saat dekripsi.
- **Panjang Kunci Terlalu Pendek**: Menggunakan kunci RSA di bawah 2048 bit (misal RSA 512 bit atau 1024 bit yang sudah berhasil difaktorisasi oleh superkomputer).

---

## 4. Skenario Serangan

1. **Penyadapan Pesan Terenkripsi**: Penyerang menyadap lalu lintas pesan transaksi perbankan yang dienkripsi menggunakan mode AES-ECB.
2. **Analisis Pola Blok**: Penyerang mengetahui bahwa header pesan selalu berbunyi: `TRANSFER_DANA_RP_`.
3. **Manipulasi Blok Ciphertext (Bit-Flipping / Replay Blok)**: Karena setiap blok independen dan tidak memiliki autentikasi integritas (*MAC*), penyerang menukar urutan blok ciphertext rekening tujuan dengan nomor rekening penyerang.
4. **Pencairan Saldo**: Server mendekripsi pesan yang telah diubah urutannya dan mengeksekusi transfer ke rekening penyerang tanpa memicu error dekripsi.

---

## 5. Dampak

- **Dekripsi Data Rahasia**: Akses ke informasi perbankan, rekam medis, atau rahasia dagang.
- **Pemalsuan Integritas Pesan**: Mengubah nilai nominal transaksi tanpa merusak format enkripsi jika tidak menggunakan mode *Authenticated Encryption* (AEAD).

---

## 6. Bagian Sistem yang Terdampak

- Modul enkripsi kolom database (enkripsi nomor kartu debit/kredit).
- Protokol komunikasi jaringan kustom antar-server.
- File terenkripsi yang dibagikan ke pihak ketiga.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Voucher Hadiah Ritel BelanjaKuy:**  
BelanjaKuy menerbitkan kode voucher belanja digital. Kode voucher dibuat dengan mengenkripsi format teks:
`PENGGUNA_ID:NOMINAL_SALDO` menggunakan algoritma `DES`.

Seorang mahasiswa jurusan informatika mengumpulkan 5 contoh kode voucher yang berbeda. Karena kunci DES hanya 56-bit, mahasiswa tersebut menyewa cloud computing komputasi GPU murah selama 6 jam untuk memecahkan kunci DES melalui metode brute force ruang kunci (*key space*). Setelah kunci ditemukan, mahasiswa tersebut bebas men-generate ribuan kode voucher bernilai Rp 1.000.000 secara gratis.

---

## 8. Cara Mendeteksi

- **Pemeriksaan Nama Cipher di Kode**: Cari string algoritma berbahaya di source code:
  - `DES`, `3DES`, `TripleDES`
  - `RC4`, `ARC4`
  - `AES/ECB` atau `ECB`
  - `Blowfish`
- **Pemeriksaan Pembuatan IV**: Pastikan setiap fungsi enkripsi memanggil generator acak kriptografis (seperti `crypto/rand` di Go atau `crypto.randomBytes` di Node.js) dan bukan string statis.
- **Pemeriksaan Panjang Kunci RSA**: Pastikan kunci RSA minimal **2048 bit** (direkomendasikan **3072 bit** atau **4096 bit**), atau beralih ke Elliptic Curve Cryptography (**ECC / Ed25519**).

---

## 9. Pencegahan

1. **Gunakan Mode Authenticated Encryption with Associated Data (AEAD) (Wajib Modern)**:  
   Gunakan **AES-256-GCM** atau **ChaCha20-Poly1305**. Mode AEAD tidak hanya mengenkripsi kerahasiaan data, tetapi juga secara otomatis menyertakan tanda tangan digital (*Authentication Tag*) yang menolak dekripsi jika ciphertext dimanipulasi 1 bit saja.
2. **Hasilkan IV / Nonce Acak Kriptografis Unik untuk Setiap Enkripsi**:  
   Jangan pernah menggunakan IV yang sama untuk dua data yang berbeda. Simpan IV acak bersama dengan ciphertext (karena IV tidak perlu dirahasiakan, asalkan unik).
3. **Gunakan Kunci Minimal 256-Bit untuk Enkripsi Simetris**:  
   Gunakan AES-256 (32 byte key) untuk perlindungan jangka panjang.

---

## 10. Penanganan

1. **Re-encrypt Data di Database**: Buat skrip migrasi untuk mendekripsi data dengan algoritma lama dan mengenkripsinya kembali menggunakan AES-256-GCM dengan kunci baru.
2. **Hentikan Penggunaan Cipher Usang di Server**: Perbarui konfigurasi web server TLS untuk mematikan cipher suite CBC dan RC4.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (AES-256-GCM yang Aman)

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"errors"
	"fmt"
	"io"
)

// ❌ KODE SANGAT RENTAN: Menggunakan AES-ECB atau IV statis (JANGAN DILAKUKAN!)

// ✅ KODE AMAN: Menggunakan AES-GCM (Authenticated Encryption)
func EncryptAESGCM(plaintext []byte, key []byte) ([]byte, error) {
	// 1. Kunci WAJIB 32 byte untuk AES-256
	if len(key) != 32 {
		return nil, errors.New("panjang kunci harus 32 byte (256-bit)")
	}

	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}

	// 2. Bungkus dalam mode GCM (Galois/Counter Mode)
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	// 3. Wajib buat Nonce/IV unik dan acak secara kriptografis
	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return nil, err
	}

	// 4. Seal mengenkripsi dan otomatis menempelkan Authentication Tag di akhir ciphertext
	// nonce ditaruh di awal hasil agar mudah dibaca saat dekripsi
	ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
	return ciphertext, nil
}

func DecryptAESGCM(ciphertext []byte, key []byte) ([]byte, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	nonceSize := gcm.NonceSize()
	if len(ciphertext) < nonceSize {
		return nil, errors.New("ciphertext terlalu pendek")
	}

	nonce, actualCiphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
	// Open akan memverifikasi integritas data; jika data diedit penyerang, otomatis return error!
	plaintext, err := gcm.Open(nil, nonce, actualCiphertext, nil)
	if err != nil {
		return nil, errors.New("dekripsi gagal: data korup atau telah dimanipulasi!")
	}

	return plaintext, nil
}
```

### B. Contoh dalam TypeScript (Node.js & `crypto` AES-GCM)

```typescript
import crypto from 'crypto';

// Kunci 32 bytes (256-bit)
const SECRET_KEY = crypto.randomBytes(32);

// ✅ KODE AMAN: AES-256-GCM dengan tag autentikasi
export function encryptDataSecure(text: string): { iv: string; content: string; tag: string } {
  // IV unik 12 bytes untuk GCM
  const iv = crypto.randomBytes(12);

  const cipher = crypto.createCipheriv('aes-256-gcm', SECRET_KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  // Tag autentikasi (mencegah tampering)
  const tag = cipher.getAuthTag().toString('hex');

  return {
    iv: iv.toString('hex'),
    content: encrypted,
    tag: tag,
  };
}

export function decryptDataSecure(encryptedData: { iv: string; content: string; tag: string }): string {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    SECRET_KEY,
    Buffer.from(encryptedData.iv, 'hex')
  );

  // Set tag autentikasi yang diharapkan
  decipher.setAuthTag(Buffer.from(encryptedData.tag, 'hex'));

  let decrypted = decipher.update(encryptedData.content, 'hex', 'utf8');
  // final() akan melempar error jika data atau tag telah diubah penyerang
  decrypted += decipher.final('utf8');

  return decrypted;
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Integritas Authenticated Encryption (Tamper Test)
1. Enkripsi teks: `"TRANSFER Rp 1.000.000 ke Rekening 123"`.
2. Ubah 1 karakter pada string ciphertext hex.
3. Jalankan fungsi dekripsi.
- **Kondisi Rentan (CBC tanpa MAC)**: Dekripsi berhasil dan menghasilkan teks kacau tanpa error penolakan.
- **Kondisi Aman (GCM)**: Fungsi dekripsi langsung melempar error `dekripsi gagal: data korup atau telah dimanipulasi` dan menolak memproses teks.

---

## Referensi

- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [MITRE CWE-327: Use of a Broken or Risky Cryptographic Algorithm](https://cwe.mitre.org/data/definitions/327.html)
- [NIST Special Publication 800-175B: Guideline for Using Cryptographic Standards](https://csrc.nist.gov/publications/detail/sp/800-175b/rev-1/final)
