# Reflection
1. Perbedaan Utama antara Unary, Server Streaming, dan Bi-directional Streaming RPC
- Unary RPC: Klien mengirimkan satu request tunggal dan menerima satu response tunggal dari server.
Skenario: Cocok untuk operasi CRUD standar, autentikasi (login), atau memproses satu aksi spesifik seperti fungsi ProcessPayment.
- Server Streaming RPC: Klien mengirimkan satu request, namun server membalas dengan aliran data (stream) yang berisi banyak response secara berurutan.
Skenario: Ideal untuk mengunduh dataset berukuran besar, mengambil log sistem secara real-time, atau memuat daftar riwayat panjang seperti fungsi GetTransactionHistory agar memori tidak kelebihan beban.
- Bi-directional Streaming RPC: Klien dan server dapat saling mengirimkan aliran pesan secara bersamaan (simultan) melalui satu koneksi secara penuh (full-duplex).
Skenario: Sangat tepat untuk aplikasi yang membutuhkan interaktivitas real-time tinggi, seperti aplikasi chat (ChatService), multiplayer game, atau sinkronisasi data langsung antar node dalam sistem terdistribusi.

2. Pertimbangan Keamanan dalam Implementasi gRPC di Rust
- Enkripsi Data: gRPC menggunakan HTTP/2, yang secara praktis sangat bergantung pada enkripsi TLS. Menerapkan mTLS (Mutual TLS) sangat penting agar klien dan server dapat saling memverifikasi identitas satu sama lain sebelum bertukar data.
- Autentikasi: Karena gRPC tidak menggunakan session cookies seperti web tradisional, autentikasi sering dilakukan menggunakan token (seperti JWT) yang disisipkan melalui gRPC Metadata (mirip dengan HTTP Headers). Di Rust, ini dapat ditangani menggunakan Interceptor pada level tonic.
- Otorisasi: Setelah pengguna terautentikasi, layanan perlu memastikan hak aksesnya (Role-Based Access Control). Penting untuk memvalidasi request secara ketat untuk mencegah eskalasi hak istimewa (privilege escalation).

3. Tantangan Menangani Bidirectional Streaming di Rust (Skenario Aplikasi Chat)
- State Management: Melacak klien mana saja yang terhubung dan menyebarkan (broadcast) pesan ke klien yang tepat membutuhkan data structure bersama (seperti HashMap atau DashMap) yang harus dibungkus dengan Arc dan Mutex atau RwLock.
- Race Conditions & Deadlocks: Menahan lock melintasi titik tunggu (.await) dapat menyebabkan deadlock pada aplikasi asinkronus. Anda perlu memastikan lock dilepas secepat mungkin atau menggunakan primitif sinkronisasi milik Tokio (tokio::sync::Mutex).
- Resource Leaks: Jika klien terputus secara mendadak, server harus memiliki mekanisme untuk mendeteksinya dan membersihkan koneksi yang menggantung agar tidak terjadi kebocoran memori (memory leak) akibat channel yang dibiarkan terbuka.

4. Keuntungan dan Kerugian Menggunakan tokio_stream::wrappers::ReceiverStream
- Keuntungan: Pustaka ini menyediakan jembatan yang sangat elegan antara channel bawaan Tokio (mpsc::Receiver) dengan trait Stream milik gRPC. Ini memudahkan pemisahan logika bisnis dari antarmuka jaringan; Anda cukup memutar task latar belakang (tokio::spawn) untuk memproses data dan mengirimkannya ke saluran tx, sementara ReceiverStream menangani sisa pengiriman ke klien secara otomatis.
- Kerugian: Penggunaan channel mpsc membutuhkan penentuan kapasitas buffer. Jika buffer terlalu kecil dan klien membaca terlalu lambat, server bisa terblokir. Jika terlalu besar, bisa memakan banyak memori sistem.

5. Struktur Kode gRPC Rust untuk Penggunaan Ulang dan Modularitas
Untuk memastikan skalabilitas jangka panjang dan menerapkan prinsip-prinsip desain yang solid (seperti prinsip SOLID dan kemudahan unit testing):

