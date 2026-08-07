```nginx
# ==========================================
# 1. BLOK GLOBAL / HTTP
# (Pastikan ini ada di nginx.conf di dalam blok http {},
# JANGAN letakkan di dalam sites-available/default jika terpisah)
# ==========================================
server_tokens off;
client_max_body_size 10M;
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;

# Gunakan header dari Cloudflare untuk mendeteksi IP Asli pengunjung
set_real_ip_from 127.0.0.1; # Sesuaikan jika cloudflared pakai IP Docker (misal 172.x.x.x)
real_ip_header CF-Connecting-IP;

# Setup Rate Limiting berdasarkan IP Asli pengunjung
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

# ==========================================
# 2. BLOK SERVER UTAMA (Hanya Port 80 - CF Tunnel yang akan handle HTTPS di luar)
# ==========================================
server {
        listen 80;
        listen [::]:80;
        server_name domainanda.com www.domainanda.com;

        # --- WAJIB UNTUK LARAVEL ---
        root /var/www/domainanda/public;
        index index.php index.html index.htm;

        # --- SECURITY HEADERS (Dari CF Tunnel ke pengunjung) ---
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        # CSP disesuaikan jika Laravel butuh Vite (saat development) atau script CDN eksternal
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

        # Sembunyikan versi PHP
        fastcgi_hide_header X-Powered-By;

        # --- OPTIMASI KINERJA: Gzip Compression ---
        gzip on;
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

        # --- PENANGANAN UTAMA LARAVEL ---
        location / {
                # WAJIB tambahkan OPTIONS untuk Preflight CORS API
                limit_except GET POST HEAD PUT PATCH DELETE OPTIONS {
                        deny all;
                }

                limit_req zone=one burst=10 nodelay;
                try_files $uri $uri/ /index.php?$query_string;
        }

        # --- PENANGANAN PHP ---
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Sesuaikan php8.x
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        # --- OPTIMASI KINERJA: Browser Caching ---
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|eot|ttf|woff|woff2|svg)$ {
                expires max;
                add_header Cache-Control "public, no-transform";
                access_log off;
        }

        # --- HARDENING ---
        # Blokir Akses ke File/Folder Tersembunyi (.env, .git)
        location ~ /\.(?!well-known).* {
                deny all;
                access_log off;
                log_not_found off;
        }

        # Blokir eksekusi PHP di folder storage (Laravel)
        location ~* /storage/.*\.php$ {
                deny all;
        }
}
```
---
# Buat Symlink

```bash
sudo ln -s /etc/nginx/sites-available/domainanda.com /etc/nginx/sites-enabled/
```
