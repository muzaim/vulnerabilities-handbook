# Sensitive Data Exposure (Paparan Data Sensitif)

> **Metadata Keamanan:**
> - **CWE**: [CWE-311: Missing Encryption of Sensitive Data](https://cwe.mitre.org/data/definitions/311.html) & [CWE-319: Cleartext Transmission of Sensitive Information](https://cwe.mitre.org/data/definitions/319.html)
> - **OWASP**: [A02:2021 – Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 9.0)

---

## 1. Pengertian

**Sensitive Data Exposure** (Paparan Data Sensitif) adalah kondisi di mana data penting dan rahasia pengguna (seperti nomor kartu kredit, NIK KTP, riwayat kesehatan, atau kredensial) tidak dilindungi secara memadai, baik saat data sedang dikirimkan melalui jaringan internet (*data in transit*) maupun saat data disimpan di media penyimpanan (*data at rest*).

**Analogi Sederhana:**  
Bayangkan Anda mengirimkan buku tabungan dan nomor PIN ATM Anda ke saudara di luar kota.  
- **Aman**: Anda memasukkannya ke dalam kotak baja terkunci gembok kombinasi tebal dan mengirimkannya via kurir berpengawal.  
- **Sensitive Data Exposure**: Anda menuliskan nomor rekening dan nomor PIN di selembar kartu pos terbuka dan mengirimkannya lewat pos biasa, sehingga setiap kurir atau tetangga yang memegang kartu pos itu bisa membaca seluruh data rahasia Anda.

---

## 2. Cara Kerja

Data dapat bocor dalam dua kondisi utama:

### A. Data in Transit (Saat Bergerak di Jaringan)
Ketika aplikasi berkomunikasi melalui protokol HTTP biasa (tanpa gembok ganda HTTPS / TLS):
- Komunikasi berjalan dalam teks polos (*plaintext*).
- Siapapun yang berada di jaringan yang sama (misal jaringan Wi-Fi kafe, ISP, atau penyerang Man-in-the-Middle) dapat menggunakan software sniffer (seperti Wireshark) untuk menyalin nomor kartu kredit, kata sandi, dan token login pengguna.

### B. Data at Rest (Saat Tersimpan di Server / Database)
Ketika data disimpan di disk atau database tanpa enkripsi:
- Jika berkas backup database dicuri atau hard disk server dibuang tanpa diformat aman, penyerang langsung membaca jutaan data NIK dan rekam medis pengguna.
- Log server mencetak data sensitif pengguna (misal: mencatat query URL yang memuat nomor kartu debit).

---

## 3. Penyebab

- **Tidak Menggunakan atau Mengabaikan Penegakan HTTPS**: Membiarkan pengguna mengakses situs melalui HTTP biasa tanpa memaksakan pengalihan ke HTTPS (*Strict-Transport-Security / HSTS*).
- **Mencatat Data Sensitif ke Berkas Log (Log Poisoning / PII Leak)**: Mencetak seluruh body request atau nomor kartu kredit ke file `access.log` atau console.
- **Ketiadaan Enkripsi Database Tingkat Kolom**: Menyimpan data sensitif regulasi (seperti nomor identitas kependudukan, nomor rekening) tanpa fungsi enkripsi simetris (misal AES-256).
- **Protokol TLS Usang**: Mengizinkan koneksi dengan SSL v3, TLS 1.0, atau TLS 1.1 yang rentan terhadap serangan downgrade (seperti POODLE).

---

## 4. Skenario Serangan

1. **Penyadapan di Wi-Fi Publik (Man-in-the-Middle - MitM)**: Korban terhubung ke jaringan Wi-Fi gratis di bandara. Penyerang menjalankan teknik ARP Spoofing.
2. **Koneksi HTTP Tanpa Enkripsi**: Aplikasi mobile korban mengirimkan data login ke endpoint `http://api.layanan.example/login` (bukan https).
3. **Penyalinan Paket Data**: Penyerang menangkap paket data HTTP menggunakan Wireshark. Karena tidak ada enkripsi TLS, email dan password korban terlihat dalam teks polos (*cleartext*).
4. **Pengambilalihan Akun**: Penyerang langsung menggunakan kredensial tersebut untuk masuk ke akun korban.

---

## 5. Dampak

- **Denda Regulasi Raksasa**: Pelanggaran terhadap UU Pelindungan Data Pribadi (UU PDP), GDPR, atau standar PCI-DSS bagi industri perbankan yang dapat berujung sanksi hingga miliaran rupiah dan pencabutan izin operasional.
- **Pencurian Identitas Massal (Identity Theft)**: Data NIK dan foto KTP disalahgunakan untuk pinjaman online ilegal atau pemalsuan akun bank.
- **Kerugian Finansial Korban**: Kartu kredit disedot oleh komplotan pembobol kartu (*carding*).

---

## 6. Bagian Sistem yang Terdampak

- Seluruh komunikasi client-server (web dan mobile).
- File log aplikasi, log error, dan pipeline observabilitas (Sentry, Datadog).
- Database cadangan (*backup files* dan snapshots cloud).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Rekam Medis KlinikHarmoni:**  
KlinikHarmoni mengintegrasikan sistem pencatatan rekam medis dengan server cloud.

Developer memasang logger debug:
```javascript
console.log('Incoming Request:', req.url, req.body);
```

Setiap kali pasien memperbarui riwayat penyakit, seluruh diagnosis dokter, nomor BPJS, dan nama obat dicetak ke file `app.log`. File log tersebut secara berkala di-backup ke server penyimpanan publik yang lupa diproteksi kata sandi. Peneliti keamanan menemukan 500.000 data riwayat penyakit pasien (termasuk status HIV dan penyakit kejiwaan) dapat diunduh bebas dari internet oleh siapa saja.

---

## 8. Cara Mendeteksi

- **Pemeriksaan SSL/TLS**: Gunakan alat pemindai SSL Labs (Qualys SSL Labs) untuk memastikan konfigurasi TLS server berperingkat A+ dan menolak protokol usang.
- **Audit Berkas Log**: Cari kata kunci sensitif di dalam file log: `grep -E "password|cvv|credit_card|nik" /var/log/app.log`. Jika ditemukan, log mengalami kebocoran PII.
- **Pemeriksaan Header HSTS**: Periksa apakah ada header `Strict-Transport-Security` pada respon server.

---

## 9. Pencegahan

1. **Wajibkan Enkripsi TLS Modern (HTTPS Everywhere)**:  
   Tolak seluruh koneksi HTTP biasa. Gunakan TLS 1.3 atau minimal TLS 1.2 dengan cipher suite modern.
2. **Aktifkan HTTP Strict Transport Security (HSTS)**:  
   Kirimkan header `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` agar browser tidak pernah mencoba membuka situs Anda lewat HTTP.
3. **Masking / Sanitasi Log (Wajib)**:  
   Sensor data sensitif sebelum dicetak ke log (misal: nomor kartu kredit `4111-XXXX-XXXX-1111` atau kata sandi `***`).
4. **Enkripsi Data Sensitif Saat Disimpan (Encryption at Rest)**:  
   Gunakan enkripsi terotentikasi seperti AES-256-GCM untuk kolom-kolom data krusial di database.

---

## 10. Penanganan

1. **Paksa Redirect ke HTTPS Segera**: Aktifkan aturan pengalihan otomatis HTTP -> HTTPS di level cloud (Cloudflare/AWS ALB) atau web server (Nginx).
2. **Bersihkan File Log dari Server**: Hapus atau rotasi file log lama yang memuat teks data pribadi mentah.
3. **Notifikasi Korban dan Otoritas**: Jika data identitas warga negara sempat bocor, ikuti prosedur pelaporan insiden sesuai amanat UU PDP.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Masking Data Sensitif pada Logger)

