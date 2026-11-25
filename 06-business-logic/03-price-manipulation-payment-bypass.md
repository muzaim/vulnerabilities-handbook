# Price Manipulation & Payment Bypass (Manipulasi Harga & Bypass Pembayaran)

> **Metadata Keamanan:**
> - **CWE**: [CWE-472: External Control of Assumed-Immutable Web Parameter](https://cwe.mitre.org/data/definitions/472.html) & [CWE-840: Business Logic Errors](https://cwe.mitre.org/data/definitions/840.html)
> - **OWASP**: [A04:2021 – Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/)
> - **Tingkat Keparahan (Severity)**: Critical (CVSS ~ 8.5 - 9.8)

---

## 1. Pengertian

**Price Manipulation** (Manipulasi Harga) dan **Payment Bypass** (Pembobolan Pembayaran) adalah kelompok celah logika bisnis di mana pembeli dapat mengubah harga barang, nilai diskon, kuantitas produk, atau memalsukan notifikasi status pelunasan dari payment gateway, sehingga pembeli dapat memperoleh barang atau layanan mahal dengan membayar harga yang sangat murah (misalnya Rp 1) atau bahkan tanpa membayar sama sekali.

**Analogi Sederhana:**  
- **Price Manipulation**: Anda pergi ke supermarket, mengambil sebotol parfum mahal seharga Rp 1.500.000. Sebelum menuju kasir, Anda melepas stiker barcode parfum tersebut dan menempelkan stiker barcode mie instan seharga Rp 3.000. Kasir langsung men-scan stiker mie instan tanpa mencocokkan nama barang dan membiarkan Anda pulang dengan membayar Rp 3.000.
- **Payment Bypass**: Anda memesan makanan di kasir. Kasir berkata: *"Silakan bayar di kasir sebelah dan bawakan bukti struk stempel LUNAS"*. Anda mengambil spidol merah di tas Anda, menulis kata "LUNAS" di secarik kertas tisu, lalu menyerahkannya ke kasir. Kasir langsung memberikan seluruh makanan kepada Anda tanpa mengecek komputer apakah uangnya benar-benar sudah masuk ke rekening toko.

---

## 2. Cara Kerja

### A. Manipulasi Harga Langsung (Client-Side Price Parameter)
Celah ini terjadi saat frontend web atau mobile mengirimkan parameter harga langsung ke backend:
```json
// Request yang dikirim frontend ke endpoint checkout:
{
  "product_id": 105,
  "quantity": 1,
  "price": 5000000 // Penyerang mengubah nilai ini di Burp Suite menjadi: 100
}
```
Jika backend hanya mengalikan `req.body.price * req.body.quantity` dan langsung membuat tagihan pembayaran, penyerang berhasil membeli barang seharga jutaan rupiah dengan membayar seratus perak!

### B. Kuantitas Negatif (Negative Quantity Flaw)
Sebagian developer memeriksa harga dari database, namun lupa membatasi kuantitas:
- Produk A: Laptop seharga Rp 10.000.000 (Jumlah: 1) = Rp 10.000.000
- Produk B: Aksesoris seharga Rp 100.000 (Jumlah: -99) = -Rp 9.900.000  
Total Belanja: Rp 100.000. Pembeli mendapatkan laptop seharga 10 juta hanya dengan membayar Rp 100.000.

### C. Bypass Notifikasi Pembayaran (Webhook Callback Forgery)
Payment Gateway (seperti Midtrans, Xendit, Stripe) memberitahu server toko ketika pembayaran berhasil melalui Webhook:
`POST /api/webhooks/payment-success`
Jika backend tidak memverifikasi **Tanda Tangan Digital Kriptografis (Signature)** dari payment gateway, penyerang dapat menembakkan request webhook palsu secara manual untuk mengubah status pesanan menjadi "LUNAS / SIAP KIRIM".

---

## 3. Penyebab

- **Mempercayai Harga dari Klien**: Menerima field `price`, `subtotal`, atau `total_amount` dari request pengguna alih-alih mengambil harga resmi dari database server.
- **Ketiadaan Validasi Batas Nilai (*Boundary Check*)**: Mengizinkan nilai kuantitas negatif (`quantity < 1`) atau nilai desimal tidak wajar.
- **Ketiadaan Verifikasi Webhook Signature**: Tidak mencocokkan signature rahasia yang dikirimkan oleh payment gateway resmi.
- **Tidak Memverifikasi Jumlah Nominal Saat Pelunasan**: Hanya memeriksa status `"PAID"`, tetapi tidak mencocokkan apakah nominal uang yang masuk ke payment gateway sama persis dengan total tagihan di pesanan.

---

## 4. Skenario Serangan

1. **Pemilihan Barang**: Penyerang memasukkan smartphone seharga Rp 15.000.000 ke keranjang belanja.
2. **Intersepsi Request Checkout**: Penyerang menyalakan proxy Burp Suite dan mengklik "Bayar".
3. **Modifikasi Nilai JSON**: Penyerang melihat request:  
   `{"items": [{"id": 44, "name": "Smartphone", "price": 15000000, "qty": 1}]}`.  
   Penyerang mengubah `price` menjadi `1` atau menyisipkan item `{"id": 99, "qty": -1}`.
4. **Pembayaran Berhasil**: Payment gateway menerima tagihan sebesar Rp 1. Penyerang membayar Rp 1 via e-wallet.
5. **Pelepasan Barang**: Sistem e-commerce menandai pesanan lunas dan gudang mengirimkan smartphone tersebut ke rumah penyerang.

---

## 5. Dampak

- **Kerugian Finansial Langsung**: Barang fisik bernilai mahal dikirimkan ke pelanggan dengan harga nyaris nol.
- **Pencucian Uang (Money Laundering)**: Penyerang membeli barang manipulasi lalu menjualnya kembali di marketplace lain untuk mendapatkan uang tunai bersih.
- **Gangguan Pembukuan Keuangan**: Ketidaksesuaian antara stok barang yang keluar dan pencatatan kas perbankan.

---

## 6. Bagian Sistem yang Terdampak

- Alur keranjang belanja (*cart*) dan kalkulasi checkout.
- Handler webhook / IPN (*Instant Payment Notification*) dari payment gateway.
- Modul voucher diskon dan konversi mata uang (*multi-currency*).

---

## 7. Contoh Kasus Nyata dalam Sebuah Sistem

> *Catatan: Kasus berikut adalah simulasi edukatif yang realistis untuk mempermudah pemahaman.*

**Sistem Pemesanan Tiket Pesawat TerbangCepat:**  
TerbangCepat memiliki integrasi pembayaran dengan payment gateway pihak ketiga.

Ketika transaksi pembayaran kartu kredit selesai, payment gateway memanggil webhook backend:
`POST /api/payments/callback` dengan body `{"order_id": "TC-8812", "status": "SETTLEMENT", "gross_amount": 5000000}`.

Developer TerbangCepat mempercayai siapapun yang memanggil endpoint tersebut tanpa memverifikasi signature `X-Callback-Signature`.

Seorang penyerang memesan tiket pesawat internasional seharga Rp 25.000.000. Saat masuk ke halaman pembayaran, penyerang tidak membayar. Sebagai gantinya, penyerang menggunakan cURL untuk menembakkan request POST palsu langsung ke endpoint webhook TerbangCepat. Sistem TerbangCepat langsung menerbitkan tiket pesawat elektronik resmi berkode booking PNR yang valid, dan penyerang terbang ke luar negeri tanpa membayar sepeser pun.

---

## 8. Cara Mendeteksi

- **Pemeriksaan Parameter Request**: Periksa apakah ada parameter harga yang dikirim dari browser ke server saat checkout.
- **Uji Kuantitas Negatif & Nol**: Coba kirim `quantity: 0`, `quantity: -1`, `quantity: 0.001`.
- **Uji Coba Webhook Tanpa Signature**: Tembakkan request palsu ke endpoint callback webhook tanpa header signature. Jika server mengembalikan 200 OK dan mengubah status pesanan menjadi lunas, sistem rentan.

---

## 9. Pencegahan

1. **Ambil Harga HANYA dari Database Server (Wajib Mutlak)**:  
   Klien HANYA boleh mengirimkan `product_id` dan `quantity`. Server yang wajib mencari harga asli di database dan menghitung total tagihan di sisi backend:
   ```text
   Klien mengirim: { product_id: 101, quantity: 2 }
   Server menghitung: total = db.getProductPrice(101) * 2;
   ```
2. **Validasi Kuantitas Secara Tegas**:  
   Wajibkan kuantitas berupa integer bulat positif minimal 1 (`qty >= 1`). Tolak nilai negatif atau desimal.
3. **Verifikasi Webhook Signature Kriptografis (HMAC)**:  
   Setiap payment gateway menyediakan secret webhook key. Server toko wajib menghitung hash HMAC dari payload yang diterima dan menolak proses jika signature tidak cocok.
4. **Validasi Ganda Nominal (*Double Check Amount*)**:  
   Saat notifikasi "LUNAS" diterima, periksa apakah nominal uang yang dilaporkan oleh gateway sama persis dengan nominal pesanan di database toko:
   `if (callback.gross_amount !== order.total_amount) { return FRAUD_ALERT; }`.

---

## 10. Penanganan

1. **Tahan Pengiriman Barang (Hold Shipment)**: Tahan seluruh barang yang memiliki harga bayar tidak sesuai dengan harga katalog.
2. **Pasang Verifikasi Signature Webhook Segera**: Tolak seluruh panggilan callback yang tidak memiliki header signature resmi.
3. **Audit Transaksi Masa Lalu**: Jalankan kueri database untuk mencari transaksi di mana `amount_paid < product_price`.

---

## 11. Contoh Implementasi

### A. Contoh dalam Golang (Kalkulasi Harga di Server & Verifikasi HMAC Webhook)

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"net/http"
)

