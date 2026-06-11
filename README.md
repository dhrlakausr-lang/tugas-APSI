| Keterangan | Isi |
| --- | --- |
| Nama | Nama Mahasiswa |
| NIM | 000000000 |
| Kelas | Nama Kelas |
| Mata Kuliah | Nama Mata Kuliah |
| Dosen Pengampu | Nama Dosen |
| Judul Tugas | Judul Sistem / Studi Kasus |

# E-Commerce System (Tokopedia Core Transaction Module)

> **Tugas Analisis dan Perancangan Sistem Informasi (APSI) - Pertemuan 9**
> 
> Topik: Perancangan Sistem Informasi Penjualan Online untuk Mendukung Proses Transaksi, Pengelolaan Produk, dan Pengiriman Barang (Studi Kasus: Modul Checkout & Pembayaran).

---

## 1. Business Requirements Document (BRD)

### A. Latar Belakang
Platform membutuhkan sistem checkout terpusat untuk mengamankan transaksi antara pembeli dan penjual. Proses transaksi multi-penjual dalam satu keranjang menuntut akurasi perhitungan biaya secara dinamis (termasuk ongkos kirim), pengamanan dana via escrow (rekening bersama), pemotongan stok real-time, serta otomatisasi verifikasi pembayaran melalui integrasi payment gateway.

### B. Tujuan Bisnis
* Mempermudah pembeli melakukan simulasi biaya, pemilihan kurir, dan penyelesaian pembayaran secara instan.
* Mengotomatisasi verifikasi pembayaran tanpa intervensi manual menggunakan sistem callback webhook.
* Mengurangi risiko overselling dengan menerapkan sistem penguncian stok sementara (soft-book) saat checkout.
* Menyediakan transparansi status pesanan bagi pembeli maupun penjual sejak dana masuk hingga barang diproses.

### C. Ruang Lingkup Sistem
Sistem mencakup manajemen keranjang belanja, kalkulasi biaya produk, integrasi API logistik untuk cek ongkir, integrasi API payment gateway untuk penerbitan Virtual Account (VA), pembaruan status pesanan berbasis callback, dan distribusi notifikasi ke penjual.

### D. Pemangku Kepentingan (Stakeholders)
* **Pembeli:** Melakukan checkout barang, memilih kurir, memilih metode pembayaran, dan melunasi tagihan.
* **Penjual:** Menerima notifikasi pesanan lunas dan memproses pengiriman fisik.
* **Pihak Ketiga (Third-Party API):** Payment Gateway (memproses transaksi uang) dan API Logistik (menyediakan data tarif ongkir).
* **Admin Platform:** Memantau kelancaran transaksi dan mengelola dana di rekening escrow.

### E. Kebutuhan Fungsional
1. Sistem mampu mengonsolidasi produk dari penjual yang berbeda dalam satu kali checkout.
2. Sistem dapat melakukan request API ke kurir logistik untuk menghitung ongkos kirim otomatis.
3. Sistem mampu menerbitkan nomor Virtual Account (VA) unik sesuai metode pembayaran pilihan pembeli.
4. Sistem melakukan soft-book (mengunci stok) selama batas waktu pembayaran aktif.
5. Sistem menerima sinyal callback dari payment gateway untuk mengubah status pesanan menjadi "Sudah Dibayar".
6. Sistem otomatis memicu notifikasi pesanan masuk ke dashboard penjual setelah pembayaran sah.

---

## 2. Entity Relationship Diagram (ERD)

### Komponen Entitas
* **Users:** Menyimpan data pengguna platform (berperan sebagai Pembeli atau Penjual lewat pemisahan role).
* **Produk:** Menyimpan informasi barang yang dijual (nama, harga, stok, dan terikat ke ID_Penjual).
* **Pesanan / Invoice:** Mencatat transaksi utama (nomor invoice, tanggal, total harga, ongkir, status pesanan, dan terikat ke ID_Pembeli).
* **Detail_Pesanan:** Tabel penghubung antara Pesanan dan Produk untuk merinci jumlah barang, harga satuan, dan subtotal item.
* **Pembayaran:** Menyimpan detail pembayaran (nomor VA, metode pembayaran, jumlah dana, status kelunasan, dan terikat ke ID_Pesanan).

> **Catatan Pertahanan Nilai (Logika Relasi):**
> Satu Kategori memiliki banyak Produk; satu Pelanggan dapat membuat banyak Pesanan; satu Pesanan terdiri dari satu atau lebih Detail_Pesanan; satu Produk bisa muncul di banyak Detail_Pesanan; dan satu Pesanan dibayar dengan tepat satu Pembayaran.

---

## 3. Data Flow Diagram (DFD)

