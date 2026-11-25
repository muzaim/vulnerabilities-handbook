# 05. Data & Cryptography 🔐

Kategori **Data & Cryptography** membahas perlindungan informasi rahasia (*data protection*), manajemen kunci rahasia (*secrets management*), serta penerapan algoritma kriptografi yang benar untuk menjamin kerahasiaan (*confidentiality*) dan integritas (*integrity*) data.

---

## 💡 Konsep Dasar

Data adalah aset paling berharga dalam aplikasi digital. Kriptografi adalah seni menyamarkan data sehingga hanya pihak yang berhak yang dapat membacanya.

Kelemahan dalam kategori ini terjadi ketika:
1. Data rahasia dikirimkan atau disimpan tanpa enkripsi (**Sensitive Data Exposure**).
2. Kunci rahasia, token API, atau password database ditinggalkan langsung di dalam kode program (**Hardcoded Secrets**).
3. Kata sandi disimpan tanpa hash atau menggunakan algoritma hash usang yang mudah dibongkar (**Insecure Password Storage**).
4. Penggunaan algoritma enkripsi kuno, mode operasi yang cacat, atau kunci yang terlalu pendek (**Weak Encryption**).

---

## 📑 Daftar Topik

1. **[01. Sensitive Data Exposure](01-sensitive-data-exposure.md)**  
   Paparan data sensitif (PII, nomor kartu kredit, rekam medis) akibat transmisi data tanpa enkripsi (HTTP/TLS lemah) atau penyimpanan terbuka.
2. **[02. Hardcoded Secrets](02-hardcoded-secrets.md)**  
   Penyimpanan kunci privat, token API, atau kredensial database langsung di dalam kode sumber aplikasi atau repositori Git.
3. **[03. Insecure Password Storage](03-insecure-password-storage.md)**  
   Penyimpanan kata sandi dalam bentuk teks polos (*plaintext*) atau menggunakan fungsi hash cepat yang tidak aman (seperti MD5 dan SHA-1 tanpa salt).
4. **[04. Weak Encryption](04-weak-encryption.md)**  
   Penggunaan algoritma kriptografi yang sudah usang (DES, RC4), mode operasi yang tidak aman (ECB), atau penggunaan generator acak yang lemah.
