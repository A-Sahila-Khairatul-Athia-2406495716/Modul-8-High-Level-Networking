# Module 08 - High Level Networking

## 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

- **Unary**: Client kirim 1 request, server balas 1 response. Cocok untuk operasi yang hasilnya langsung bisa didapat, seperti payment processing, kalkulasi sederhana, autentikasi user.

- **Server Streaming**: Client kirim 1 request, server membalas dengan banyak response secara bertahap. Cocok untuk case di mana data yang dikembalikan besar atau terus bertambah, seperti update harga saham real-time, download file besar dalam chunk, news feed.

- **Bi-Directional Streaming**: Client dan server saling kirim pesan secara bersamaan dan independen. Cocok untuk komunikasi real-time dua arah seperti aplikasi chat, live analytics, atau game online.


## 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

- **Authentication**: gRPC tidak otomatis mengamankan siapa yang boleh mengakses service. Perlu ditambahkan mekanisme seperti token (misalnya JWT) yang dikirim lewat metadata di setiap request.

- **Authorization**: Setelah tahu siapa yang request, perlu dicek apakah mereka punya hak akses ke resource tersebut. Ini harus diimplementasi secara manual di level service.

- **Data Encryption**: Secara default, koneksi gRPC tidak terenkripsi. Di production, wajib menggunakan TLS agar data yang dikirim antara client dan server tidak bisa dibaca pihak lain. `tonic` mendukung TLS melalui konfigurasi tambahan.


## 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

- **Error handling**: Kalau salah satu sisi (client atau server) disconnect secara tiba-tiba, sisi yang lain perlu mendeteksi dan menangani kondisi ini dengan baik agar tidak stuck.

- **Backpressure**: Kalau client mengirim pesan lebih cepat dari yang bisa diproses server, buffer bisa penuh. Di implementasi kita, channel buffer dibatasi 10. Kalau penuh, pesan baru bisa tertolak.

- **Concurrency**: Karena client dan server berjalan bersamaan dalam async context, perlu hati-hati dengan shared state agar tidak terjadi race condition.


## 4. What are the advantages and disadvantages of using the `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?

**Keuntungan:**
- Mudah digunakan. Mengubah `mpsc::Receiver` menjadi stream yang langsung bisa dikembalikan sebagai response gRPC.
- Karena `tokio` sudah dipakai untuk async runtime, jadi tidak perlu dependency tambahan.
- Memungkinkan pemisahan antara logic produksi data (di `tokio::spawn`) dan pengiriman ke client, sehingga kode lebih rapi.

**Kekurangan:**
- Buffer size channel harus ditentukan di awal. Kalau terlalu kecil bisa menyebabkan blocking, terlalu besar bisa boros memory.
- Tidak ada mekanisme retry bawaan. Kalau pengiriman gagal (`is_err()`), data tersebut hilang begitu saja.


## 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?

- Pisahkan setiap service ke file tersendiri, misalnya `payment_service.rs`, `transaction_service.rs`, dan `chat_service.rs`, lalu import di `grpc_server.rs`. Kode akan jadi lebih mudah dibaca dan dimaintain.
- Buat shared utility module untuk hal-hal yang dipakai bersama, seperti error handling atau konversi format data.
- Gunakan trait yang sudah digenerate Protobuf sebagai blueprint yang jelas antara implementasi server dan client, sehingga perubahan di satu sisi tidak langsung merusak sisi lain.


## 6. In the `MyPaymentService` implementation, what additional steps might be necessary to handle more complex payment processing logic?

Di implementasi saat ini, `MyPaymentService` langsung mengembalikan `success: true` tanpa benar-benar memproses apapun. Untuk kasus nyata, beberapa langkah tambahan yang diperlukan:

- **Validasi input**: Cek apakah `user_id` valid dan `amount` tidak negatif atau nol.
- **Koneksi ke database atau payment gateway**: Misalnya memanggil API eksternal seperti Midtrans atau Stripe untuk memproses pembayaran sesungguhnya.
- **Error handling yang proper**: Kembalikan `Status::invalid_argument` atau `Status::internal` kalau ada yang gagal, bukan selalu `success: true`.
- **Logging dan audit trail**: Catat setiap transaksi untuk keperluan audit dan debugging.


## 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?

- **Interoperability**: Karena gRPC mendukung banyak bahasa (Rust, Java, Go, Python, dll), service yang berbeda bahasa bisa berkomunikasi dengan mudah menggunakan file `.proto` yang sama sebagai blueprint.
- **Performance**: Dengan HTTP/2 dan Protobuf, komunikasi antar service jauh lebih efisien dibanding REST dengan JSON, terutama saat ada banyak service yang saling berkomunikasi (microservices).
- **Tight coupling pada schema**: Setiap perubahan di file `.proto` perlu diregenerate di semua service yang menggunakannya, sehingga perubahan API perlu dilakukan dengan lebih hati-hati.


## 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

**Keuntungan HTTP/2 (gRPC):**
-  Bisa mengirim banyak request sekaligus dalam satu koneksi TCP tanpa saling menunggu (multiplexing), tidak seperti HTTP/1.1 yang harus antri (head-of-line blocking).
- Header compression (HPack) memungkinkan header yang sama tidak perlu dikirim ulang sepenuhnya di setiap request, sehingga mengurangi overhead jaringan.
- Mendukung server push dan bidirectional streaming secara built-in.

**Kekurangan HTTP/2 (gRPC):**
- Browser tidak bisa langsung memanggil gRPC tanpa bantuan proxy (seperti gRPC-Web), berbeda dengan REST yang didukung semua browser secara native.
- Lebih kompleks untuk didebug karena data dikirim dalam format binary, jadi tidak bisa dibaca langsung seperti JSON di browser DevTools.


## 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

REST menggunakan model request-response berupa: client minta, server jawab, selesai. Ini sebenarnya cukup untuk banyak case, tapi kurang efisien untuk komunikasi real-time karena client harus terus polling (mengirim request berulang) untuk mendapatkan update terbaru.

gRPC dengan bidirectional streaming memungkinkan client dan server saling bertukar pesan kapan saja tanpa perlu membuka koneksi baru setiap kali. Jauh lebih responsif dan efisien untuk skenario real-time seperti chat, notifikasi live, atau monitoring sistem, karena data langsung dikirim begitu tersedia tanpa menunggu request dari client.


## 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

**Protobuf (gRPC):**
- Setiap message harus didefinisikan terlebih dahulu di file `.proto`. Hal ini memastikan struktur data selalu konsisten antara client dan server.
- Ukuran data lebih kecil dan parsing lebih cepat karena format binary.
- Perubahan schema perlu hati-hati agar tidak breaking change, misalnya jangan hapus field yang masih dipakai.

**JSON (REST):**
- Lebih fleksibel. Bisa menambah atau mengubah field tanpa harus recompile kode di semua sisi.
- Mudah dibaca manusia dan didebug langsung di browser atau tools seperti Postman.
- Tapi karena tidak ada schema yang ketat, lebih rentan terhadap kesalahan struktur data yang baru ketahuan saat runtime.

Kesimpulannya, Protobuf lebih cocok untuk sistem internal yang sudah stabil dan butuh performa tinggi, sedangkan JSON lebih cocok untuk public API yang perlu fleksibilitas dan kemudahan akses.