# Unrestricted File Upload (Unggah Berkas Tanpa Batasan)

> **Metadata Keamanan:**
> - **CWE**: [CWE-434: Unrestricted Upload of File with Dangerous Type](https://cwe.mitre.org/data/definitions/434.html)
> - **OWASP**: [A04:2021 – Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/) / [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 9.0 - 9.8)

---

## 1. Pengertian

**Unrestricted File Upload** adalah celah keamanan di mana aplikasi web mengizinkan pengguna mengunggah berkas ke server tanpa memvalidasi secara memadai jenis berkas (*file type*), ekstensi, ukuran, atau isi berkas, sehingga penyerang dapat mengunggah skrip berbahaya (seperti *Webshell*) dan mengeksekusinya di server.

**Analogi Sederhana:**  
Bayangkan sebuah pameran seni di mana panitia mengizinkan pengunjung menitipkan lukisan kanvas untuk dipajang di dinding galeri.  
Seorang penjahat tidak membawa lukisan, melainkan membawa bom dinamit yang ditempeli stiker bertuliskan "Lukisan Pemandangan". Petugas galeri hanya membaca stiker tersebut tanpa membuka bungkusnya dan langsung memasangnya di dinding pameran. Ketika bom meledak, seluruh galeri hancur.

---

## 2. Cara Kerja

Aplikasi web sering kali memiliki fitur unggah berkas: foto profil, lampiran CV, berkas PDF faktur, dsb.

Jika server menyimpan file tersebut di direktori yang dapat diakses oleh publik (misal: `/public/uploads/`) dan server web (seperti Apache / Nginx / PHP-FPM) dikonfigurasi untuk mengeksekusi skrip di folder tersebut:
1. Penyerang mengunggah berkas skrip backend, misalnya `shell.php`, `backdoor.jsp`, atau `malicious.aspx`.
2. Server menyimpannya sebagai `https://toko.example/uploads/shell.php`.
3. Penyerang membuka URL tersebut di browsernya.
4. Server mengeksekusi skrip PHP tersebut dan memberikan terminal kendali kepada penyerang (*Remote Code Execution*).

---

## 3. Penyebab

- **Hanya Memvalidasi di Sisi Frontend (HTML `accept="image/*"`)**: Penyerang dapat mem-bypass validasi peramban dengan sangat mudah menggunakan cURL atau Burp Suite.
- **Hanya Memeriksa Header `Content-Type`**: Header MIME type dikirimkan oleh klien, sehingga penyerang bisa mengirim file `shell.php` dengan memalsukan header menjadi `Content-Type: image/jpeg`.
- **Blacklist Ekstensi yang Tidak Lengkap**: Hanya memblokir `.php`, tetapi lupa memblokir variasi lain seperti `.php5`, `.phtml`, `.phar`, `.jsp`, `.svg` (mengandung XSS), atau file ganda seperti `foto.php.jpg`.
- **Menyimpan Berkas di Web Root yang Memiliki Izin Eksekusi (*Execute Permission*)**: Membiarkan web server menjalankan kode yang berada di dalam folder unggahan.

---

## 4. Skenario Serangan

1. **Pembuatan Webshell**: Penyerang membuat file teks bernama `avatar.php` dengan isi satu baris:
   `<?php system($_GET['cmd']); ?>`.
2. **Pengunggahan File**: Penyerang mengunggah file tersebut melalui form foto profil.
3. **Pencarian Jalur Berkas**: Penyerang melihat URL gambar di profil: `https://toko.example/static/avatars/avatar.php`.
4. **Eksekusi Perintah Sistem (RCE)**: Penyerang membuka URL:
   `https://toko.example/static/avatars/avatar.php?cmd=id;uname -a;cat /etc/passwd`.
   Server menjalankan perintah terminal dan menampilkan hasilnya di layar browser.

---

## 5. Dampak

- **Remote Code Execution (RCE) Penuh**: Penyerang memiliki akses langsung ke sistem operasi server.
- **Defacement Total**: Mengganti halaman depan website dengan pesan penyerang.
- **Penyisipan Malware & Phishing**: Menggunakan server perusahaan untuk menyebarkan ransomware ke pengguna lain.
- **Pengunduhan Seluruh Database**: Menggunakan webshell untuk mengakses database lokal.

---

## 6. Bagian Sistem yang Terdampak

- Formulir upload foto profil atau logo organisasi.
- Fitur upload lampiran formulir lowongan kerja (CV/Resume).
- Fitur import file data (CSV, Excel, XML).
- Editor konten CMS yang mendukung upload media.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Perekrutan Kerja KarirBagus:**  
Portal KarirBagus mengizinkan pelamar kerja mengunggah CV dalam format PDF. Sistem hanya memeriksa ekstensi di frontend.

Di backend (Node.js & Express), developer menggunakan konfigurasi:
```javascript
const upload = multer({ dest: 'public/resumes/' });
```

Penyerang mengunggah file `exploit.html` yang memuat skrip pencuri cookie dan malware unduhan otomatis. Saat tim HRD membuka tautan CV di dashboard:
`https://karirbagus.example/resumes/exploit.html`
Browser HRD membuka halaman HTML tersebut di bawah domain resmi perusahaan, menjalankan skrip pencurian sesi, dan mencuri kredensial login admin HRD.

---

## 8. Cara Mendeteksi

- **Fuzzing Ekstensi Ganda & Alternatif**: Coba unggah file dengan nama:
  `test.php`, `test.php.png`, `test.phtml`, `test.svg`, `test.html`.
- **Inspeksi Magic Bytes**: Unggah file teks dengan ekstensi `.jpg`. Jika server menerima, sistem tidak memeriksa konten riil berkas.
- **Audit Izin Folder**: Periksa apakah folder `/uploads/` memiliki izin eksekusi (`chmod +x`).

---

## 9. Pencegahan

1. **Gunakan Whitelist Ekstensi yang Sangat Ketat (Bukan Blacklist)**:  
   Hanya izinkan ekstensi yang benar-benar dibutuhkan (misal: HANYA `.jpg`, `.jpeg`, `.png`, `.pdf`). Tolak seluruh ekstensi lainnya.
2. **Validasi Tipe Konten Sejati Melalui Magic Bytes (File Signatures)**:  
   Jangan percaya ekstensi atau `Content-Type`. Baca beberapa byte pertama berkas untuk memverifikasi apakah berkas tersebut benar-benar gambar JPEG (`FF D8 FF`), PNG (`89 50 4E 47`), atau PDF (`%PDF`).
3. **Ganti Nama Berkas Secara Acak (Randomized Filename)**:  
   Ubah nama file menjadi UUID acak saat disimpan (misal: `550e8400.png`), sehingga penyerang tidak dapat memanggil file berbahaya dengan nama aslinya.
4. **Simpan Berkas di Luar Direktori Publik atau Gunakan Cloud Object Storage**:  
   Simpan di AWS S3, Google Cloud Storage, atau folder di luar `public_html`. Layanan cloud storage menyajikan file murni sebagai data statis tanpa pernah mengeksekusi kode skrip di dalamnya.
5. **Nonaktifkan Izin Eksekusi di Web Server**:  
   Pada Nginx/Apache, konfigurasikan folder `/uploads/` agar melarang eksekusi interpreter skrip PHP/CGI.

---

## 10. Penanganan

1. **Karantina Folder Unggahan**: Segera cabut izin eksekusi (*execute permission*) dari direktori upload.
2. **Pindai Berkas Asing**: Cari file dengan ekstensi executable (`.php`, `.sh`, `.exe`, `.jsp`) di folder uploads:
   `find /var/www/uploads/ -type f \( -name "*.php*" -o -name "*.sh" \)`.
3. **Hapus Berkas Berbahaya**: Hapus seluruh file mencurigakan dan periksa log akses web server untuk melihat aktivitas webshell tersebut.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Validasi Magic Bytes & Whitelist)

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"errors"
	"io"
	"net/http"
	"os"
	"path/filepath"
)

