# Race Condition & TOCTOU (Time-of-Check to Time-of-Use)

> **Metadata Keamanan:**
> - **CWE**: [CWE-362: Concurrent Execution using Shared Resource with Improper Synchronization ('Race Condition')](https://cwe.mitre.org/data/definitions/362.html) & [CWE-367: Time-of-check Time-of-use (TOCTOU) Race Condition](https://cwe.mitre.org/data/definitions/367.html)
> - **OWASP**: [A04:2021 – Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/)
> - **Tingkat Keparahan (Severity)**: High / Critical (CVSS ~ 7.5 - 8.8)

---

## 1. Pengertian

**Race Condition** (Kondisi Balapan) adalah situasi di mana perilaku sistem bergantung pada urutan atau waktu eksekusi dari beberapa proses yang berjalan secara bersamaan (*konkuren*).

**TOCTOU (Time-of-Check to Time-of-Use)** adalah bentuk spesifik dari race condition di mana terdapat celah waktu (*window of vulnerability*) antara saat sistem **memeriksa** suatu kondisi (*check*) dan saat sistem **menggunakan** hasil pemeriksaan tersebut (*use*). Jika keadaan berubah di antara kedua titik waktu tersebut, keputusan yang diambil sistem menjadi salah dan berbahaya.

**Analogi Sederhana:**  
Bayangkan sebuah mesin ATM yang melayani Anda. Saldo rekening Anda adalah Rp 100.000.  
Anda menekan tombol tarik tunai Rp 100.000 di dua mesin ATM yang berbeda pada **detik dan milidetik yang persis sama**.  
- **Mesin A**: Memeriksa saldo Anda -> Saldo cukup (Rp 100.000).  
- **Mesin B**: Memeriksa saldo Anda -> Saldo masih Rp 100.000 (karena Mesin A belum selesai mencairkan uang).  
- **Mesin A**: Mengeluarkan uang Rp 100.000 dan memotong saldo menjadi Rp 0.  
- **Mesin B**: Mengeluarkan uang Rp 100.000 lagi dan memotong saldo menjadi Rp 0.  
Anda berhasil mendapatkan uang Rp 200.000 padahal saldo Anda hanya Rp 100.000!

---

## 2. Cara Kerja

Pada aplikasi web multi-threaded / asynchronous (seperti Go dengan Goroutines atau Node.js event loop dengan panggilan I/O database):

```
WAKTU     THREAD 1 (Permintaan Penarikan 1)         THREAD 2 (Permintaan Penarikan 2)
  │
 t1       BACA SALDO DARI DB (Saldo: 100.000)
  │                                                  BACA SALDO DARI DB (Saldo: 100.000)
 t2       VALIDASI: Apakah 100.000 >= 100.000?       VALIDASI: Apakah 100.000 >= 100.000?
  │       [ VALIDASI LOLOS ]                         [ VALIDASI LOLOS ]
  │
 t3       TRANSFER UANG KE REKENING TUJUAN           TRANSFER UANG KE REKENING TUJUAN
  │
 t4       UPDATE DB: Saldo = Saldo - 100.000         UPDATE DB: Saldo = Saldo - 100.000
          (Saldo sekarang: 0)                        (Saldo sekarang: -100.000 atau tetap 0)
```

Karena *Thread 2* membaca database sebelum *Thread 1* sempat memperbarui saldo, kedua thread sama-sama melihat saldo mencukupi dan meloloskan aksi ganda (*double spending*).

---

## 3. Penyebab

- **Pemisahan Logika 'Check' dan 'Update' Tanpa Transaksi Terisolasi**: Melakukan `SELECT` data terlebih dahulu di kode aplikasi, mengecek kondisi dengan `if (balance >= amount)`, lalu baru menjalankan `UPDATE`.
- **Ketiadaan Penguncian Database (*Database Locking*)**: Tidak menggunakan *Pessimistic Locking* (`SELECT ... FOR UPDATE`) atau *Optimistic Locking* (versioning).
- **Operasi Non-Atomik**: Tidak menggunakan kueri pembaruan langsung yang atomik seperti `UPDATE accounts SET balance = balance - 100 WHERE balance >= 100`.
- **Ketiadaan Distributed Lock pada Microservices**: Banyak instance microservice membaca cache Redis tanpa sinkronisasi kunci terdistribusi (seperti Redlock).

---

## 4. Skenario Serangan

1. **Identifikasi Fitur Terbatas**: Penyerang melihat promo e-commerce: *"Kupon diskon 50% hanya berlaku 1 kali per pengguna"*.
2. **Pembuatan Skrip Paralel**: Penyerang menulis skrip Python atau menggunakan fitur *Turbo Intruder* di Burp Suite untuk mengirim 50 permintaan klaim kupon pada satu milidetik yang sama menggunakan HTTP/2 multiplexing (*single-packet attack*).
3. **Eksekusi Konkuren**: Ke-50 permintaan tiba di backend pada saat yang hampir bersamaan. Server memverifikasi bahwa kupon belum pernah digunakan untuk ke-50 proses tersebut.
4. **Klaim Berulang**: Ke-50 proses berhasil menambahkan diskon, sehingga total belanjaan penyerang menjadi gratis atau saldo dompet penyerang bertambah puluhan kali lipat.

---

## 5. Dampak

- **Kerugian Finansial Langsung (Double Spending)**: Penarikan saldo ganda, pencairan dana berulang kali.
- **Manipulasi Stok Barang (Over-selling)**: Membeli barang flash sale melebihi batas stok fisik yang tersedia.
- **Eksploitasi Kupon & Reward**: Klaim kupon promo sekali pakai berkali-kali.

---

## 6. Bagian Sistem yang Terdampak

- Endpoint transfer saldo dompet digital (*wallet transfer*).
- Endpoint checkout barang promosi atau *flash sale*.
- Fitur penukaran kupon diskon atau referral reward.
- Alur voting / polling sekali pilih.

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Dompet Digital PayKilat:**  
PayKilat memiliki fitur transfer saldo antar-pengguna. Kode backend Go menangani transfer dengan alur:
1. `user = db.FindUser(senderID)`
2. `if user.Balance < amount { return "Saldo tidak cukup" }`
3. `db.SendMoney(receiverID, amount)`
4. `db.DeductBalance(senderID, amount)`

Seorang pengguna memiliki sisa saldo Rp 50.000 di akunnya. Pengguna tersebut mengirim 10 permintaan transfer sebesar Rp 50.000 secara paralel ke akun temannya. Karena server Go menangani setiap permintaan dalam goroutine yang berbeda secara konkuren dan tidak mengunci baris data di database, sebanyak 7 dari 10 request berhasil diproses sebelum saldo terpotong menjadi 0. Saldo teman penyerang bertambah Rp 350.000 dari saldo asli yang hanya Rp 50.000. PayKilat mengalami kerugian Rp 300.000 dalam sekejap mata.

---

## 8. Cara Mendeteksi

- **Concurrency Testing (Turbo Intruder / k6 / Artillery)**: Kirimkan 20-50 request identik pada milidetik yang sama menggunakan single-packet attack (HTTP/2 connection).
- **Code Review**: Cari pola logika di mana pembacaan data (`SELECT`) dan pembaruan data (`UPDATE`) dipisahkan oleh logika percabangan `if` tanpa blok transaksi terkunci (`FOR UPDATE`).

---

## 9. Pencegahan

1. **Gunakan Atomic Database Operations (Solusi Termudah & Tercepat)**:  
   Lakukan pengecekan dan pengurangan dalam SATU perintah kueri SQL atomik:
   ```sql
   UPDATE accounts 
   SET balance = balance - 100000 
   WHERE id = $user_id AND balance >= 100000;
   ```
   Jika baris yang terpengaruh (*rows affected*) bernilai 1, aksi berhasil. Jika 0, saldo tidak cukup. Database menjamin kueri ini dieksekusi secara serial dan kebal terhadap balapan thread!
2. **Gunakan Pessimistic Locking (`SELECT FOR UPDATE`)**:  
   Kunci baris data di database selama transaksi berlangsung sehingga thread lain wajib mengantre sampai transaksi selesai:
   ```sql
   BEGIN;
   SELECT balance FROM accounts WHERE id = $user_id FOR UPDATE;
   -- Lakukan validasi dan transfer...
   UPDATE accounts SET balance = balance - 100000 WHERE id = $user_id;
   COMMIT;
   ```
3. **Gunakan Distributed Lock (Redis Redlock)**:  
   Pada arsitektur microservices terdistribusi, pasang lock berbasis kunci pengguna (misal `lock:user:123`) di Redis sebelum memproses transaksi.
4. **Terapkan Uniqueness Constraints di Database**:  
   Untuk klaim kupon satu kali, buat tabel `coupon_claims` dengan constraint gabungan unik `UNIQUE(user_id, coupon_id)`. Percobaan klaim kedua otomatis ditolak oleh database dengan error *unique constraint violation*.

---

## 10. Penanganan

1. **Karantina Transaksi Ganda**: Identifikasi transaksi yang terjadi dalam selisih waktu kurang dari 500ms dari akun yang sama.
2. **Koreksi Saldo Minus**: Hitung ulang saldo riil berdasarkan total mutasi masuk dan keluar.
3. **Deploy Kueri Atomik Segera**: Ganti logika pembaruan di backend dengan atomic update atau transaksi terkunci.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Kueri Atomik vs Kode Rentan)

