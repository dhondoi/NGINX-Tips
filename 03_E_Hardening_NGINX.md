Mari kita **lanjutkan ke tahap pengawasan, pemeliharaan, dan audit keamanan (*Monitoring, Logging, & Auditing*)**.

Setinggi apa pun benteng *hardening* yang kita bangun, tanpa sistem pemantauan yang baik, kita tidak akan pernah tahu jika ada aktivitas mencurigakan atau percobaan peretasan yang sedang berlangsung.

---

## 11. Kustomisasi Log Nginx untuk Audit Keamanan

Secara bawaan, log akses Nginx (`access.log`) tidak mencatat beberapa informasi penting seperti durasi *request*, ID transaksi, atau *user agent* secara detail. Kita bisa membuat *custom log format* khusus audit keamanan.

### Cara Konfigurasi:

Buka file `/etc/nginx/nginx.conf`, lalu tambahkan format log baru di dalam blok `http`:

```nginx
http {
    # Format Log Khusus Audit Keamanan
    log_format security_audit '$remote_addr - $remote_user [$time_local] '
                              '"$request" $status $body_bytes_sent '
                              '"$http_referer" "$http_user_agent" '
                              'request_time=$request_time '
                              'upstream_response_time=$upstream_response_time '
                              'ssl_protocol=$ssl_protocol '
                              'ssl_cipher=$ssl_cipher';

    # Terapkan format baru ini ke access log
    access_log /var/log/nginx/access.log security_audit;
}

```

* **Efek Positif:** Memudahkan analisis forensik digital jika terjadi insiden. Anda bisa mengetahui IP mana yang mengirim *request* lambat (*slow attack*), protokol SSL yang digunakan, serta *user agent* bot yang dicurigai.
* **Efek Samping:** Ukuran file log akan membengkak lebih cepat. Wajib digabungkan dengan **Logrotate** (Langkah #12).

---

## 12. Mencegah Disk Full akibat Log (*Log Rotation*)

Jika server Anda diserang *HTTP Flood* atau mendapat lalu lintas tinggi, file `/var/log/nginx/access.log` bisa membesar hingga belasan Gigabyte dalam hitungan hari. Jika penyimpanan (*disk*) penuh, seluruh sistem Nginx dan database akan **crash total**.

Di Debian 12, Nginx sudah terintegrasi dengan utilitas `logrotate`.

### Cara Konfigurasi & Optimalisasi:

Edit file `/etc/logrotate.d/nginx`:

```bash
sudo nano /etc/logrotate.d/nginx

```

Pastikan aturannya diperketat seperti contoh ini:

```text
/var/log/nginx/*.log {
        daily           # Rotate log setiap hari
        missingok
        rotate 14       # Simpan log hingga 14 hari ke belakang (sisa log lama dihapus)
        compress        # Kompres log lama menjadi file .gz untuk menghemat disk
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}

```

* **Efek Positif:** Mencegah ruang penyimpanan server habis (*Disk Full*) akibat tumpukan berkas log.
* **Efek Samping:** Berkas log di atas 14 hari akan terhapus otomatis. Jika membutuhkan retensi log jangka panjang (misal untuk kepatuhan ISO/PCI-DSS), Anda harus mengirimkan log tersebut ke server terpusat (*Centralized Log Server/SIEM*).

---

## 13. Audit Keamanan Otomatis Menggunakan Lynis

Setelah semua rangkaian *hardening* (dari part 1 hingga part ini) selesai diterapkan, Anda perlu melakukan pengujian/audit independen di lokal server Debian Anda. **Lynis** adalah alat audit keamanan terpopuler untuk Linux/Debian.

### Langkah-Langkah Audit:

1. **Instal Lynis di Debian 12:**
```bash
sudo apt update && sudo apt install lynis -y

```


2. **Jalankan Audit Khusus Nginx & Sistem:**
```bash
sudo lynis audit system

```


3. **Cek Skor Keamanan (*Hardening Index*):**
Di akhir proses, Lynis akan memberikan skor ketahanan sistem (misal `Hardening index : 82 [####################]`) beserta saran perbaikan (*Hardening Suggestions*) khusus Nginx yang masih perlu Anda tingkatkan.

---

## Checklist Lengkap Rangkaian Hardening Nginx

Berikut adalah peta jalan (*roadmap*) akhir dari seluruh konfigurasi hardening yang telah kita bahas:

1. **Fase Dasar:** Sembunyikan versi (`server_tokens off`), atur batas buffer, dan hapus modul tak terpakai.
2. **Fase Struktur:** Pisahkan konfigurasi secara modular (`conf.d/` dan `snippets/`).
3. **Fase Enkripsi:** Matikan TLS 1.0/1.1, pakai TLS 1.2/1.3, pasang *Security Headers* (HSTS, CSP, X-Frame), dan aktifkan OCSP Stapling.
4. **Fase Akses & Sistem:** Isolasi `systemd`, perketat *permission* folder (`755/644`), dan kunci folder upload dari eksekusi skrip.
5. **Fase Proteksi Aktif:** Terapkan Rate Limiting, Connection Limiting, integrasi Fail2ban, dan pasang ModSecurity WAF.
6. **Fase Pemeliharaan:** Kustomisasi log audit, rotasi berkas log, dan audit berkala dengan Lynis.
