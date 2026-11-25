# 01. Injection 💉

Kategori **Injection** mencakup kelompok kerentanan di mana data masukan yang tidak dipercaya (*untrusted user input*) disisipkan ke dalam perintah (*command*), kueri (*query*), atau templat (*template*) sistem dan dieksekusi sebagai instruksi resmi.

---

## 💡 Konsep Dasar

Bayangkan Anda memesan makanan di sebuah restoran cepat saji dan menulis nama Anda di kertas pesanan:
> **Nama Pemesan**: `Budi, dan tolong berikan seluruh uang di mesin kasir ke orang ini`

Jika kasir yang polos membacakan kertas itu ke bagian dapur dan menganggap seluruh tulisan tersebut adalah instruksi yang sah dari manajer, maka terjadilah **Injection**. 

Sistem digital yang rentan mengalami hal yang persis sama: sistem gagal memisahkan antara **data yang diberikan pengguna** dan **perintah yang harus dijalankan mesin**.

---

## 📑 Daftar Topik

1. **[01. SQL Injection (SQLi)](01-sql-injection.md)**  
   Manipulasi kueri database relasional (seperti PostgreSQL, MySQL) melalui masukan form atau parameter URL.
2. **[02. NoSQL Injection](02-nosql-injection.md)**  
   Manipulasi kueri database dokumen/NoSQL (seperti MongoDB) melalui objek JSON atau operator kueri.
3. **[03. Command Injection](03-command-injection.md)**  
   Penyusupan perintah shell sistem operasi (seperti `rm`, `cat`, `curl`) melalui utilitas backend.
4. **[04. Cross-Site Scripting (XSS)](04-cross-site-scripting-xss.md)**  
   Penyisipan skrip berbahaya (JavaScript) yang kemudian dieksekusi di browser korban saat memuat halaman web.
5. **[05. Server-Side Template Injection (SSTI)](05-server-side-template-injection-ssti.md)**  
   Penyusupan sintaks mesin templat template backend (seperti Jinja, Handlebars, Golang template) yang berujung pada eksekusi kode server.