```go
package main

import (
	"database/sql"
	"errors"
)

// ❌ KODE RENTAN (TOCTOU Race Condition)
// Membaca dulu, cek if, baru kurangi saldo
func TransferVulnerable(db *sql.DB, userID int, amount int64) error {
	var currentBalance int64
	// Time-of-Check: Baca saldo
	err := db.QueryRow("SELECT balance FROM accounts WHERE id = $1", userID).Scan(&currentBalance)
	if err != nil {
		return err
	}

	if currentBalance < amount {
		return errors.New("saldo tidak cukup")
	}

	// Jeda waktu di sini rentan dieksploitasi oleh goroutine lain!

	// Time-of-Use: Potong saldo
	_, err = db.Exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, userID)
	return err
}

// ✅ KODE AMAN: Operasi Atomik Database Tunggal (Atomic Update)
func TransferSecure(db *sql.DB, userID int, amount int64) error {
	// Kunci Keamanan: Pengecekan saldo dilakukan langsung di dalam klausa WHERE
	// Database engine menjamin konsistensi ACID secara atomik!
	result, err := db.Exec(
		"UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
		amount, userID,
	)
	if err != nil {
		return err
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return err
	}

	// Jika 0 baris terpengaruh, berarti kondisi `balance >= amount` tidak terpenuhi
	if rowsAffected == 0 {
		return errors.New("saldo tidak cukup atau akun tidak ditemukan")
	}

	return nil
}
```

