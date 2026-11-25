# Server-Side Request Forgery (SSRF)

> **Metadata Keamanan:**
> - **CWE**: [CWE-918: Server-Side Request Forgery (SSRF)](https://cwe.mitre.org/data/definitions/918.html)
> - **OWASP**: [A10:2021 – Server-Side Request Forgery (SSRF)](https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 8.0 - 9.8)

---

## 1. Pengertian

**Server-Side Request Forgery (SSRF)** adalah celah keamanan di mana penyerang dapat memanipulasi server web agar mengirimkan permintaan jaringan (*network request*) HTTP/TCP ke alamat yang ditentukan penyerang, khususnya ke jaringan internal perusahaan yang terlindungi atau ke layanan metadata infrastruktur cloud.

**Analogi Sederhana:**  
Bayangkan sebuah kantor rahasia dengan satpam di pintu depan. Warga umum dilarang masuk ke dalam gedung. Namun, kantor tersebut menyediakan layanan: Anda boleh menitipkan surat di loket dan meminta kurir internal kantor mengantarkannya ke alamat mana pun.  
Seorang penyerang menitipkan catatan: *"Tolong ambilkan dokumen di meja brankas bos di lantai 3 dan bawa ke sini."*  
Karena kurir tersebut adalah orang dalam kantor (memiliki akses internal), kurir itu dengan polos mengambil dokumen di brankas dan menyerahkannya ke penyerang di luar gerbang.

---

## 2. Cara Kerja

Banyak aplikasi web memiliki fitur yang membutuhkan server untuk mengambil data dari internet:
- Mengambil gambar profil dari URL eksternal (*import avatar from URL*).
- Menampilkan pratinjau tautan (*link preview* seperti di WhatsApp/Slack).
- Mengunduh webhook dari layanan pembayaran.
- Menghasilkan file PDF dari halaman web.

Celah SSRF terjadi ketika aplikasi menerima parameter URL dari pengguna dan langsung melakukan request menggunakan library HTTP server (seperti `curl`, `http.Get()`, atau `fetch()`) **tanpa membatasi alamat IP tujuan**:

```
[ Penyerang ] ---> Mengirim input URL: "http://169.254.169.254/latest/meta-data/"
                         │
                         ▼
             [ Server Web (Cloud AWS/GCP) ]
                         │
                         ▼ (Server meminta data ke IP internal lokal)
             [ Layanan Metadata Cloud ]
                         │
                         ▼ (Mengembalikan Kunci Akses AWS Kredensial!)
[ Penyerang ] <--- Server menampilkan respon metadata ke layar
```

---

## 3. Penyebab

- **Tidak Memvalidasi Alamat IP Tujuan**: Membiarkan server mengakses alamat IP *loopback* (`127.0.0.1`, `localhost`) atau IP privat (*private network* RFC 1918 seperti `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).
- **Ketiadaan Proteksi Cloud Metadata Service**: Tidak memblokir akses ke alamat metadata cloud `169.254.169.254` (AWS, Google Cloud, Azure, DigitalOcean).
- **Menerima Skema Protokol Berbahaya**: Mengizinkan skema selain `http://` dan `https://`, seperti `file://` (membaca berkas server langsung), `gopher://`, atau `ftp://`.
- **Rentan terhadap DNS Rebinding**: Hanya memeriksa domain di awal, tetapi saat kueri kedua dilakukan domain sudah diarahkan penyerang ke IP lokal `127.0.0.1`.

---

## 4. Skenario Serangan

1. **Eksplorasi Fitur**: Penyerang menemukan fitur "Import Gambar Produk dari URL" di dashboard e-commerce: `POST /api/products/import-image` dengan body `{"imageUrl": "https://example.com/foto.jpg"}`.
2. **Uji Coba IP Internal**: Penyerang mengirimkan `imageUrl: "http://127.0.0.1:8080/admin"`. Server merespons dengan menampilkan HTML halaman admin lokal yang seharusnya tidak bisa diakses dari internet publik.
3. **Penyedotan Kredensial Cloud**: Penyerang mengirimkan URL metadata AWS:  
   `http://169.254.169.254/latest/meta-data/iam/security-credentials/s3-role`.
4. **Pengambilalihan Infrastruktur Cloud**: Server mengembalikan Access Key, Secret Key, dan Token AWS. Penyerang mengonfigurasi AWS CLI miliknya dan mengunduh seluruh database backup dari AWS S3.

---

## 5. Dampak

- **Pencurian Kredensial Cloud (AWS/GCP/Azure)**: Mengakibatkan pengambilalihan seluruh akun cloud infrastruktur perusahaan.
- **Port Scanning Jaringan Internal**: Memetakan port dan layanan database yang berjalan di jaringan privat perusahaan (*intranet*).
- **Remote Code Execution (RCE)**: Pada beberapa kasus, penyerang dapat berinteraksi dengan Redis atau Memcached internal tanpa kata sandi via protokol gopher/HTTP untuk menaruh script eksekusi.

---

## 6. Bagian Sistem yang Terdampak

- Fitur import file / gambar dari URL.
- Fitur Link Preview atau Web Scraper.
- Fitur Webhook callback yang dapat dikustomisasi pengguna.
- Converter HTML to PDF (seperti Headless Chrome, wkhtmltopdf).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Cloud Invoicing SaaS FakturOnline:**  
FakturOnline menyediakan fitur bagi pengguna untuk mengunggah logo perusahaan melalui URL eksternal untuk dicetak di invoice PDF.

Penyerang memasukkan URL logo:
`http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token`
dengan menambahkan header GCP.

Saat PDF faktur digenerate dan diunduh, di tempat gambar logo tercetak teks token akses Google Cloud Platform (GCP). Penyerang menyalin token tersebut dan mendapatkan kendali penuh atas cluster Kubernetes perusahaan.

---

## 8. Cara Mendeteksi

- **Fuzzing URL Parameter**: Masukkan payload alamat localhost (`http://127.0.0.1`, `http://localhost`, `http://[::1]`) dan amati apakah server merespons dengan error koneksi internal.
- **Uji Coba Metadata Cloud**: Masukkan URL `http://169.254.169.254/latest/meta-data/`.
- **Burp Collaborator / Webhook.site**: Kirimkan URL dari domain unik untuk melihat apakah ada permintaan masuk dari alamat IP server target.

---

## 9. Pencegahan

1. **Blokir Seluruh IP Privat dan Loopback (Wajib)**:  
   Validasi alamat IP hasil resolusi DNS. Tolak keras jika IP masuk dalam rentang:
   - `127.0.0.0/8` (Loopback)
   - `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (Private RFC 1918)
   - `169.254.169.254` (Link-Local / Cloud Metadata)
   - `::1` (IPv6 Loopback)
2. **Wajibkan Protokol HTTPS / HTTP Saja**: Tolak skema `file://`, `ftp://`, `gopher://`.
3. **Gunakan Whitelist Domain (Jika Memungkinkan)**: Jika fitur hanya perlu mengambil data dari domain tertentu (misal: hanya dari `cdn.trusted.com`), batasi hanya ke domain tersebut.
4. **Aktifkan AWS IMDSv2**: Pada infrastruktur AWS EC2, wajibkan IMDSv2 yang membutuhkan token sesi HTTP PUT terlebih dahulu sebelum metadata dapat diakses, sehingga mematikan eksploitasi SSRF standar.

---

## 10. Penanganan

1. **Nonaktifkan Fitur Fetch URL Sementara**: Hentikan pemrosesan URL eksternal hingga patch validasi IP diterapkan.
2. **Rotasi Kredensial IAM Cloud Segera**: Jika instance berjalan di cloud, segera putar (*rotate*) seluruh role dan access key yang terpasang pada VM.
3. **Konfigurasi Firewall Egress**: Batasi server web aplikasi agar secara jaringan tidak diizinkan membuat koneksi keluar ke subnet internal kecuali port database yang telah ditentukan.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Validasi IP Resolusi DNS & Proteksi SSRF)

```go
package main

import (
	"errors"
	"fmt"
	"net"
	"net/http"
	"net/url"
	"time"
)

// Memeriksa apakah sebuah IP adalah private, loopback, atau cloud metadata
func isPrivateIP(ip net.IP) bool {
	if ip.IsLoopback() || ip.IsLinkLocalUnicast() || ip.IsLinkLocalMulticast() || ip.IsPrivate() {
		return true
	}
	// Blokir eksplisit 169.254.169.254 (Cloud Metadata)
	if ip.String() == "169.254.169.254" {
		return true
	}
	return false
}

// ✅ KODE AMAN: HTTP Client kustom dengan validasi IP saat DNS dialer
func FetchURLSecure(targetURL string) (*http.Response, error) {
	parsedURL, err := url.Parse(targetURL)
	if err != nil {
		return nil, errors.New("URL tidak valid")
	}

	// 1. Wajibkan hanya protokol HTTP atau HTTPS
	if parsedURL.Scheme != "http" && parsedURL.Scheme != "https" {
		return nil, errors.New("hanya skema HTTP dan HTTPS yang diizinkan")
	}

	// 2. Kustom Dialer untuk memvalidasi setiap IP hasil resolusi DNS sebelum koneksi dibuka
	customTransport := &http.Transport{
		DialContext: (&net.Dialer{
			Timeout: 5 * time.Second,
		}).DialContext,
	}

	client := &http.Client{
		Transport: customTransport,
		Timeout:   10 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			// Periksa kembali IP saat terjadi redirect (mencegah Open Redirect ke SSRF)
			if len(via) >= 3 {
				return errors.New("terlalu banyak redirect")
			}
			return nil
		},
	}

	// Resolusi DNS dan validasi
	ips, err := net.LookupIP(parsedURL.Hostname())
	if err != nil {
		return nil, errors.New("gagal meresolusi hostname")
	}

	for _, ip := range ips {
		if isPrivateIP(ip) {
			return nil, fmt.Errorf("akses ke IP lokal/privat (%s) dilarang!", ip.String())
		}
	}

	return client.Get(targetURL)
}
```

### B. Contoh dalam TypeScript (Node.js & ipaddr.js)

```typescript
import url from 'url';
import dns from 'dns/promises';
import ipaddr from 'ipaddr.js';

// ✅ KODE AMAN: Memvalidasi URL sebelum dilakukan request
export async function validateUrlForSSRF(rawUrl: string): Promise<string> {
  const parsed = new URL(rawUrl);

  // 1. Validasi protokol
  if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') {
    throw new Error('Protokol tidak diizinkan. Hanya HTTP/HTTPS.');
  }

  // 2. Resolusi Hostname ke IP
  const addresses = await dns.lookup(parsed.hostname, { all: true });

  for (const record of addresses) {
    const addr = ipaddr.parse(record.address);
    const range = addr.range();

    // 3. Blokir jika alamat masuk kategori privat, loopback, atau linkLocal
    const forbiddenRanges = ['loopback', 'private', 'linkLocal', 'carrierGradeNat'];
    if (forbiddenRanges.includes(range) || record.address === '169.254.169.254') {
      throw new Error(`Akses ke alamat IP privat/internal (${record.address}) diblokir!`);
    }
  }

  return rawUrl;
}

// Penggunaan aman:
export async function safeFetch(targetUrl: string) {
  const safeUrl = await validateUrlForSSRF(targetUrl);
  const response = await fetch(safeUrl);
  return response.text();
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Penolakan IP Internal
Coba lakukan impor berkas menggunakan alamat loopback dan cloud metadata:
```bash
curl -X POST http://localhost:8080/api/import \
  -H "Content-Type: application/json" \
  -d '{"url": "http://127.0.0.1:22"}'

curl -X POST http://localhost:8080/api/import \
  -H "Content-Type: application/json" \
  -d '{"url": "http://169.254.169.254/latest/meta-data/"}'
```
- **Kondisi Rentan**: Sistem mengembalikan banner SSH (`SSH-2.0-OpenSSH...`) atau metadata cloud.
- **Kondisi Aman**: Sistem mengembalikan error `400 Bad Request` ("Akses ke alamat IP privat/internal diblokir").

---

## Referensi

- [OWASP Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger: Server-side Request Forgery (SSRF)](https://portswigger.net/web-security/ssrf)
- [AWS Security: What is IMDSv2?](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/)
