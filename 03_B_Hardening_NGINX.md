Mari kita lanjutkan ke tingkat hardening Nginx yang lebih dalam (*Advanced Hardening*).

Setelah sebelumnya kita membahas dasar-dasar seperti mematikan `server_tokens` dan membatasi buffer, berikut adalah langkah-langkah keamanan tingkat lanjut yang sangat krusial untuk server produksi.

---

## 1. Amankan TLS/SSL (HTTPS Hardening)

Menggunakan SSL saja tidak cukup jika protokol dan *cipher suite* yang digunakan sudah usang (misalnya TLS 1.0 atau 1.1 yang rentan terhadap serangan *POODLE* atau *BEAST*).

### Cara Konfigurasi:

Di dalam blok `server` yang menggunakan HTTPS (port 443):

```nginx
server {
    listen 443 ssl http2;
    server_name domainanda.com;

    # Hanya izinkan protokol TLS modern yang aman
    ssl_protocols TLSv1.2 TLSv1.3;

    # Batasi Cipher Suites ke algoritma kriptografi yang kuat
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Optimasi SSL Cache untuk performa tanpa mengorbankan keamanan
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
}

```

* **Efek Positif:** Melindungi komunikasi data dari *eavesdropping* (penyadapan) dan serangan *man-in-the-middle* (MITM) tingkat tinggi. Mengamankan skor SSL Anda menjadi **A+** di *Qualys SSL Labs*.
* **Efek Samping:** Perangkat atau browser yang sangat lama (seperti Windows XP dengan IE8, atau Android versi 4.4 ke bawah) **tidak akan bisa mengakses situs Anda** karena tidak mendukung TLS 1.2+.

---

## 2. Terapkan Content Security Policy (CSP)

CSP adalah salah satu *header* keamanan paling ampuh untuk mencegah serangan **Cross-Site Scripting (XSS)** dan **Data Injection**. CSP mengatur dari mana saja browser boleh memuat skrip, gambar, atau *stylesheet*.

### Cara Konfigurasi:

Tambahkan di dalam blok `server` atau `location`:

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://trustedscripts.example.com; object-src 'none';" always;

```

* **Efek Positif:** Walaupun peretas berhasil menyisipkan skrip *malicious* (misal `<script src="http://hacker.com/evil.js">`), browser pengunjung akan **menolak mengeksekusi** skrip tersebut.
* **Efek Samping:** Sangat riskan merusak tampilan/fungsi web jika tidak dikonfigurasi dengan teliti. Jika situs Anda memakai Google Analytics, FontAwesome, atau CDN eksternal tanpa memasukkannya ke daftar izin CSP, fitur-fitur tersebut akan **terblokir total**.

---

## 3. Cegah Eksekusi Skrip di Folder Upload

Jika web Anda memiliki fitur upload gambar/file (misalnya WordPress atau CMS lain), peretas sering mencoba mengunggah skrip PHP/Shell lalu mengeksekusinya secara langsung.

### Cara Konfigurasi:

Blokir eksekusi berkas skrip di direktori upload (misal `/uploads/` atau `/media/`):

```nginx
location /uploads/ {
    # Matikan eksekusi skrip PHP
    location ~ \.php$ {
        deny all;
    }
}

```

* **Efek Positif:** Meskipun peretas berhasil menembus validasi upload dan menyimpan file `webshell.php` di folder gambar, mereka tidak akan bisa menjalankannya (mendapatkan respon **403 Forbidden**).
* **Efek Samping:** Hampir tidak ada, selama folder upload tersebut murni hanya digunakan untuk menyimpan media statis (gambar, PDF, video).

---

## 4. Gunakan Fail2ban untuk Mitigasi Serangan Otomatis

Meskipun ini dilakukan di luar file Nginx, **Fail2ban** adalah pasangan wajib untuk hardening Nginx. Fail2ban secara otomatis memindai log error Nginx dan memblokir IP penyerang di level firewall (`iptables`/`nftables`).

### Cara Kerja Singkat:

1. Instal Fail2ban:
```bash
sudo apt install fail2ban -y

```


2. Buat jail khusus Nginx di `/etc/fail2ban/jail.local`:
```ini
[nginx-http-auth]
enabled = true
port    = http,https
logpath = /var/log/nginx/error.log
maxretry = 3
bantime  = 1h

```



* **Efek Positif:** Jika ada bot mencoba *brute-force* password (gagal login 3 kali berturut-turut), IP penyerang akan **langsung diblokir oleh sistem** selama 1 jam.
* **Efek Samping:** Pengguna asli yang lupa password dan salah mengetik berkali-kali bisa ikut terkunci dari server selama durasi `bantime`.

---

## Ringkasan Matriks Hardening Lanjutan

| Fitur Hardening | Fungsi Utama | Potensi Efek Samping |
| --- | --- | --- |
| **TLS 1.2/1.3 Only** | Mencegah dekripsi HTTPS oleh peretas. | Browser/perangkat jadul gagal terkoneksi. |
| **Content Security Policy** | Mencegah XSS & injeksi skrip jahat. | Merekontruksi cara kerja aset eksternal (CDN/Analytics). |
| **Disable Exec di Uploads** | Mencegah eksekusi *Webshell/Backdoor*. | Hampir tidak ada (aman). |
| **Fail2ban Integration** | Otomatis blokir IP pembobol/bot. | Pengguna yang lupa password bisa terkena blokir sementara. |

---

## Rekomendasi Alur Penerapan

1. **Jalankan Uji Coba Konfigurasi:**
```bash
sudo nginx -t

```


2. **Reload Nginx:**
```bash
sudo systemctl reload nginx

```


3. **Uji Keamanan Situs Anda:**
* Gunakan [SSL Labs](https://www.ssllabs.com/ssltest/) untuk mengevaluasi kekuatan SSL/TLS Anda.
* Gunakan [Security Headers](https://securityheaders.com/) untuk mengecek kerapian *header* keamanan Nginx Anda.