```go
package main

import (
	"fmt"
	"regexp"
)

// Regex untuk mendeteksi 16 digit nomor kartu kredit
var creditCardRegex = regexp.MustCompile(`\b(?:\d[ -]*?){13,16}\b`)

// ✅ KODE AMAN: Fungsi penyensoran data sensitif sebelum dicatat ke log
func MaskSensitiveData(logMessage string) string {
	// Ganti digit di tengah kartu kredit dengan simbol bintang (Masking)
	return creditCardRegex.ReplaceAllStringFunc(logMessage, func(match string) string {
		if len(match) >= 12 {
			return match[:4] + "-XXXX-XXXX-" + match[len(match)-4:]
		}
		return "XXXX-MASKED-XXXX"
	})
}

func main() {
	rawLog := "Transaksi gagal untuk nomor kartu 4532123456789012 atas nama Budi"
	safeLog := MaskSensitiveData(rawLog)
	
	// Hasil cetak log: "Transaksi gagal untuk nomor kartu 4532-XXXX-XXXX-9012 atas nama Budi"
	fmt.Println(safeLog)
}
```

### B. Contoh dalam TypeScript (Enkripsi Data Sensitif AES-256-GCM)

```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
// Kunci rahasia 32 bytes (256 bits) disimpan aman di Environment Variable
const ENCRYPTION_KEY = Buffer.from(process.env.DB_FIELD_KEY || '12345678901234567890123456789012', 'utf-8');

// ✅ KODE AMAN: Enkripsi data sensitif (misal NIK) sebelum disimpan ke Database
export function encryptData(plaintext: string): string {
  const iv = crypto.randomBytes(12); // Initialization Vector 12 bytes untuk GCM
  const cipher = crypto.createCipheriv(ALGORITHM, ENCRYPTION_KEY, iv);

  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag().toString('hex'); // Tanda tangan integritas

  // Gabungkan iv + authTag + cipherText menjadi satu string aman untuk DB
  return `${iv.toString('hex')}:${authTag}:${encrypted}`;
}

// Dekripsi data saat dibutuhkan oleh sistem berwenang
export function decryptData(cipherString: string): string {
  const [ivHex, authTagHex, encryptedText] = cipherString.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');

  const decipher = crypto.createDecipheriv(ALGORITHM, ENCRYPTION_KEY, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Header HSTS dengan cURL
```bash
curl -I https://aplikasi-anda.example
```
- **Kondisi Aman**: Respons memuat header:  
  `Strict-Transport-Security: max-age=31536000; includeSubDomains`.

---

## Referensi

- [OWASP Transport Layer Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
- [MITRE CWE-311: Missing Encryption of Sensitive Data](https://cwe.mitre.org/data/definitions/311.html)
- [Qualys SSL Labs: Best Practices](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices)