### B. Contoh dalam TypeScript (Database Transaction & Pessimistic Lock Prisma/TypeORM)

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// ✅ KODE AMAN: Menggunakan Transaksi Database Interaktif dengan Kueri Kunci
export async function claimCouponSecure(userId: string, couponCode: string) {
  return await prisma.$transaction(async (tx) => {
    // 1. Kunci baris kupon menggunakan raw SQL FOR UPDATE agar thread lain wajib mengantre
    const coupons: any[] = await tx.$queryRaw`
      SELECT id, quota, is_active FROM coupons 
      WHERE code = ${couponCode} FOR UPDATE
    `;

    const coupon = coupons[0];
    if (!coupon || !coupon.is_active || coupon.quota <= 0) {
      throw new Error('Kupon sudah habis atau tidak aktif');
    }

    // 2. Kurangi kuota kupon
    await tx.coupon.update({
      where: { code: couponCode },
      data: { quota: { decrement: 1 } },
    });

    // 3. Catat riwayat klaim dengan constraint UNIQUE(userId, couponId)
    await tx.userCouponClaim.create({
      data: {
        userId,
        couponId: coupon.id,
      },
    });

    return { message: 'Kupon berhasil diklaim secara aman!' };
  });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Konkurensi Otomatis (Concurrency Test)
Gunakan skrip Node.js untuk menembak endpoint lokal secara bersamaan menggunakan `Promise.all`:
```typescript
it('harus menolak penarikan ganda yang melebihi saldo', async () => {
  // Saldo awal: 100.000. Coba tarik 100.000 sebanyak 5 kali bersamaan.
  const requests = Array(5).fill(null).map(() => 
    fetch('http://localhost:8080/api/withdraw', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId: 1, amount: 100000 })
    })
  );

  const responses = await Promise.all(requests);
  const successCount = responses.filter(r => r.status === 200).length;

  // HANYA boleh ada TEPAT 1 transaksi yang sukses! 4 lainnya harus gagal.
  expect(successCount).toBe(1);
});
```

---

## Referensi

- [PortSwigger: Race Conditions](https://portswigger.net/web-security/race-conditions)
- [OWASP Concurrency Testing](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References)
- [MITRE CWE-362: Race Condition](https://cwe.mitre.org/data/definitions/362.html)
- [MITRE CWE-367: TOCTOU](https://cwe.mitre.org/data/definitions/367.html)