var allowedMimeTypes = map[string]string{
	"image/jpeg": ".jpg",
	"image/png":  ".png",
	"application/pdf": ".pdf",
}

// ✅ KODE AMAN: Validasi tipe konten berdasarkan isi berkas (Magic Bytes)
func SaveUploadedFileSecure(file io.Reader, uploadDir string) (string, error) {
	// 1. Baca 512 byte pertama untuk mendeteksi MIME type sebenarnya (Magic Bytes)
	buffer := make([]byte, 512)
	n, err := file.Read(buffer)
	if err != nil && err != io.EOF {
		return "", err
	}

	realMimeType := http.DetectContentType(buffer[:n])
	ext, ok := allowedMimeTypes[realMimeType]
	if !ok {
		return "", errors.New("tipe berkas tidak diizinkan atau berbahaya!")
	}

	// 2. Buat nama file acak (UUID/random hex) - Abaikan nama file bawaan klien!
	randomBytes := make([]byte, 16)
	rand.Read(randomBytes)
	safeFilename := hex.EncodeToString(randomBytes) + ext

	targetPath := filepath.Join(uploadDir, safeFilename)

	// 3. Tulis file dengan izin non-executable (0644)
	dst, err := os.OpenFile(targetPath, os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		return "", err
	}
	defer dst.Close()

	// Tulis kembali 512 byte pertama, lalu salin sisa file
	dst.Write(buffer[:n])
	_, err = io.Copy(dst, file)
	return safeFilename, err
}
```

### B. Contoh dalam TypeScript (Express & File Type Sniffing)

```typescript
import multer from 'multer';
import { fromBuffer } from 'file-type';
import crypto from 'crypto';
import path from 'path';
import fs from 'fs/promises';

