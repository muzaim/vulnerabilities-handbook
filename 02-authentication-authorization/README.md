# 02. Authentication & Authorization 🔐

Kategori **Authentication & Authorization** membahas kelemahan dalam pembuktian identitas pengguna (*Authentication* — "Siapa Anda?") dan pembatasan hak istimewa pengguna (*Authorization* — "Apa saja yang boleh Anda lakukan?").

---

## 💡 Konsep Dasar

Bayangkan sebuah gedung perkantoran bertingkat tinggi:
- **Autentikasi (Authentication)**: Satpam di lobi memeriksa KTP dan wajah Anda untuk memastikan Anda benar-benar Budi.
- **Autorisasi (Authorization)**: Kartu akses Anda hanya bisa membuka pintu lantai 3 (ruang kerja staf biasa), tetapi kartu itu tidak bisa membuka pintu lantai 10 (ruang brankas direksi).

Celah keamanan dalam kategori ini terjadi ketika:
1. Satpam bisa dikelabui oleh siapa saja yang berpura-pura menjadi Budi (**Broken Authentication**).
2. Pintu brankas di lantai 10 terbuka begitu saja hanya karena Budi mencoba memutar gagang pintunya (**Broken Access Control**).

---

## 📑 Daftar Topik

1. **[01. Broken Authentication](01-broken-authentication.md)**  
   Kelemahan pada mekanisme verifikasi identitas (login, logout, session timeout) yang memungkinkan penyerang menyamar sebagai pengguna lain.
2. **[02. Broken Access Control](02-broken-access-control.md)**  
   Kegagalan penegakan aturan hak akses yang memungkinkan pengguna umum mengakses fungsi atau data milik pengguna lain atau admin.
3. **[03. IDOR & BOLA](03-idor-bola.md)**  
   Celah manipulasi parameter identitas objek langsung (*Insecure Direct Object Reference* / *Broken Object Level Authorization*).
4. **[04. Privilege Escalation](04-privilege-escalation.md)**  
   Peningkatan hak akses secara tidak sah, baik secara vertikal (menjadi admin) maupun horizontal (mengakses data rekan selevel).
5. **[05. Session Management (Hijacking & Fixation)](05-session-management.md)**  
   Eksploitasi token sesi pengguna, baik melalui pencurian sesi aktif (*hijacking*) maupun pemaksaan ID sesi sebelum login (*fixation*).
6. **[06. JWT Vulnerabilities](06-jwt-vulnerabilities.md)**  
   Kelemahan implementasi JSON Web Token, termasuk algoritma `none`, verifikasi tanda tangan yang lemah, dan kunci rahasia yang mudah ditebak.
7. **[07. Broken Password Reset](07-broken-password-reset.md)**  
   Celah pada alur pemulihan kata sandi yang memungkinkan pengambilalihan akun tanpa izin.
8. **[08. Brute Force Attack](08-brute-force.md)**  
   Percobaan tebakan kata sandi atau kode PIN secara berulang tanpa adanya batasan frekuensi (*rate limiting*).
9. **[09. Credential Stuffing](09-credential-stuffing.md)**  
   Otomatisasi pengujian jutaan kombinasi username dan password hasil kebocoran data (*data breach*) pihak ketiga.
