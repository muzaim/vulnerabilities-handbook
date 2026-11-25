# Insecure Deserialization (Deserialisasi Tidak Aman)

> **Metadata Keamanan:**
> - **CWE**: [CWE-502: Deserialization of Untrusted Data](https://cwe.mitre.org/data/definitions/502.html)
> - **OWASP**: [A08:2021 – Software and Data Integrity Failures](https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 8.5 - 9.8)

---

## 1. Pengertian

**Insecure Deserialization** adalah celah keamanan tingkat tinggi yang terjadi ketika aplikasi merekonstruksi (*deserialize*) data yang diterima dari pengguna menjadi objek di memori sistem tanpa memverifikasi integritas atau keabsahan objek tersebut, sehingga penyerang dapat memanipulasi alur logika aplikasi atau mengeksekusi kode berbahaya secara langsung (*Remote Code Execution*).

**Konsep Dasar:**
- **Serialisasi (Serialization)**: Mengubah objek di memori (seperti variabel, fungsi, state) menjadi format byte atau teks agar bisa disimpan ke disk atau dikirim lewat jaringan.
- **Deserialisasi (Deserialization)**: Kebalikannya, membaca format byte/teks tersebut dan membangkitkannya kembali menjadi objek aktif di memori komputer.

**Analogi Sederhana:**  
Bayangkan Anda membeli lemari pakaian dari toko mebel dalam bentuk kemasan kardus belum dirakit (*flat-pack*). Anda membawa kardus itu pulang dan merakitnya mengikuti instruksi di dalam kardus.  
Jika ada orang jahat mengganti isi kardus tersebut dengan bom rakitan beserta instruksi: *"Saat sekrup ini diputar, picu ledakan"*, dan Anda merakitnya tanpa memeriksa apakah isinya benar-benar potongan kayu lemari, Anda akan meledakkan rumah Anda sendiri saat merakitnya.

---

## 2. Cara Kerja

Banyak bahasa pemrograman (seperti Java `ObjectInputStream`, Python `pickle`, PHP `unserialize()`, Ruby `Marshal`, atau library Node.js seperti `node-serialize`) memiliki kemampuan untuk mengemas objek lengkap beserta metode dan kelasnya.

Ketika proses deserialisasi berjalan:
1. Aplikasi membaca data serial yang dikirimkan pengguna (misalnya melalui cookie base64 atau body HTTP).
2. Runtime bahasa memanggil fungsi inisialisasi bawaan (*magic methods* / constructor / destructor seperti `__destruct()`, `readObject()`, atau getter otomatis).
3. Penyerang menyusun struktur data berantai yang disebut **Gadget Chain**: memanfaatkan potongan-potongan kode legal yang sudah ada di dalam library aplikasi untuk secara bertahap memicu eksekusi perintah shell sistem (`exec` / `system`).

---

## 3. Penyebab

- **Menerima Objek Serialisasi Biner/Native dari Pengguna Luar**: Menggunakan pustaka serialisasi bahasa (seperti `pickle` di Python, `unserialize` di PHP, atau `node-serialize` di JavaScript) untuk memproses data dari peramban atau API.
- **Ketiadaan Tanda Tangan Digital Kriptografi**: Tidak menyertakan tanda tangan HMAC untuk membuktikan bahwa data serial tersebut belum pernah dimodifikasi sejak dibuat oleh server.
- **Mengira Enkripsi Base64 Sama dengan Enkripsi Rahasia**: Mengira data aman hanya karena di-encode ke base64 (padahal base64 sangat mudah didekode dan diedit).

---

## 4. Skenario Serangan

1. **Analisis Cookie**: Penyerang melihat cookie sesi bernilai aneh: `profile=eyJuYW1lIjoiQnVkaSIsInJvbGUiOiJ1c2VyIn0%3D`.
2. **Dekode Data**: Penyerang mendekode base64 dan melihat struktur objek serial dari library `node-serialize`.
3. **Penyusunan Payload IIFE (Immediately Invoked Function Expression)**: Penyerang membuat objek serial berbahaya yang memuat fungsi eksekusi mandiri:
   ```json
   {"rce": "_$$ND_FUNC$$_function(){ require('child_process').execSync('rm -rf /tmp/data'); }()"}
   ```
4. **Pengiriman**: Penyerang meng-encode objek tersebut ke base64 dan menaruhnya kembali di cookie sesi.
5. **Eksekusi di Server**: Saat server membaca cookie dan memanggil `unserialize()`, fungsi tersebut langsung dieksekusi di proses backend.

---

## 5. Dampak

- **Remote Code Execution (RCE) Instan**: Penyerang memiliki hak akses penuh ke server web.
- **Manipulasi State Aplikasi**: Mengubah status autentikasi dari pengguna biasa menjadi administrator di memori.
- **Denial of Service (DoS)**: Mengirimkan objek melingkar (*circular reference*) atau payload *Billion Laughs* yang menghabiskan memori RAM server seketika.

---

## 6. Bagian Sistem yang Terdampak

- Mekanisme penyimpanan sesi berbasis cookie objek serial.
- Antrean pesan (*Message Brokers* seperti RabbitMQ / Kafka) yang menerima payload serialized objek antar microservices.
- Caching layer (Redis / Memcached) yang menyimpan objek bahasa mentah.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Pelaporan Pajak Keuangan TaxEasy:**  
TaxEasy mengizinkan pengguna mengekspor draft laporan pajak untuk dilanjutkan nanti. Draft tersebut disimpan dalam bentuk file binary terkompresi.

Aplikasi menggunakan pustaka deserialisasi native untuk membaca kembali file draft tersebut saat pengguna menekan tombol "Lanjutkan Pengisian".

Seorang penyerang menggunakan alat *ysoserial* untuk membuat file draft palsu yang disisipi *gadget chain* Apache Commons Collections. Ketika penyerang mengunggah file draft tersebut ke portal TaxEasy, proses deserialisasi di server otomatis menjalankan perintah untuk membuat akun admin baru di server Linux TaxEasy.

---

## 8. Cara Mendeteksi

- **Pola Tanda Tangan Serialisasi**:
  - Java: Berawalan byte hex `ac ed 00 05` atau base64 `rO0AB...`.
  - Python Pickle: Berawalan `cos\nsystem` atau byte `\x80\x03` / `\x80\x04`.
  - PHP: Berawalan string `O:4:"User":...` atau `a:2:{...}`.
  - Node.js: Berawalan `_$$ND_FUNC$$_`.
- **SAST**: Pemindai kode otomatis akan memberikan peringatan merah (*Critical Alert*) jika menemukan fungsi `unserialize()`, `pickle.loads()`, atau `yaml.load(Loader=yaml.Loader)`.

---

## 9. Pencegahan

1. **Gunakan Format Pertukaran Data Murni (JSON atau Protocol Buffers) (Wajib Utama)**:  
   Hindari format serialisasi native yang bisa membungkus perilaku/fungsi (*behavior*). Gunakan format murni data seperti **JSON**, **Protocol Buffers**, atau **MessagePack** yang hanya memuat tipe data primitif (string, number, boolean, array).
2. **Tanda Tangani Data dengan HMAC Kriptografis**:  
   Jika Anda terpaksa menyimpan data state di sisi klien, selalu tandatangani data tersebut menggunakan HMAC (seperti SHA-256) dengan kunci rahasia server. Jika data diubah 1 karakter saja oleh penyerang, tanda tangan akan rusak dan proses deserialisasi wajib dibatalkan sebelum dijalankan.
3. **Terapkan Validasi Tipe yang Sangat Ketat**:  
   Gunakan skema validator (seperti Zod di TypeScript) untuk memastikan struktur objek hasil parsing murni sesuai dengan tipe yang diharapkan.

---

## 10. Penanganan

1. **Cabut Seluruh Sesi Aktif Berbasis Serialisasi**: Hapus seluruh cookie sesi bermasalah.
2. **Ganti Fungsi Deserialisasi dengan JSON**: Refactor kode untuk menggunakan `JSON.parse()` murni.
3. **Isolasi Server**: Jika server terindikasi menjalankan payload deserialisasi, lakukan investigasi forensik untuk mencari webshell yang ditinggalkan.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Penggunaan JSON yang Aman vs Gob/Encoding)

