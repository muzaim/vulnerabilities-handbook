# 04. API Security 🔌

Kategori **API Security** membahas kerentanan spesifik yang muncul pada arsitektur antarmuka pemrograman aplikasi modern (RESTful API, GraphQL, gRPC) yang melayani aplikasi seluler (Android/iOS) dan Single-Page Application (SPA).

---

## 💡 Konsep Dasar

API bertindak sebagai gerbang data langsung ke mesin backend tanpa perantara halaman HTML tradisional. Karena API dirancang untuk berkomunikasi antar-mesin (*machine-to-machine*):
1. Pengembang sering lupa bahwa klien (seperti aplikasi mobile) sepenuhnya dapat dibongkar (*reverse-engineered*) oleh penyerang.
2. Filter keamanan yang hanya diletakkan di antarmuka frontend tidak memiliki kekuatan apa pun terhadap pemanggilan endpoint API langsung melalui alat seperti cURL, Postman, atau skrip otomatis.

Standar acuan utama untuk kategori ini adalah **OWASP API Security Top 10**.

---

## 📑 Daftar Topik

1. **[01. Broken Function Level Authorization (BFLA)](01-broken-function-level-authorization-bfla.md)**  
   Kegagalan membatasi akses ke endpoint fungsi administratif atau sensitif berdasarkan peran pengguna.
2. **[02. Mass Assignment](02-mass-assignment.md)**  
   Meneruskan seluruh properti JSON dari permintaan pengguna langsung ke model database tanpa penyaringan field.
3. **[03. Excessive Data Exposure](03-excessive-data-exposure.md)**  
   Mengembalikan seluruh data objek sensitif dalam respons API dan hanya mengandalkan frontend untuk menyembunyikannya dari layar.
4. **[04. Unrestricted Resource Consumption](04-unrestricted-resource-consumption.md)**  
   Ketiadaan pembatasan ukuran request, pagination, atau rate limiting pada API, memicu kehabisan memori atau CPU server.
5. **[05. Improper Error Handling](05-improper-error-handling.md)**  
   Mengekspos stack trace, query database, atau informasi lingkungan server internal dalam pesan galat API.
