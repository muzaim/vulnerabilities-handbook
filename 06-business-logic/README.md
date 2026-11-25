# 06. Business Logic Vulnerabilities ⚙️

Kategori **Business Logic** membahas kelemahan dalam rancangan dan alur logika proses bisnis aplikasi. Berbeda dengan bug teknis biasa (seperti SQL Injection atau XSS), kerentanan logika bisnis terjadi ketika kode bekerja persis seperti yang diprogramkan oleh developer, namun alur prosesnya memiliki celah yang dapat dimanipulasi untuk menghasilkan keuntungan ilegal.

---

## 💡 Konsep Dasar

Pemindai kerentanan otomatis (seperti scanner SAST/DAST) sering kali tidak dapat mendeteksi celah logika bisnis karena sintaks kode sepenuhnya valid dan aman dari injeksi. Celah ini hanya dapat dipahami dengan mempelajari **aturan main bisnis (*business rules*)** dari aplikasi tersebut.

Celah dalam kategori ini muncul ketika:
1. Dua proses dijalankan secara bersamaan dalam hitungan milidetik sehingga memicu saldo ganda atau klaim kupon berulang (**Race Condition & TOCTOU**).
2. Permintaan transaksi lama yang sah dikirimkan kembali untuk mendapatkan barang berulang tanpa bayar (**Replay Attack**).
3. Parameter harga barang, diskon, atau mata uang dapat diedit di tengah jalan sebelum dibayar (**Price Manipulation & Payment Bypass**).

---

## 📑 Daftar Topik

1. **[01. Race Condition & TOCTOU](01-race-condition-toctou.md)**  
   Manipulasi eksekusi konkuren antara waktu pengecekan kondisi (*Time-of-Check*) dan waktu penggunaan data (*Time-of-Use*), memungkinkan penarikan saldo ganda atau klaim kupon berlebih.
2. **[02. Replay Attack](02-replay-attack.md)**  
   Penangkapan paket data transaksi sah dan pengirimannya kembali secara berulang untuk menduplikasi aksi atau otorisasi.
3. **[03. Price Manipulation & Payment Bypass](03-price-manipulation-payment-bypass.md)**  
   Manipulasi parameter harga, kuantitas negatif, atau pemalsuan status callback payment gateway yang memungkinkan pembeli checkout tanpa membayar harga sebenarnya.
