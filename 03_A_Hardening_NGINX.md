
---

Hardening Nginx adalah proses mengamankan konfigurasi Nginx agar server tidak mudah diserang (*exploit*), meminimalkan kebocoran informasi, dan menolak lalu lintas berbahaya.

Berikut adalah langkah-langkah **hardening Nginx** yang paling efektif beserta **efek langsungnya** pada sistem Anda.

---
## 0. Buat File

```bash
sudo nano /etc/nginx/conf.d/<nama_file>.conf
```

## 1. Sembunyikan Versi Nginx (*Server Tokens*)

Secara bawaan, Nginx menampilkan versi spesifiknya pada *header* HTTP dan halaman error. Peretas memanfaatkan informasi ini untuk mencari celah keamanan (*vulnerability*) spesifik versi tersebut.

### Cara Konfigurasi:

Tambahkan baris berikut di dalam blok `http (/etc/nginx/conf.d/<nama_file>.conf)`:

```nginx
server_tokens off;
```

* **Efek Positif:** Peretas tidak bisa lagi melihat versi Nginx Anda melalui *header* HTTP (`Server: nginx`) atau halaman error 404/500.
* **Efek Samping:** Sangat minim/hampir tidak ada. Hanya mematikan info versi.

---

## 2. Batasi Ukuran Buffer & Request (*Prevent Buffer Overflow / DoS*)

Serangan *Buffer Overflow* atau *Denial of Service* (DoS) sering dilakukan dengan mengirimkan request payload atau header berukuran sangat besar untuk menghabiskan memori server.

### Cara Konfigurasi:

Tambahkan baris berikut di dalam blok `http (/etc/nginx/conf.d/<nama_file>.conf)`:

```nginx
# Batasi ukuran body request (misal: upload file maks 10MB)
client_max_body_size 10M;

# Batasi ukuran header request
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;
```

* **Efek Positif:** Mencegah peretas membebani RAM server dengan payload raksasa.
* **Efek Samping:** Jika pengguna mencoba mengunggah (*upload*) file yang ukurannya melebihi batas `client_max_body_size` (misal file 15MB pada contoh di atas), Nginx akan menolaknya dengan error **`413 Request Entity Too Large`**.

---

## 3. Tambahkan Security Headers (HTTP Response Headers)

Security headers menginstruksikan browser klien untuk menerapkan perlindungan ekstra terhadap serangan seperti *Cross-Site Scripting (XSS)*, *Clickjacking*, dan *MIME-sniffing*.

### Cara Konfigurasi:

Tambahkan baris berikut di dalam blok `server (/etc/nginx/sites-available/<nama_file>)`  atau `http (/etc/nginx/conf.d/<nama_file>.conf)`:

```nginx
# Mencegah situs Anda dimasukkan ke dalam <iframe/frame> (Anti-Clickjacking)
add_header X-Frame-Options "SAMEORIGIN" always;

# Mencegah browser menebak-nebak tipe konten (Anti-MIME Sniffing)
add_header X-Content-Type-Options "nosniff" always;

# Mengaktifkan filter XSS di browser tua
add_header X-XSS-Protection "1; mode=block" always;

# Memaksa browser hanya menggunakan HTTPS (HSTS) - *Wajib jika pakai SSL*
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

```

* **Efek Positif:** Meningkatkan skor keamanan situs Anda di alat audit seperti *SSL Labs* atau *Security Headers*, serta melindungi pengguna dari eksekusi skrip jahat.
* **Efek Samping:** * `X-Frame-Options`: Jika web Anda butuh di-embed oleh situs lain melalui `<iframe>`, fitur tersebut akan terblokir.
* `HSTS`: Jika SSL/HTTPS Anda mati atau *expired*, pengunjung **sama sekali tidak bisa mengakses** situs Anda bahkan lewat HTTP biasa.



---

## 4. Matikan Method HTTP yang Tidak Diperlukan

Secara umum, aplikasi web hanya membutuhkan method `GET`, `POST`, dan `HEAD`. Method lain seperti `DELETE`, `PUT`, atau `TRACE` bisa disalahgunakan jika aplikasi tidak teramankan dengan baik.

