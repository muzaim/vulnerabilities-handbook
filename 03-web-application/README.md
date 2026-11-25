# 03. Web Application Vulnerabilities 🌐

Kategori **Web Application** membahas kelompok kerentanan yang berkaitan dengan interaksi antara browser dan server, pengelolaan permintaan berkas, pengalihan URL, dan pertukaran sumber daya lintas domain (*cross-origin*).

---

## 💡 Konsep Dasar

Aplikasi web modern menghubungkan banyak komponen: peramban pengguna (*browser*), server aplikasi, server penyimpanan berkas, dan server internal perusahaan. 

Celah dalam kategori ini muncul ketika:
1. Browser pengguna dapat dimanipulasi oleh situs asing untuk mengirimkan aksi yang tidak diinginkan (**CSRF** & **Clickjacking**).
2. Server web dapat ditipu untuk mengambil data dari jaringan internal rahasia (**SSRF**).
3. Pengelolaan berkas dan rute URL tidak dibatasi dengan ketat (**Path Traversal**, **Unrestricted File Upload**, **Open Redirect**).
4. Aturan pembagian sumber daya lintas domain disetel terlalu longgar (**CORS Misconfiguration**).

---

## 📑 Daftar Topik

1. **[01. Cross-Site Request Forgery (CSRF)](01-csrf.md)**  
   Memaksa peramban korban yang sedang terotentikasi untuk mengirimkan instruksi berbahaya tanpa sepengetahuannya.
2. **[02. Server-Side Request Forgery (SSRF)](02-ssrf.md)**  
   Memanipulasi server backend agar mengirimkan permintaan jaringan ke sistem internal perusahaan atau layanan metadata cloud.
3. **[03. Path Traversal](03-path-traversal.md)**  
   Manipulasi karakter jalur direktori (`../`) untuk mengakses berkas rahasia di luar folder web root server.
4. **[04. Unrestricted File Upload](04-unrestricted-file-upload.md)**  
   Mengunggah berkas skrip berbahaya (seperti webshell `.php`, `.jsp`, `.exe`) yang kemudian dieksekusi oleh server.
5. **[05. Open Redirect](05-open-redirect.md)**  
   Pengalihan URL pengguna ke situs penipuan (*phishing*) eksternal menggunakan parameter pengalihan yang tidak divalidasi.
6. **[06. Clickjacking](06-clickjacking.md)**  
   Menjebak pengguna untuk mengklik tombol tak terlihat yang dilapisi bingkai tersembunyi (*transparent iframe*).
7. **[07. CORS Misconfiguration](07-cors-misconfiguration.md)**  
   Konfigurasi *Cross-Origin Resource Sharing* yang terlalu longgar, memungkinkan situs pihak ketiga membaca respons API pribadi.
8. **[08. Insecure Deserialization](08-insecure-deserialization.md)**  
   Mengekstrak dan mengonversi data serial yang tidak terpercaya kembali menjadi objek runtime, memicu eksekusi kode berbahaya.