### DFD Level 0 (Context Diagram)
Menggambarkan batasan sistem transaksi yang berinteraksi dengan entitas luar:
* **Pembeli:** Input data opsi kurir dan metode bayar. Output berupa invoice tagihan, nomor VA, dan notifikasi status.
* **Payment Gateway:** Input sinyal konfirmasi transfer (Callback Webhook). Output berupa data tagihan baru.
* **API Logistik:** Input data berat dan alamat. Output berupa nominal ongkos kirim.
* **Penjual:** Output berupa notifikasi instruksi pengiriman barang.

### DFD Level 1 (Proses Utama Transaksi)
* **Proses 1.0 (Kalkulasi Checkout):** Menerima input alamat dan berat produk, melakukan hit ke API Logistik, menghasilkan total biaya, lalu menyimpan draf ke data store [Pesanan].
* **Proses 2.0 (Penerbitan Tagihan):** Memproses metode bayar, melempar data ke API Payment Gateway untuk menghasilkan Nomor VA, lalu menyimpan ke data store [Pembayaran].
* **Proses 3.0 (Verifikasi Pembayaran):** Menerima callback dari Payment Gateway, memperbarui status pada data store [Pembayaran] dan [Pesanan] menjadi "Lunas/Dibayar", serta mengupdate stok di data store [Produk].
* **Proses 4.0 (Distribusi Pesanan Baru):** Mengambil data invoice lunas dari data store [Pesanan] dan meneruskannya ke dashboard Penjual.

---

## 4. Use Case Diagram

### Hak Akses Aktor
* **Pembeli:** Mengelola Keranjang Belanja, Melakukan Checkout (Include: Cek Ongkir Dinamis), Memilih Metode Pembayaran, Menyelesaikan Pembayaran VA.
* **Payment Gateway:** Menerbitkan Nomor Tagihan, Mengirimkan Callback Notifikasi.
* **Penjual:** Menerima Notifikasi Pesanan Baru, Mengonfirmasi Pengiriman Barang.

---

## 5. Activity Diagram

### Alur Proses Checkout hingga Pembayaran
1. Pembeli klik "Bayar" setelah memilih produk dan kurir.
2. Sistem Platform meminta nomor tagihan ke bank dan menampilkan Virtual Account (VA) ke Pembeli.
3. Pembeli melakukan transfer dana lewat M-Banking.
4. Sistem Payment Gateway memvalidasi dana dan mengirim sinyal callback otomatis ke platform.
5. Sistem Platform mengubah status pesanan menjadi "Sudah Dibayar", mengunci dana di escrow, dan meneruskan notifikasi pesanan ke Penjual.

---

## 6. Sequence Diagram

### Komunikasi API Otomatis saat Pembayaran Berhasil
1. Payment Gateway mengirimkan Webhook dengan parameter Status=OK ke komponen API Gateway Platform.
2. API Gateway Platform meneruskan perintah fungsi UpdateStatus() ke komponen Order Service.
3. Order Service mengeksekusi query ke Database Transaksi untuk mengubah isi tabel data invoice (UPDATE Invoice SET status=Paid).
4. Database Transaksi mengembalikan sinyal sukses, kemudian sistem mengirim notifikasi ke Pembeli dan Penjual.

---

## 7. BPMN (Business Process Model and Notation)

### Kolaborasi Proses Bisnis Antar Pool
* **Pool Pembeli:** Menangani aktivitas pencarian produk, inisiasi checkout, transfer dana, hingga menerima status akhir.
* **Pool Tokopedia Platform:** Berperan sebagai orchestrator yang menerima draf, mengunci stok, memproses webhook, dan memperbarui status sistem internal.
* **Pool Payment Gateway:** Fokus pada verifikasi mutasi dana masuk di bank dan memberikan sinyal verifikasi kembali ke platform secara real-time.

---

## 8. Object Diagram

### Instansiasi Objek Transaksi Real-Time
* **Object User:** Representasi data nyata akun pembeli yang sedang aktif melakukan transaksi (contoh properti: id_user, nama, alamat).
* **Object Invoice:** Instansiasi data tagihan dari checkout yang berhasil dibuat (contoh properti: no_invoice, total_harga, status_pembayaran).
* **Object Product:** Instansiasi detail produk yang dibeli untuk memetakan pengurangan jumlah stok di gudang penjual.

---

## 9. Class Diagram

### Struktur Blueprint Kode Program
* **Class User & Product:** Bertindak sebagai entity class yang menyimpan skema atribut data dasar beserta method bawaannya.
* **Class OrderController & PaymentController:** Bertindak sebagai control class yang memproses logika bisnis sistem, seperti kalkulasi total belanja, pembuatan nomor invoice, penerbitan VA, hingga penanganan data callback.

---
Dibuat untuk memenuhi kriteria pemodelan sistem terintegrasi pada mata kuliah Analisis dan Perancangan Sistem Informasi (APSI).