### Cara Konfigurasi:

Di dalam blok `location / (/etc/nginx/sites-available/<nama_file>)` pada konfigurasi site Anda:

```nginx
location / {
    limit_except GET POST HEAD {
        deny all;
    }
}

```

* **Efek Positif:** Memblokir percobaan peretas yang ingin mengeksekusi method destruktif (seperti `DELETE` atau `PUT` langsung ke server).
* **Efek Samping:** Jika web Anda menggunakan REST API yang memanfaatkan method `PUT`, `PATCH`, atau `DELETE`, API tersebut akan gagal berfungsi (**Error 403 Forbidden**). Sertakan method yang dibutuhkan di dalam `limit_except`.

---

## 5. Batasi Rate Limiting (Mencegah Brute Force / DoS)

Fitur ini membatasi berapa kali IP yang sama bisa mengirimkan *request* dalam periode waktu tertentu.

### Cara Konfigurasi:

1. Di blok `http (/etc/nginx/conf.d/<nama_file>.conf)`:
```nginx
# Membuat zona bernama 'one' dengan alokasi memori 10MB, maks 10 request/detik per IP
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

```


2. Di blok `location (/etc/nginx/sites-available/<nama_file>)` (misal halaman login):
```nginx
location /login/ {
    limit_req zone=one burst=5 nodelay;
}

```



* **Efek Positif:** Menghentikan serangan *brute force* pada halaman login dan serangan bot spamming/DoS skala kecil.
* **Efek Samping:** Jika pengguna yang sah melakukan *refresh* halaman terlalu cepat atau berada di bawah NAT/WiFi publik yang sama (banyak pengguna dengan 1 IP publik), mereka bisa terkena error **`503 Service Temporarily Unavailable`**.

---

## 6. Blokir Akses ke File Sensitif & Tersembunyi

Banyak framework atau CMS menyisakan file konfigurasi (`.env`, `.git`, `.htaccess`) di direktori root web yang berisi password database atau kunci rahasia.

### Cara Konfigurasi:

Tambahkan blok ini di konfigurasi `server (/etc/nginx/sites-available/<nama_file>)`:

```nginx
# Blokir semua file yang diawali dengan titik (seperti .env, .git)
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# Blokir file skrip sensitif tertentu
location ~* \.(engine|inc|info|install|make|module|profile|test|po|sh|sql|theme|tpl(\.php)?|xtmpl)$ {
    deny all;
}

```

* **Efek Positif:** Mencegah kebocoran berkas rahasia seperti `.env` (password DB) atau repositori `.git` yang tidak sengaja terunggah.
* **Efek Samping:** Jika aplikasi Anda membutuhkan akses langsung ke file yang berakhiran ekstensi di atas, file tersebut tidak akan bisa diunduh/diakses.

---

## Ringkasan Efek Hardening Nginx

| Langkah Hardening | Potensi Efek Samping | Cara Mengatasi Efek Samping |
| --- | --- | --- |
| **Server Tokens Off** | Hampir tidak ada. | Tidak ada. |
| **Buffer Limit** | Upload file besar gagal (`413`). | Naikkan `client_max_body_size`. |
| **Security Headers** | *Frame* gagal dimuat / HSTS terkunci. | Sesuaikan kebijakan `X-Frame-Options` jika butuh embed. |
| **Limit HTTP Method** | REST API (`PUT`/`DELETE`) error 403. | Tambahkan method yang diizinkan ke `limit_except`. |
| **Rate Limiting** | Pengguna asli kena blokir (`503`). | Naikkan nilai `rate` atau batasi hanya pada URL sensitif saja. |
| **Blokir File Hidden** | File sistem tertentu tak bisa diakses. | Kecualikan lokasi spesifik jika memang aman. |

> **Langkah Wajib Setelah Mengubah Konfigurasi:**
> Selalu uji sintaks konfigurasi terlebih dahulu sebelum memuat ulang Nginx:
> ```bash
> sudo nginx -t
> sudo systemctl reload nginx
> 
> ```
> 
>
