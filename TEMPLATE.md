# Template Standar Penulisan Kerentanan

Dokumen ini merupakan panduan dan template acuan baku yang wajib dipatuhi saat menulis dokumentasi setiap jenis kerentanan dalam **Vulnerabilities Handbook**.

---

```markdown
# [Nama Kerentanan (Bahasa Indonesia & Istilah Umum)]

> **Metadata Keamanan:**
> - **CWE**: CWE-XXX ([Nama CWE Resmi](https://cwe.mitre.org/data/definitions/XXX.html))
> - **OWASP**: A0X:2021 – [Kategori OWASP](tautan) / APIX:2023 – [Kategori API](tautan)
> - **Tingkat Keparahan (Severity)**: Low / Medium / High / Critical (CVSS Estimasi)

---

## 1. Pengertian

Jelaskan apa itu kerentanan ini secara sederhana. Gunakan analogi kehidupan nyata (misalnya sistem loket, kunci pintu, kartu parkir, tanda tangan fisik) agar pembaca non-IT atau pemula langsung memiliki gambaran mental yang jelas sebelum masuk ke aspek teknis.

## 2. Cara Kerja

Jelaskan mekanisme kerja bagaimana celah keamanan ini terjadi di tingkat protokol, aplikasi, atau database. Jelaskan alur data (data flow) dari input pengguna sampai dieksekusi oleh sistem. Gunakan diagram ASCII sederhana jika alurnya bercabang.

## 3. Penyebab

Jelaskan kelalaian developer atau kelemahan arsitektur yang menyebabkan celah ini muncul. Misalnya:
- Kurangnya validasi atau sanitasi input.
- Ketiadaan parameterized queries.
- Kesalahan konfigurasi default.
- Asumsi yang salah terhadap integritas data dari klien.

## 4. Skenario Serangan

Jelaskan langkah demi langkah bagaimana seorang penyerang (attacker) menemukan dan memanfaatkan celah tersebut.
1. Tahap observasi/pengintaian input.
2. Manipulasi parameter / pengiriman payload uji coba.
3. Eksekusi payload yang tidak diinginkan pada sistem target.

## 5. Dampak

Jelaskan kerugian yang timbul jika celah ini berhasil dieksploitasi:
- **Terhadap Pengguna**: Kebocoran data pribadi, pengambilalihan akun.
- **Terhadap Perusahaan / Bisnis**: Kerugian finansial, reputasi hancur, denda regulasi (seperti UU PDP / GDPR).
- **Terhadap Sistem**: Integritas data rusak, server disusupi (RCE), layanan lumpuh (DoS).

## 6. Bagian Sistem yang Terdampak

Sebutkan modul atau komponen aplikasi yang paling rawan terhadap kerentanan ini, seperti:
- Form login / registrasi
- Parameter URL (query string / path variable)
- Header HTTP (Cookie, Authorization, Referer)
- Endpoint API internal maupun publik
- Modul upload dan pengelolaan berkas

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

Gunakan simulasi skenario realistis dalam industri konkret (E-commerce, Sistem Perbankan, SIAKAD Kampus, Rumah Sakit, Sistem Ticketing).

Ceritakan narasi:
- Siapa aktornya (misal: "Budi, seorang mahasiswa...", "Siti, seorang pembeli...").
- Sistem apa yang digunakan.
- Tindakan apa yang dilakukan dan bagaimana sistem merespons secara keliru.

*(Beri catatan: "Skenario ini merupakan simulasi edukatif yang realistis untuk mempermudah pemahaman alur kerentanan.")*

## 8. Cara Mendeteksi

Jelaskan bagaimana developer, QA, atau tim keamanan dapat menemukan celah ini:
- **Static Analysis (SAST)**: Pola kode mencurigakan yang dicari linter / scanner.
- **Dynamic Analysis (DAST) & Pentest**: Pengujian black-box pada endpoint berjalan.
- **Manual Code Review**: Titik kritis yang perlu diperiksa manual oleh sesama developer.
- **Audit Logging**: Anomali log yang menandakan adanya upaya eksploitasi.

## 9. Pencegahan

Jelaskan langkah pertahanan sejak dini (*secure by design*):
- Prinsip *defense-in-depth*.
- Penggunaan library atau framework terpercaya yang secara otomatis menangani celah tersebut.
- Konfigurasi keamanan bawaan yang harus diaktifkan.

## 10. Penanganan

Langkah darurat yang harus diambil tim pengembang jika celah ini terdeteksi pada sistem produksi yang sedang berjalan:
1. Tindakan isolasi cepat (Hotfix / WAF rule sementara).
2. Investigasi jejak penyerangan pada log (*incident response*).
3. Penerapan patch permanen dan rilis perbaikan teruji.
4. Notifikasi pihak terkait jika ada kebocoran data.

## 11. Contoh Implementasi

Tampilkan komparasi kode konkret. Sediakan kode dalam **Golang** dan **TypeScript / JavaScript**.

### A. Contoh dalam Golang

```go
// ❌ KODE RENTAN (Vulnerable)
// [Penjelasan singkat letak kerentanan]
...

// ✅ KODE AMAN (Secure)
// [Penjelasan perbaikan yang diterapkan]
...
```

### B. Contoh dalam TypeScript / JavaScript

```typescript
// ❌ KODE RENTAN (Vulnerable)
// [Penjelasan singkat letak kerentanan]
...

// ✅ KODE AMAN (Secure)
// [Penjelasan perbaikan yang diterapkan]
...
```

Jelaskan perbedaan kedua versi kode tersebut dengan bahasa yang mudah dipahami.

## 12. Pengujian dan Verifikasi

Panduan menguji bahwa patch atau mitigasi yang diterapkan benar-benar berhasil menutup celah:
- Contoh unit test / integration test lokal.
- Skenario pengujian manual lokal dengan cURL atau browser tool.
- Verifikasi bahwa fungsionalitas normal aplikasi tetap berjalan tanpa regresi (*no regression*).

---

## Referensi

- [Nama Referensi 1 (OWASP)](tautan)
- [Nama Referensi 2 (MITRE CWE)](tautan)
- [Nama Referensi 3 (PortSwigger Web Security Academy)](tautan)
```
