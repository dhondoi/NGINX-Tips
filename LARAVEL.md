```nginx
# ==========================================
# 1. BLOK GLOBAL / HTTP (Bisa diletakkan di nginx.conf)
# ==========================================
server_tokens off;
client_max_body_size 10M;
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;

# Setup Rate Limiting berdasarkan IP asli (karena langsung terekspos ke internet)
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

# ==========================================
# 2. BLOK HTTP (Redirect ke HTTPS)
# ==========================================
server {
        listen 127.0.0.1:80;
        listen [::1]:80;
        server_name domainanda.com www.domainanda.com;
        allow 127.0.0.1;
        allow ::1;
        deny all;

        # Redirect permanen semua trafik HTTP ke HTTPS
        return 301 https://$host$request_uri;
}

# ==========================================
# 3. BLOK HTTPS (Server Utama)
# ==========================================
server {
        # Catatan: Jika Nginx versi >= 1.25.1, gunakan 'listen 443 ssl;' dan baris baru 'http2 on;'
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name domainanda.com www.domainanda.com;
        # Hanya izinkan localhost / loopback (Cloudflare Tunnel mengirim via localhost)
        allow 127.0.0.1;
        allow ::1;
        # Jika cloudflared running di dalam Docker Container, masukkan subnet Docker (opsional):
        # allow 172.16.0.0/12;
        # Tolak semua IP lainnya (Termasuk IP Lokal 192.168.x.x)
        deny all;

        # --- WAJIB UNTUK LARAVEL ---
        root /var/www/domainanda/public; # Harus mengarah ke folder public
        index index.php index.html index.htm;

        # --- SERTIFIKAT SSL (Pastikan file ini sudah ada) ---
        ssl_certificate /etc/letsencrypt/live/domainanda.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/domainanda.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;

        # --- SECURITY HEADERS ---
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
        
        # Sembunyikan versi PHP
        fastcgi_hide_header X-Powered-By;

        # --- PENANGANAN UTAMA LARAVEL API ---
        location / {
                # Matikan Method HTTP yang Tidak Diperlukan
                # WAJIB tambahkan OPTIONS agar preflight CORS API frontend tidak diblokir
                limit_except GET POST HEAD PUT PATCH DELETE OPTIONS {
                        deny all;
                }

                # Batasi Rate Limiting di level lokasi
                limit_req zone=one burst=10 nodelay; 

                # Routing standar Laravel
                try_files $uri $uri/ /index.php?$query_string;
        }

        # --- PENANGANAN PHP ---
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Sesuaikan dengan versi PHP-FPM Anda (8.2/8.3)
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        # --- OPTIMASI KINERJA: Browser Caching untuk File Statis ---
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|eot|ttf|woff|woff2|svg)$ {
                expires max;
                add_header Cache-Control "public, no-transform";
                access_log off; # Matikan log file statis
        }

        # --- HARDENING: Blokir File Tersembunyi (.env, .git) ---
        # Pengecualian .well-known diperlukan agar Let's Encrypt (Certbot) bisa auto-renew
        location ~ /\.(?!well-known).* {
                deny all;
                access_log off;
                log_not_found off;
        }

        # --- HARDENING: Blokir Skrip Sensitif & PHP di Folder Upload/Storage ---
        location ~* \.(engine|inc|info|install|make|module|profile|test|po|sh|sql|theme|tpl(\.php)?|xtmpl)$ {
                deny all;
        }

        # Blokir eksekusi PHP di folder uploads, images, dan storage Laravel
        location ~* /(uploads|images|storage)/.*\.php$ {
                deny all;
        }
}
```
---
# Buat Symlink

```bash
sudo ln -s /etc/nginx/sites-available/domainanda.com /etc/nginx/sites-enabled/
```