type CartItemInput struct {
	ProductID int `json:"product_id"`
	Quantity  int `json:"quantity"` // Hanya terima ID dan Kuantitas!
}

// ✅ KODE AMAN 1: Menghitung total harga HANYA dari data server
func CalculateTotalSecure(items []CartItemInput, getProductPriceFromDB func(int) (int64, error)) (int64, error) {
	var totalOrderPrice int64 = 0

	for _, item := range items {
		// Validasi bahwa kuantitas wajib angka bulat positif minimal 1
		if item.Quantity <= 0 {
			return 0, errors.New("kuantitas barang tidak valid (harus minimal 1)")
		}

		price, err := getProductPriceFromDB(item.ProductID)
		if err != nil {
			return 0, errors.New("produk tidak ditemukan")
		}

		totalOrderPrice += price * int64(item.Quantity)
	}

	return totalOrderPrice, nil
}

// ✅ KODE AMAN 2: Memverifikasi Tanda Tangan Webhook Payment Gateway
func VerifyWebhookSignature(rawPayload []byte, signatureHeader string, webhookSecret string) bool {
	mac := hmac.New(sha256.New, []byte(webhookSecret))
	mac.Write(rawPayload)
	expectedSignature := hex.EncodeToString(mac.Sum(nil))

	// Gunakan hmac.Equal untuk perbandingan aman terhadap timing attack
	return hmac.Equal([]byte(expectedSignature), []byte(signatureHeader))
}

