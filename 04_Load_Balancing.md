Tentu, mari kita lihat bagaimana NGINX dikonfigurasi untuk menjalankan fungsi *load balancing*.

Bayangkan Anda memiliki sebuah situs web (`www.aplikasi-saya.com`) yang sangat ramai. Agar server tidak *down* karena kelebihan beban, Anda menyiapkan **tiga server aplikasi** (backend) yang identik. NGINX akan berdiri di depan ketiga server ini untuk menerima semua pengunjung, lalu membagi-bagikan pengunjung tersebut ke ketiga server secara adil.

Berikut adalah contoh penulisan konfigurasinya di dalam file `nginx.conf`:

```nginx
# 1. Mendefinisikan grup server backend (Upstream)
upstream server_aplikasi_saya {
    # Secara default, NGINX menggunakan algoritma Round Robin (bergantian: 1-2-3, 1-2-3)
    server 192.168.1.10:8080; # Server Backend 1
    server 192.168.1.11:8080; # Server Backend 2
    server 192.168.1.12:8080; # Server Backend 3
}

# 2. Mengatur server virtual untuk menerima lalu lintas masuk
server {
    listen 80;
    server_name www.aplikasi-saya.com;

    location / {
        # 3. Meneruskan semua permintaan ke grup 'server_aplikasi_saya'
        proxy_pass http://server_aplikasi_saya;
        
        # Pengaturan tambahan untuk meneruskan informasi asli pengguna ke backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

### Penjelasan Komponen Utama:

* **`upstream`**: Blok ini berfungsi untuk membuat grup server. Pada contoh di atas, kita membuat grup bernama `server_aplikasi_saya` yang berisi tiga alamat IP server yang berbeda.
* **`server` (di dalam `upstream`)**: Mendefinisikan alamat IP dan *port* dari masing-masing server backend yang siap menerima beban kerja.
* **`proxy_pass`**: Perintah sakti yang memberitahu NGINX: *"Tolong ambil semua permintaan dari pengunjung (yang masuk ke `location /`), lalu lemparkan ke grup `http://server_aplikasi_saya`."*

### Mengubah Metode Pembagian Beban (Algoritma)

Pada contoh di atas, NGINX menggunakan metode **Round Robin** (mengirim pengunjung ke Server 1, lalu pengunjung berikutnya ke Server 2, lalu Server 3, dan kembali lagi ke Server 1). Jika Anda ingin mengubah perilakunya, Anda cukup menambahkan satu baris instruksi di dalam blok `upstream`:

**1. Least Connections (Koneksi Paling Sedikit)**
NGINX akan memeriksa server mana yang sedang paling tidak sibuk (paling sedikit menangani koneksi aktif) dan mengirim pengunjung baru ke server tersebut.

```nginx
upstream server_aplikasi_saya {
    least_conn;  # Tambahkan baris ini
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

```

**2. IP Hash**
NGINX akan menggunakan alamat IP pengunjung sebagai patokan. Jika pengunjung A dialokasikan ke Server 1 pada kunjungan pertama, maka NGINX akan memastikan pengunjung A selalu diarahkan ke Server 1 pada kunjungan berikutnya (sangat berguna jika aplikasi Anda membutuhkan sistem *login/session*).

```nginx
upstream server_aplikasi_saya {
    ip_hash;     # Tambahkan baris ini
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

```
