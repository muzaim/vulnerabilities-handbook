# Vulnerabilities Handbook 🛡️

Panduan komprehensif, praktis, dan ramah pemula tentang berbagai celah keamanan (*vulnerabilities*) yang sering ditemukan pada aplikasi web dan API modern.

---

## 🎯 Tujuan Proyek

Keamanan aplikasi sering kali dianggap sebagai topik yang rumit, menakutkan, dan hanya bisa dipahami oleh praktisi keamanan siber berpengalaman. Buku panduan ini dibuat untuk mengubah pandangan tersebut.

**Vulnerabilities Handbook** bertujuan:
1. Membantu pembaca memahami akar penyebab terjadinya celah keamanan pada sistem digital.
2. Menjelaskan bagaimana celah tersebut dapat dieksploitasi oleh pihak yang tidak bertanggung jawab, apa dampak nyatanya, serta langkah konkret untuk mencegah dan memperbaikinya.
3. Menjembatani komunikasi antara tim teknis (developer, DevOps, QA) dan non-teknis (manajemen produk, pemilik bisnis, pemula) agar memiliki kesadaran keamanan yang selaras.

---

## 👥 Siapa yang Bisa Menggunakan Handbook Ini?

Buku panduan ini dirancang untuk dapat dibaca oleh berbagai kalangan tanpa harus memiliki latar belakang keamanan siber yang mendalam:

- **Pemula & Antusias Teknologi**: Ingin memahami bagaimana aplikasi web bekerja di balik layar dan di mana titik-titik rawannya dengan bantuan analogi sehari-hari.
- **Orang Non-IT & Pemilik Produk (Product Manager / Business Owner)**: Ingin memahami risiko bisnis dari celah keamanan tanpa terbebani istilah teknis yang rumit.
- **Software Engineer & Web Developer**: Ingin melihat perbandingan kode nyata (*vulnerable* vs *secure patch*) dalam **Golang** dan **TypeScript/JavaScript**, serta cara menguji perbaikannya.
- **QA Engineer & Peneliti Keamanan Pemula**: Ingin memperkaya daftar uji keamanan fungsional dan logika bisnis aplikasi.

---

## 🧭 Format Konsisten Pembahasan

Setiap topik kerentanan dalam handbook ini dibahas dengan 12 struktur baku:
1. **Pengertian**: Penjelasan konsep dengan analogi kehidupan sehari-hari.
2. **Cara Kerja**: Alur mekanisme terjadinya celah keamanan.
3. **Penyebab**: Kesalahan logika kode atau arsitektur sistem.
4. **Skenario Serangan**: Langkah konkret bagaimana celah dimanfaatkan.
5. **Dampak**: Kerugian teknis, finansial, dan reputasi.
6. **Bagian Sistem yang Terdampak**: Komponen arsitektur yang rentan.
7. **Contoh Kasus Nyata dalam Sebuah Sistem**: Simulasi skenario sistem nyata (e-commerce, perbankan, akademik, RS).
8. **Cara Mendeteksi**: Teknik deteksi melalui SAST, DAST, code review, atau log audit.
9. **Pencegahan**: Langkah preventif sejak fase desain/coding.
10. **Penanganan**: Tindakan mitigasi jika kerentanan ditemukan di sistem yang sedang berjalan.
11. **Contoh Implementasi**: Komparasi kode rentan vs kode aman dalam **Golang** dan **TypeScript**.
12. **Pengujian dan Verifikasi**: Prosedur validasi lokal untuk memastikan celah tertutup.

---

## 📚 Daftar Kategori & Topik

### [01. Injection](01-injection/README.md)
Celah keamanan akibat data masukan pengguna dieksekusi secara keliru sebagai bagian dari perintah atau kueri sistem.
- [01. SQL Injection (SQLi)](01-injection/01-sql-injection.md): Penyusupan perintah SQL ilegal ke dalam kueri database melalui input pengguna, memungkinkan pembacaan atau perusakan data secara tidak sah.
- [02. NoSQL Injection](01-injection/02-nosql-injection.md): Manipulasi logika kueri database dokumen (MongoDB) menggunakan objek JSON atau operator khusus (`$ne`, `$gt`, `$regex`) untuk mem-bypass autentikasi.
- [03. Command Injection](01-injection/03-command-injection.md): Pengeksekusian perintah sistem operasi (shell Linux/Windows) secara sewenang-wenang melalui input yang diteruskan ke proses sistem.
- [04. Cross-Site Scripting (XSS)](01-injection/04-cross-site-scripting-xss.md): Penyisipan skrip JavaScript berbahaya ke halaman web yang otomatis dieksekusi di browser korban untuk mencuri cookie sesi atau membajak akun.
- [05. Server-Side Template Injection (SSTI)](01-injection/05-server-side-template-injection-ssti.md): Penyuntikan sintaks template engine backend yang dievaluasi di server, berujung pada eksekusi kode jarak jauh (*Remote Code Execution*).

