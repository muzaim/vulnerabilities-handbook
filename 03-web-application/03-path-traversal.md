# Path Traversal (Directory Traversal)

> **Metadata Keamanan:**
> - **CWE**: [CWE-22: Improper Limitation of a Pathname to a Restricted Directory ('Path Traversal')](https://cwe.mitre.org/data/definitions/22.html)
> - **OWASP**: [A01:2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.1)

---

## 1. Pengertian

**Path Traversal** (atau *Directory Traversal*) adalah celah keamanan di mana penyerang memanipulasi parameter nama file atau jalur direktori untuk keluar dari folder yang diizinkan dan mengakses berkas-berkas penting di bagian lain dari sistem operasi server (seperti file konfigurasi, kode sumber, atau file sistem `/etc/passwd`).

Serangan ini sering dikenal juga sebagai serangan **Dot-Dot-Slash (`../`)**.

**Analogi Sederhana:**  
Bayangkan Anda berkunjung ke perpustakaan umum dan meminta petugas:  
"Tolong ambilkan buku di Rak Anak-Anak nomor 5."  
Namun Anda mengubah permintaan menjadi:  
"Tolong mundur 3 langkah keluar dari ruangan ini, belok kanan ke ruang arsip rahasia direktur, lalu ambilkan berkas laporan keuangan di laci mejanya."  
Jika petugas menurutinya secara membabi buta tanpa memastikan bahwa langkah kakinya hanya boleh berada di dalam Rak Anak-Anak, petugas tersebut terkena *path traversal*.

---

## 2. Cara Kerja

Pada sistem operasi seperti Linux dan Windows:
- Simbol `.` berarti "direktori saat ini".
- Simbol `..` berarti "naik satu tingkat ke folder induk (parent directory)".

Aplikasi sering kali menyajikan berkas statis (seperti gambar profil, invoice, atau laporan) melalui parameter query:
```http
GET /download?file=laporan_januari.pdf
```
Di sisi backend, developer menyusun path berkas:
```text
filePath = "/var/www/uploads/" + input_file
```

Jika pengguna memasukkan nama file biasa, path menjadi: `/var/www/uploads/laporan_januari.pdf` (Aman).  
Namun jika penyerang memasukkan:
```text
file=../../../../etc/passwd
```
Path yang dibentuk server menjadi:
```text
/var/www/uploads/../../../../etc/passwd  ===>  /etc/passwd
```
Server naik keluar dari folder `uploads`, melewati folder `www`, melewati folder `var`, mencapai root direktori `/`, dan membaca file `/etc/passwd` sistem operasi!

---

## 3. Penyebab

- **Penggabungan String Path Mentah**: Menggabungkan input pengguna langsung dengan direktori dasar tanpa melakukan sanitasi jalur.
- **Ketiadaan Pengecekan Batas Jalur (*Canonicalization / Sandboxing*)**: Tidak memeriksa apakah path akhir yang sudah dinormalisasi masih berada di dalam folder yang dituju.
- **Bypass Filter Sederhana**: Menggunakan filter string naif seperti menghapus `../` satu kali saja. Penyerang dapat mengakalinya dengan teknik nested traversal: `....//` (jika `../` di tengah dihapus, sisanya kembali membentuk `../`) atau encoding URL (`%2e%2e%2f`).

---

## 4. Skenario Serangan

1. **Identifikasi Endpoint Berkas**: Penyerang melihat endpoint pengunduhan faktur: `GET /api/invoices/download?filename=inv-2023.pdf`.
2. **Penyisipan Karakter Traversal**: Penyerang mencoba: `filename=../../../../etc/passwd`.
3. **Analisis Output**: Server mengembalikan file teks `/etc/passwd` yang berisi daftar seluruh akun pengguna sistem Linux.
4. **Pencurian Kunci Rahasia**: Penyerang mencoba membaca file konfigurasi aplikasi: `filename=../../../../var/www/html/.env` untuk mencuri password database dan secret key JWT.

---

## 5. Dampak

- **Kebocoran Informasi Sensitif Paling Vital**: Membaca file `.env`, file konfigurasi server, sertifikat SSL privat, dan source code aplikasi.
- **Pencurian Hash Password Sistem**: Membaca file shadow/passwd sistem operasi.
- **Potensi Remote Code Execution (RCE)**: Jika digabungkan dengan fitur upload (misal: menulis file dengan nama `../../var/spool/cron/root` untuk membuat cron job jahat).

---

## 6. Bagian Sistem yang Terdampak

- Endpoint pengunduhan berkas (`/download?file=...`, `/view?path=...`).
- Modul pembacaan template tampilan dinamis (`/page?view=about`).
- Penanganan berkas gambar profil atau lampiran email.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Manajemen Dokumen Karyawan HR-Cloud:**  
Aplikasi HR-Cloud memiliki fitur bagi HRD untuk melihat slip gaji karyawan dengan URL:
`GET /view-slip?doc=slip_desember.pdf`

Backend aplikasi mencari file di: `/opt/app/storage/slips/` + `doc`.

Seorang staf IT magang iseng memasukkan:
`doc=../../../../etc/shadow`
Meskipun file shadow dilindungi hak akses root, ia kemudian mencoba:
`doc=../../../../opt/app/.env`

Server merespons dan menampilkan isi file `.env` yang memuat password database PostgreSQL produksi dan API Key AWS S3. Staf tersebut menyadari bahwa ia baru saja menemukan celah pembacaan berkas sewenang-wenang (*Arbitrary File Read*).

---

## 8. Cara Mendeteksi

- **Fuzzing Simbol Jalur**: Uji parameter file dengan variasi traversal:
  - `../`
  - `..\` (pada Windows)
  - `..;/`
  - `%2e%2e%2f` (URL Encoded)
  - `%252e%252e%252f` (Double URL Encoded)
- **SAST**: Pemindai kode mendeteksi pemanggilan fungsi pembaca berkas seperti `os.Open()` di Go atau `fs.readFile()` di Node.js yang menerima variabel dari request HTTP tanpa melalui fungsi sanitasi path.

---

## 9. Pencegahan

1. **Gunakan ID Database Alih-alih Nama File Mentah (Sangat Direkomendasikan)**:  
   Jangan biarkan pengguna menentukan nama berkas di disk secara langsung. Gunakan ID acak (misal: `/download?file_id=8812`) dan simpan nama file fisik yang terenkripsi di database.
2. **Normalisasi dan Validasi Batas Direktori (Path Canonicalization)**:  
   Gunakan fungsi normalisasi bawaan bahasa (seperti `filepath.Clean()` di Go atau `path.resolve()` di Node.js), lalu pastikan secara tegas bahwa hasil path diawali oleh folder dasar yang diizinkan (*Base Directory Check*).
3. **Ambil Nama File Murni (Basename Only)**:  
   Gunakan fungsi `filepath.Base()` di Go atau `path.basename()` di Node.js untuk membuang seluruh karakter folder (`../`) dan hanya mengambil nama filenya saja.

---

## 10. Penanganan

1. **Matikan Endpoint Download Sementara**: Hentikan fitur download yang bermasalah.
2. **Audit File yang Pernah Diakses**: Periksa web server access log untuk mencari request yang memuat pola `..` atau `%2e%2e`.
3. **Pindahkan Penyimpanan ke Object Storage (S3 / GCS)**: Simpan berkas di cloud storage terisolasi alih-alih di sistem berkas lokal server web.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"errors"
	"net/http"
	"os"
	"path/filepath"
	"strings"
)

const safeBaseDir = "/var/www/uploads"

// ❌ KODE RENTAN (Vulnerable)
// Langsung menggabungkan path tanpa normalisasi batas direktori
func DownloadVulnerable(w http.ResponseWriter, r *http.Request) {
	filename := r.URL.Query().Get("file")
	// Jika filename = "../../../../etc/passwd"
	targetPath := filepath.Join(safeBaseDir, filename)
	data, err := os.ReadFile(targetPath)
	if err != nil {
		http.Error(w, "File tidak ditemukan", http.StatusNotFound)
		return
	}
	w.Write(data)
}

// ✅ KODE AMAN (Secure)
// 1. Ekstrak nama file murni menggunakan filepath.Base
// 2. Normalisasi dan verifikasi bahwa path final tetap berada di dalam safeBaseDir
func DownloadSecure(w http.ResponseWriter, r *http.Request) {
	rawFilename := r.URL.Query().Get("file")

	// Cara 1: Hanya ambil nama berkas murni (membuang semua tanda ../)
	cleanName := filepath.Base(rawFilename)

	// Gabungkan dengan direktori aman
	targetPath := filepath.Join(safeBaseDir, cleanName)

	// Cara 2: Verifikasi batas direktori (Canonical Path Check)
	cleanTarget := filepath.Clean(targetPath)
	if !strings.HasPrefix(cleanTarget, safeBaseDir) {
		http.Error(w, "Akses jalur berkas tidak diizinkan", http.StatusForbidden)
		return
	}

	data, err := os.ReadFile(cleanTarget)
	if err != nil {
		http.Error(w, "File tidak ditemukan", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/octet-stream")
	w.Write(data)
}
```

### B. Contoh dalam TypeScript (Node.js & `path`)

```typescript
import path from 'path';
import fs from 'fs/promises';
import { Request, Response } from 'express';

const UPLOAD_DIR = path.resolve('/var/www/uploads');

// ❌ KODE RENTAN (Vulnerable)
export async function downloadFileVulnerable(req: Request, res: Response) {
  const filename = req.query.file as string;
  // Jika filename = "../../../.env"
  const filePath = path.join(UPLOAD_DIR, filename);
  const content = await fs.readFile(filePath);
  res.send(content);
}

// ✅ KODE AMAN (Secure)
export async function downloadFileSecure(req: Request, res: Response) {
  const filename = req.query.file as string;
  if (!filename) {
    return res.status(400).json({ error: 'Nama file harus disertakan' });
  }

  // 1. Ambil nama file murni (menghilangkan folder dan karakter ../)
  const safeFilename = path.basename(filename);

  // 2. Bentuk path absolut yang dinormalisasi
  const safePath = path.resolve(UPLOAD_DIR, safeFilename);

  // 3. Verifikasi apakah path aman masih berawalan folder UPLOAD_DIR
  if (!safePath.startsWith(UPLOAD_DIR)) {
    return res.status(403).json({ error: 'Akses direktori ditolak' });
  }

  try {
    const content = await fs.readFile(safePath);
    res.setHeader('Content-Disposition', `attachment; filename="${safeFilename}"`);
    return res.send(content);
  } catch (error) {
    return res.status(404).json({ error: 'File tidak ditemukan' });
  }
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian dengan cURL
Uji coba traversal menuju file sistem lokal:
```bash
curl -X GET "http://localhost:8080/api/download?file=../../../../etc/passwd"
```
- **Kondisi Rentan**: Isi berkas `root:x:0:0:...` muncul di terminal.
- **Kondisi Aman**: Sistem mengembalikan status `403 Forbidden` atau `404 Not Found` (karena sistem mencari file bernama `passwd` di dalam folder uploads, bukan di root sistem operasi).

---

## Referensi

- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [PortSwigger: Directory Traversal](https://portswigger.net/web-security/file-path-traversal)
- [MITRE CWE-22](https://cwe.mitre.org/data/definitions/22.html)