func PaymentWebhookHandler(w http.ResponseWriter, r *http.Request) {
	// Jika signature tidak sah, tolak seketika!
	signature := r.Header.Get("X-Payment-Signature")
	if !VerifyWebhookSignature([]byte("payload"), signature, "SECRET_KEY_DARI_GATEWAY") {
		http.Error(w, "Tanda tangan webhook palsu / tidak sah!", http.StatusUnauthorized)
		return
	}
	// Lanjutkan proses status pesanan...
}
```

### B. Contoh dalam TypeScript (Express & Validasi Zod Kuantitas + Server Price Lookup)

```typescript
import { Request, Response } from 'express';
import { z } from 'zod';
import crypto from 'crypto';

// Skema input: Klien HANYA boleh mengirim id produk dan kuantitas positif
const CheckoutSchema = z.object({
  items: z.array(z.object({
    productId: z.number().int().positive(),
    quantity: z.number().int().min(1).max(100), // Wajib minimal 1, tolak negatif!
  })).min(1),
});

// ✅ KODE AMAN: Backend menghitung harga murni dari database
export async function createCheckoutSession(req: Request, res: Response) {
  const parseResult = CheckoutSchema.safeParse(req.body);
  if (!parseResult.success) {
    return res.status(400).json({ error: 'Kuantitas atau format item tidak valid' });
  }

  const { items } = parseResult.data;
  let calculatedGrandTotal = 0;

  for (const item of items) {
    // Ambil harga resmi dari database server, ABAIKAN harga apapun dari klien
    const product = await prisma.product.findUnique({ where: { id: item.productId } });
    if (!product) {
      return res.status(404).json({ error: `Produk ID ${item.productId} tidak ditemukan` });
    }

    calculatedGrandTotal += product.price * item.quantity;
  }

  // Daftarkan tagihan ke payment gateway dengan nominal resmi server
  const order = await prisma.order.create({
    data: {
      userId: req.user!.id,
      totalAmount: calculatedGrandTotal,
      status: 'PENDING_PAYMENT',
    },
  });

  return res.json({ orderId: order.id, totalToPay: calculatedGrandTotal });
}
```

---

## 12. Pengujian dan Verifikasi

### Pengujian Manipulasi Kuantitas Negatif
Kirimkan request checkout dengan kuantitas bernilai negatif atau nol:
```bash
curl -X POST http://localhost:8080/api/checkout \
  -H "Content-Type: application/json" \
  -d '{"items": [{"productId": 1, "quantity": -5}]}'
```
- **Kondisi Rentan**: Sistem menerima request dan menghasilkan nilai tagihan minus atau diskon tidak sah.
- **Kondisi Aman**: Sistem menolak dengan status `400 Bad Request` ("Kuantitas barang tidak valid").

---

## Referensi

- [PortSwigger: Flaws in Business Logic](https://portswigger.net/web-security/logic-flaws)
- [OWASP Business Logic Testing](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/README)
- [MITRE CWE-472](https://cwe.mitre.org/data/definitions/472.html)