### [02. Authentication & Authorization](02-authentication-authorization/README.md)
Kegagalan dalam membuktikan identitas pengguna (*siapa Anda*) atau membatasi hak akses (*apa yang boleh Anda lakukan*).
- [01. Broken Authentication](02-authentication-authorization/01-broken-authentication.md): Kelemahan pada alur verifikasi login, pengelolaan sesi, atau logout yang memungkinkan penyerang menyamar sebagai pengguna sah.
- [02. Broken Access Control](02-authentication-authorization/02-broken-access-control.md): Kegagalan penegakan aturan hak akses di server yang memungkinkan pengguna biasa mengakses fungsi atau data milik admin atau pihak lain.
- [03. IDOR & BOLA (Insecure Direct Object Reference / Broken Object Level Authorization)](02-authentication-authorization/03-idor-bola.md): Manipulasi ID objek pada parameter URL/API untuk mengakses atau mengubah data milik pengguna lain tanpa validasi kepemilikan.
- [04. Privilege Escalation (Eskalasi Hak Akses)](02-authentication-authorization/04-privilege-escalation.md): Peningkatan hak akses secara ilegal, baik menjadi peran yang lebih tinggi (*vertikal*) maupun mengakses akun selevel (*horizontal*).
- [05. Session Management (Hijacking & Fixation)](02-authentication-authorization/05-session-management.md): Pencurian token sesi aktif korban (*hijacking*) atau pemaksaan ID sesi tertentu sebelum korban login (*fixation*).
- [06. JWT Vulnerabilities](02-authentication-authorization/06-jwt-vulnerabilities.md): Celah verifikasi token JWT, termasuk penyalahgunaan algoritma `none`, kunci rahasia lemah, dan manipulasi klaim data.
- [07. Broken Password Reset](02-authentication-authorization/07-broken-password-reset.md): Kelemahan pada alur pemulihan kata sandi (token mudah ditebak atau manipulasi Host header) yang berujung pengambilalihan akun.
- [08. Brute Force Attack](02-authentication-authorization/08-brute-force.md): Percobaan tebakan kata sandi atau kode PIN secara berulang tanpa adanya batasan frekuensi percobaan (*rate limiting*).
- [09. Credential Stuffing](02-authentication-authorization/09-credential-stuffing.md): Otomatisasi pengujian jutaan pasangan username dan kata sandi hasil kebocoran data (*data breach*) pihak ketiga secara massal.

### [03. Web Application](03-web-application/README.md)
Kerentanan pada interaksi browser-server, pengelolaan berkas, dan konfigurasi protokol web.
- [01. Cross-Site Request Forgery (CSRF)](03-web-application/01-csrf.md): Penjebakan browser korban yang sedang login untuk mengirimkan aksi atau transfer dana tanpa disadari korban saat membuka situs jahat.
- [02. Server-Side Request Forgery (SSRF)](03-web-application/02-ssrf.md): Memanipulasi server backend agar mengirimkan permintaan jaringan ke sistem internal rahasia atau layanan metadata cloud.
- [03. Path Traversal](03-web-application/03-path-traversal.md): Manipulasi parameter jalur berkas (`../`) untuk keluar dari direktori web dan membaca berkas rahasia sistem operasi (seperti `/etc/passwd`).
- [04. Unrestricted File Upload](03-web-application/04-unrestricted-file-upload.md): Pengunggahan berkas skrip berbahaya (seperti webshell) tanpa validasi jenis berkas yang dapat dieksekusi di server.
- [05. Open Redirect](03-web-application/05-open-redirect.md): Pengalihan otomatis pengguna ke situs web penipuan (*phishing*) eksternal menggunakan parameter URL yang tidak divalidasi.
- [06. Clickjacking](03-web-application/06-clickjacking.md): Menjebak pengguna mengklik tombol penting (seperti transfer atau hapus akun) yang dilapisi bingkai iframe transparan tak terlihat.
- [07. CORS Misconfiguration](03-web-application/07-cors-misconfiguration.md): Kesalahan konfigurasi aturan berbagi sumber daya lintas domain yang memungkinkan situs asing membaca data respons API privat.
- [08. Insecure Deserialization](03-web-application/08-insecure-deserialization.md): Rekonstruksi objek data serial yang tidak terpercaya di memori server, memicu manipulasi logika atau eksekusi kode acak (*RCE*).

