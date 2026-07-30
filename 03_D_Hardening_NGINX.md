Mari kita **lanjut ke tingkat Hardening & Keamanan Infrastruktur (Web Application Firewall / WAF & Anti-DDoS)**.

Setelah SSL terenkripsi kuat dan sistem file diisolasi, ancaman terbesar berikutnya adalah serangan di tingkat aplikasi (*Layer 7*), seperti **SQL Injection (SQLi)**, **Cross-Site Scripting (XSS)**, dan **HTTP Flood (DDoS)**.

---

## 8. Pasang ModSecurity WAF (Web Application Firewall)

Nginx secara bawaan hanya memproses request. Dengan menambahkan **ModSecurity**, Nginx Anda memiliki *satpam* cerdas yang menganalisis isi payload request secara *real-time* sebelum diteruskan ke aplikasi web.

### Langkah 1: Instal ModSecurity Module

Di Debian 12 (Bookworm), Anda bisa menginstal modul ModSecurity langsung dari repositori:

```bash
sudo apt install libnginx-mod-http-modsecurity -y

```

### Langkah 2: Aktifkan Ruleset Otomatis (OWASP CRS)

Gunakan aturan standar industri **OWASP Core Rule Set (CRS)** yang secara otomatis memblokir serangan populer (SQLi, XSS, Local File Inclusion, dll.):

1. Aktifkan modul di file `/etc/nginx/nginx.conf` atau direktori `conf.d`:
```nginx
http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
}

```



* **Efek Positif:** Memblokir percobaan pembobolan web/database secara otomatis bahkan jika skrip web Anda (PHP/Node.js/Python) memiliki celah keamanan (*vulnerability*).
* **Efek Samping:** *False Positive* (Request pengguna asli/sah yang dianggap berbahaya oleh sistem bisa ikut terblokir/error 403). Perlu penyesuaian (*tuning*) aturan pada tahap awal instalasi.

---

## 9. Proteksi Serangan Slowloris & Slow HTTP Attacks

Serangan **Slowloris** dilakukan penyerang dengan cara membuka banyak koneksi ke Nginx, lalu mengirimkan data secara *sangat lambat* bit demi bit. Ini membuat koneksi Nginx menggantung sampai server kehabisan alokasi slot koneksi untuk pengguna lain.

### Cara Konfigurasi:

Atur batas waktu penerimaan request dari klien agar Nginx tidak menunggu terlalu lama. Tambahkan di blok `http` pada `/etc/nginx/nginx.conf`:

```nginx
http {
    # Batas waktu maksimal membaca header dari klien (misal 10 detik)
    client_header_timeout 10s;

    # Batas waktu maksimal membaca body dari klien (misal 10 detik)
    client_body_timeout 10s;

    # Batas waktu menutup koneksi yang diam/idle
    keepalive_timeout 15s;

    # Batas waktu mengirimkan respons balik ke klien
    send_timeout 10s;
}

```

* **Efek Positif:** Mencegah server *down/unresponsive* akibat kehabisan koneksi dari serangan *Slow HTTP/Slowloris*.
* **Efek Samping:** Pengguna dengan koneksi internet yang **sangat lambat atau tidak stabil** (misal jaringan edge di pelosok) mungkin mengalami pemutusan koneksi secara tiba-tiba jika pengiriman data mereka melebihi durasi timeout di atas.

---

## 10. Membatasi Koneksi Ekstrem Per IP (Anti HTTP Flood)

Berbeda dengan *Rate Limiting* (yang membatasi jumlah request per detik), **Connection Limiting** membatasi *berapa banyak soket koneksi simultan* yang boleh dibuka oleh 1 IP secara bersamaan.

### Cara Konfigurasi:

1. Di blok `http`:
```nginx
# Alokasikan memori 10MB untuk mencatat koneksi per IP
limit_conn_zone $binary_remote_addr zone=addr:10m;

```


2. Di blok `server` atau `location`:
```nginx
server {
    # Batasi maksimal 15 koneksi simultan dari 1 IP yang sama
    limit_conn addr 15;
}

```



* **Efek Positif:** Mencegah bot/script penyerang membuka ribuan *concurrent connections* yang bisa melumpuhkan RAM dan CPU server.
* **Efek Samping:** Aplikasi web modern yang mengunduh banyak file aset (CSS, JS, Gambar) secara paralel mungkin akan terhambat jika nilai limit ditaruh terlalu rendah (di bawah 5-10 koneksi).

---

## Matriks Rangkuman Seluruh Arsitektur Keamanan Nginx

| Lapisan Keamanan | Alat / Direktif Utama | Menghentikan Jenis Serangan |
| --- | --- | --- |
| **System Level** | Systemd Protect, Permission `755/644` | Privilege Escalation, Remote Code Execution (RCE). |
| **Network Level** | TLS 1.3, HSTS, Fail2ban | Eavesdropping, MITM, Brute Force IP. |
| **Protocol Level** | Timeout Limits, Connection Limit | Slowloris, SYN Flood, HTTP Flood (DDoS). |
| **Application Level** | ModSecurity WAF, Security Headers | SQL Injection, XSS, Clickjacking, LFI/RFI. |

---

### Langkah Verifikasi Akhir

Selalu periksa sintaks sebelum memuat ulang server:

```bash
sudo nginx -t && sudo systemctl reload nginx

```