- Layering: Pisahkan kode definisi proto (lapisan transport) dari lapisan aplikasi (logika bisnis) dan lapisan domain/repositori (akses database).
- Dependency Injection: Definisikan interaksi ke database atau layanan pihak ketiga menggunakan Traits. Struct layanan gRPC (seperti MyPaymentService) harus menerima injeksi dari trait tersebut, bukan bergantung pada implementasi spesifik. Ini memungkinkan pembuatan mock dengan mudah sehingga mencapai code coverage yang tinggi saat testing.
- Pemisahan Workspace: Untuk proyek skala besar, kode client, server, dan proto-build dapat diletakkan di crates berbeda dalam satu Cargo Workspace.

6. Langkah Tambahan untuk MyPaymentService yang Kompleks
Saat ini implementasi hanya merespons success: true. Logika nyata akan membutuhkan:

- Validasi Input: Memeriksa apakah amount lebih besar dari nol dan user_id valid.
- Integrasi Pihak Ketiga & Idempotency: Berkomunikasi dengan Payment Gateway (Stripe/Midtrans). Harus menerapkan kunci idempotency agar masalah jaringan tidak menyebabkan tagihan ganda (double charge).
- Properti ACID: Memastikan bahwa pengurangan saldo dan pencatatan transaksi terjadi dalam satu database transaction tunggal.
- Penanganan Error Struktural: Memetakan error internal ke kode Status gRPC yang tepat (misalnya Status::internal, Status::invalid_argument).

7. Dampak gRPC pada Arsitektur Sistem Terdistribusi dan Interoperabilitas
- Arsitektur: Mengadopsi gRPC sangat mendorong pendekatan microservices. Komunikasi antar layanan menjadi jauh lebih efisien dan terstruktur karena kontrak datanya sangat jelas.
- Interoperabilitas : Adanya file .proto sebagai sumber kebenaran tunggal memungkinkan satu tim untuk menulis server menggunakan Rust, sementara tim lain mengembangkan klien menggunakan Java, Python, atau Flutter tanpa perlu menerka-nerka struktur payload JSON. Namun, kelemahannya adalah gRPC tidak didukung secara native oleh browser web biasa tanpa bantuan proxy perantara (seperti Envoy atau gRPC-Web).

8. Keuntungan dan Kerugian HTTP/2 (gRPC) vs HTTP/1.1 (REST) atau WebSockets
- Keuntungan HTTP/2: Mendukung Multiplexing (banyak permintaan dalam satu koneksi TCP tanpa saling memblokir), kompresi header (HPACK), dan pengiriman data biner yang membuat parsing sangat ringan di sisi CPU dibandingkan dengan parsing teks polos.
- Kerugian HTTP/2: Pengembang tidak bisa lagi menggunakan alat sederhana seperti cURL standar atau membaca payload di jaringan semudah membaca JSON di HTTP/1.1 karena format datanya biner dan terenkripsi.
- WebSockets: WebSockets menawarkan koneksi dua arah tetapi tanpa struktur yang baku. Sedangkan gRPC menawarkan koneksi dua arah sekaligus kerangka RPC dengan jaminan struktur tipe data bawaan.

9. REST API vs Bidirectional Streaming gRPC untuk Interaksi Real-Time
- REST API beroperasi secara eksklusif menggunakan pola request-response tanpa penyimpanan status (stateless). Untuk mencapai sifat "real-time" dengan REST, klien harus terus-menerus melakukan polling, yang memboroskan bandwidth dan CPU, serta memiliki latensi tinggi.
- Bidirectional Streaming gRPC membuat saluran tetap hidup dan stateful. Server dapat langsung "mendorong" (push) data ke klien begitu ada perubahan data, menghasilkan latensi dalam hitungan milidetik dan meminimalkan beban di level jaringan.

10. Implikasi Pendekatan Berbasis Skema (Protocol Buffers) vs JSON (REST)
- Protocol Buffers (gRPC): Mengharuskan skema divalidasi dan dikompilasi (ke struct Rust) sebelum program dijalankan (compile-time safety). Hal ini menghilangkan tebakan tipe data, memperkecil ukuran payload, dan memudahkan penjagaan kompatibilitas mundur.
- JSON (REST): Sangat fleksibel yang memungkinkan prototipe dibuat lebih cepat dan datanya mudah dibaca oleh manusia. Namun, di aplikasi berskala besar, perubahan struktur JSON secara mendadak oleh satu tim dapat mematahkan sistem tim lain pada saat runtime akibat kurangnya contract testing yang ketat layaknya .proto.