### [04. API Security](04-api-security/README.md)
Celah spesifik pada arsitektur API RESTful/GraphQL yang melayani aplikasi mobile dan single-page application.
- [01. Broken Function Level Authorization (BFLA)](04-api-security/01-broken-function-level-authorization-bfla.md): Ketiadaan pembatasan peran pada endpoint fungsi sensitif atau administratif (seperti ekspor data massal atau penghapusan sistem).
- [02. Mass Assignment](04-api-security/02-mass-assignment.md): Pemetaan otomatis seluruh properti JSON dari request klien langsung ke database, memungkinkan pengubahan kolom terlarang (`is_admin`, `balance`).
- [03. Excessive Data Exposure](04-api-security/03-excessive-data-exposure.md): Server mengembalikan seluruh atribut objek rahasia (NIK, alamat, hash password) ke API dan hanya mengandalkan frontend untuk menyembunyikannya.
- [04. Unrestricted Resource Consumption](04-api-security/04-unrestricted-resource-consumption.md): Ketiadaan batas ukuran request atau paginasi (`limit=1000000`) yang menyebabkan server kehabisan memori (*OOM*) dan tumbang.
- [05. Improper Error Handling](04-api-security/05-improper-error-handling.md): Bocornya pesan galat mentah, *stack trace*, kueri SQL, atau versi sistem ke respons klien yang memudahkan pemetaan serangan.

### [05. Data & Cryptography](05-data-cryptography/README.md)
Kelemahan dalam enkripsi, penyimpanan kunci rahasia, dan perlindungan data sensitif pengguna.
- [01. Sensitive Data Exposure](05-data-cryptography/01-sensitive-data-exposure.md): Transmisi data tanpa enkripsi TLS yang aman atau pencatatan informasi identitas pribadi (PII) ke dalam berkas log terbuka.
- [02. Hardcoded Secrets](05-data-cryptography/02-hardcoded-secrets.md): Kunci API, password database, atau private key yang ditulis langsung di dalam kode program dan bocor ke repositori Git.
- [03. Insecure Password Storage (Plaintext & Weak Hashing)](05-data-cryptography/03-insecure-password-storage.md): Penyimpanan kata sandi dalam teks polos atau fungsi hash cepat (MD5/SHA1 tanpa salt) yang sangat mudah dibongkar oleh GPU.
- [04. Weak Encryption](05-data-cryptography/04-weak-encryption.md): Penggunaan cipher usang (DES, RC4), mode operasi yang memperlihatkan pola data (AES-ECB), atau penggunaan ulang IV acak.

### [06. Business Logic](06-business-logic/README.md)
Kelemahan pada alur logika bisnis yang dirancang sah tetapi memiliki celah urutan proses atau persaingan eksekusi.
- [01. Race Condition & TOCTOU (Time-of-Check to Time-of-Use)](06-business-logic/01-race-condition-toctou.md): Manipulasi jeda waktu antara pengecekan saldo/kupon dan pemotongannya secara konkuren, memungkinkan pencairan ganda (*double spending*).
- [02. Replay Attack](06-business-logic/02-replay-attack.md): Pengiriman ulang paket transaksi sah masa lalu berulang kali tanpa perubahan untuk menduplikasi tindakan tanpa izin.
- [03. Price Manipulation & Payment Bypass](06-business-logic/03-price-manipulation-payment-bypass.md): Pengubahan parameter harga di sisi klien, kuantitas negatif, atau pemalsuan callback payment gateway untuk checkout tanpa bayar.

---

## 📖 Referensi Resmi & Standar Industri

Seluruh materi dalam handbook ini merujuk pada standar keamanan siber internasional:
- **OWASP Top 10 Web Application Security Risks**: [https://owasp.org/www-project-top-ten/](https://owasp.org/www-project-top-ten/)
- **OWASP API Security Top 10**: [https://owasp.org/www-project-api-security/](https://owasp.org/www-project-api-security/)
- **MITRE Common Weakness Enumeration (CWE)**: [https://cwe.mitre.org/](https://cwe.mitre.org/)
- **PortSwigger Web Security Academy**: [https://portswigger.net/web-security](https://portswigger.net/web-security)
- **NIST Computer Security Resource Center**: [https://csrc.nist.gov/](https://csrc.nist.gov/)

---

## ⚖️ Disclaimer & Etika Keamanan

> [!IMPORTANT]
> Buku panduan ini disusun semata-mata untuk **tujuan edukasi, pembelajaran, dan peningkatan pertahanan keamanan sistem (defensive security)**.
> 
> Seluruh contoh kode yang disertakan dirancang untuk dijalankan di lingkungan lokal mandiri (*localhost*). Dilarang keras menggunakan informasi, skenario, atau teknik yang dijelaskan dalam handbook ini untuk melakukan pengujian, eksploitasi, atau serangan terhadap sistem milik pihak lain tanpa izin resmi tertulis.