// Gunakan memory storage agar bisa menganalisis buffer sebelum disimpan ke disk
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 }, // Maksimal 5MB
});

// ✅ KODE AMAN: Memverifikasi Magic Bytes menggunakan library file-type
export async function handleSecureUpload(req: Request, res: Response) {
  if (!req.file) {
    return res.status(400).json({ error: 'Tidak ada berkas yang diunggah' });
  }

  // 1. Deteksi jenis berkas dari buffer biner asli (Magic Bytes)
  const fileInfo = await fromBuffer(req.file.buffer);
  const allowedTypes = ['jpg', 'png', 'pdf'];

  if (!fileInfo || !allowedTypes.includes(fileInfo.ext)) {
    return res.status(400).json({ error: 'Format berkas ditolak! Hanya JPG, PNG, atau PDF asli yang diterima.' });
  }

  // 2. Hasilkan nama acak baru
  const randomName = `${crypto.randomUUID()}.${fileInfo.ext}`;
  const saveDirectory = '/var/app/private_uploads';
  const finalPath = path.join(saveDirectory, randomName);

  // 3. Simpan berkas ke direktori yang tidak bisa dieksekusi langsung
  await fs.writeFile(finalPath, req.file.buffer);

  return res.json({ message: 'Berkas berhasil diunggah dengan aman', fileId: randomName });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Unggah Webshell Palsu
Coba unggah file skrip berekstensi `.php` atau `.sh`:
```bash
curl -X POST http://localhost:8080/api/upload \
  -F "file=@malicious.php"
```
- **Kondisi Rentan**: Server merespons `200 OK` dan berkas tersimpan di web root.
- **Kondisi Aman**: Server merespons `400 Bad Request` ("Tipe berkas tidak diizinkan atau berbahaya").

---

## Referensi

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [PortSwigger: File Upload Vulnerabilities](https://portswigger.net/web-security/file-upload)
- [MITRE CWE-434](https://cwe.mitre.org/data/definitions/434.html)