```go
package main

import (
	"encoding/json"
	"errors"
	"net/http"
)

type UserSession struct {
	UserID   string `json:"user_id"`
	Username string `json:"username"`
	IsAdmin  bool   `json:"is_admin"`
}

// ✅ KODE AMAN: Go secara default menggunakan encoding/json yang aman dari eksekusi kode acak
// JSON murni hanya membaca struktur data teks, tidak pernah mengeksekusi fungsi/metode berbahaya di memori
func DeserializeSessionSecure(rawJSONData []byte) (*UserSession, error) {
	var session UserSession
	err := json.Unmarshal(rawJSONData, &session)
	if err != nil {
		return nil, errors.New("data sesi korup atau tidak valid")
	}

	// Validasi tambahan: Pastikan data masuk akal
	if session.UserID == "" {
		return nil, errors.New("ID pengguna kosong")
	}

	return &session, nil
}
```

### B. Contoh dalam TypeScript (Node.js: Bahaya `node-serialize` vs Aman `JSON.parse`)

```typescript
// ❌ KODE SANGAT RENTAN (Vulnerable - JANGAN PERNAH GUNAKAN INI)
// import serialize from 'node-serialize';
// export function parseCookieVulnerable(cookieStr: string) {
//   // Jika cookieStr memuat payload _$$ND_FUNC$$_, fungsi acak akan dieksekusi di server!
//   return serialize.unserialize(Buffer.from(cookieStr, 'base64').toString());
// }

import { z } from 'zod';

// ✅ KODE AMAN (Secure): Menggunakan JSON murni + Validasi Skema Zod
const SessionSchema = z.object({
  userId: z.string().uuid(),
  username: z.string().alphanumeric(),
  role: z.enum(['user', 'admin']),
});

export function parseSessionSecure(cookiePayload: string) {
  try {
    // 1. Decode string dari base64
    const jsonString = Buffer.from(cookiePayload, 'base64').toString('utf-8');

    // 2. Parse menggunakan JSON murni (JSON.parse kebal terhadap eksekusi fungsi/code execution)
    const rawObject = JSON.parse(jsonString);

    // 3. Validasi bahwa bentuk data cocok seratus persen dengan skema yang diizinkan
    const validSession = SessionSchema.parse(rawObject);

    return validSession;
  } catch (error) {
    throw new Error('Sesi tidak valid atau telah dimanipulasi');
  }
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Penolakan Payload Non-JSON
Kirimkan string biner atau format fungsi ke endpoint deserialisasi:
```bash
curl -X POST http://localhost:8080/api/session \
  -H "Content-Type: application/json" \
  -d '{"userId": "_$$ND_FUNC$$_function(){ console.log(1); }()"}'
```
- **Kondisi Rentan**: Muncul log eksekusi kode di terminal server.
- **Kondisi Aman**: Sistem mengembalikan error `400 Bad Request` karena parser memperlakukan string tersebut murni sebagai teks biasa dan validasi skema menolak format UUID yang salah.

---

## Referensi

- [OWASP Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)
- [PortSwigger: Insecure Deserialization](https://portswigger.net/web-security/deserialization)
- [MITRE CWE-502](https://cwe.mitre.org/data/definitions/502.html)
