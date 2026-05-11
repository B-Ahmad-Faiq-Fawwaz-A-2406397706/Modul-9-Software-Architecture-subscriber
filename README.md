**a. What is AMQP?**

AMQP atau Advanced Message Queuing Protocol adalah protokol standar terbuka yang digunakan untuk komunikasi antar aplikasi melalui perantara pesan atau message broker. Protokol ini mendefinisikan bagaimana pesan diformat, dikirim, dan dikonfirmasi penerimaannya antara satu aplikasi dengan aplikasi lain. AMQP dirancang agar aplikasi yang ditulis dalam bahasa pemrograman berbeda tetap bisa berkomunikasi secara andal. Protokol ini juga mendukung pengiriman pesan secara asynchronous sehingga pengirim tidak perlu menunggu penerima selesai memproses pesan. Dalam tutorial ini, AMQP digunakan sebagai protokol yang menghubungkan publisher dan subscriber melalui RabbitMQ.

**b. What does `guest:guest@localhost:5672` mean?**

String `guest:guest@localhost:5672` adalah URL koneksi yang digunakan oleh publisher maupun subscriber untuk terhubung ke RabbitMQ. `guest` yang pertama adalah username yang digunakan untuk autentikasi ke RabbitMQ, sedangkan `guest` yang kedua adalah password dari username tersebut. Keduanya merupakan kredensial bawaan yang sudah disediakan oleh RabbitMQ secara default tanpa perlu konfigurasi tambahan. `localhost` menunjukkan bahwa server RabbitMQ berjalan di mesin yang sama dengan aplikasi kita, bukan di server eksternal. Angka `5672` adalah nomor port yang digunakan RabbitMQ untuk menerima koneksi menggunakan protokol AMQP.

## Simulation slow subscriber

![](media/RabbitMQ%20Queue.png)

Dari grafik Queued messages terlihat bahwa jumlah pesan yang mengantri sempat mencapai sekitar 6 pesan sebelum akhirnya turun kembali ke 0. Hal ini terjadi karena subscriber dibuat lambat dengan delay 1 detik per pesan menggunakan thread::sleep, sementara publisher dapat mengirimkan 5 pesan sekaligus dalam waktu yang jauh lebih singkat. Ketika publisher dijalankan beberapa kali secara cepat, pesan terus masuk ke antrian lebih cepat daripada kecepatan subscriber memprosesnya. Total antrian sebanyak itu karena jumlah pesan yang masuk adalah kelipatan 5 (setiap satu kali cargo run publisher mengirim 5 pesan), sedangkan subscriber hanya mampu memproses satu pesan per detik. RabbitMQ menyimpan semua pesan yang belum diproses tersebut di dalam antrian hingga subscriber berhasil memprosesnya satu per satu.

## Running at least three subscribers

![](media/RabbitMQ%20Queue%202.png)

![](media/3%20Terminal.png)

Ketika menjalankan tiga subscriber secara bersamaan, terlihat bahwa pesan dari publisher terdistribusi secara merata ke masing-masing subscriber. Dari screenshot konsol, subscriber pertama memproses Cica, Budi, Amir, Emir, dan Dira, sementara subscriber kedua dan ketiga juga menerima bagian pesannya masing-masing. Hal ini menunjukkan bahwa RabbitMQ secara otomatis membagi beban kerja ke semua subscriber yang terhubung ke antrian yang sama menggunakan mekanisme round-robin. Dari grafik RabbitMQ juga terlihat bahwa jumlah koneksi naik menjadi 4 dan consumers menjadi 4, serta grafik Queued messages tidak menunjukkan penumpukan berarti karena pesan langsung diproses oleh salah satu dari tiga subscriber yang tersedia. Ini adalah solusi yang tepat untuk menangani subscriber yang lambat, yaitu dengan menambah jumlah subscriber agar throughput pemrosesan meningkat secara proporsional. Dari kode publisher dan subscriber, hal yang bisa ditingkatkan adalah menambahkan logging yang lebih informatif agar bisa diketahui subscriber mana yang memproses pesan tertentu, serta menambahkan error handling yang lebih baik jika koneksi ke RabbitMQ terputus.

## Possible improvements

Setelah melihat kode publisher dan subscriber, ada beberapa hal yang bisa diperbaiki. Pertama, pada subscriber tidak ada mekanisme reconnect otomatis jika koneksi ke RabbitMQ terputus, sehingga subscriber akan berhenti bekerja sepenuhnya jika koneksi gagal. Kedua, pada publisher tidak ada konfirmasi bahwa pesan berhasil diterima oleh broker, sehingga jika pengiriman gagal tidak ada penanganan error yang memadai. Ketiga, nilai hardcoded seperti URL koneksi amqp://guest:guest@localhost:5672 sebaiknya dipindahkan ke file konfigurasi atau environment variable agar lebih fleksibel saat di-deploy di environment yang berbeda. Keempat, data pengguna pada publisher juga masih hardcoded langsung di dalam fungsi main, yang seharusnya bisa dibaca dari input eksternal agar lebih dinamis.