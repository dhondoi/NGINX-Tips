```nginx
# ==========================================
# 1. BLOK GLOBAL / HTTP
# (Letakkan di nginx.conf di dalam blok http {})
# ==========================================
server_tokens off;
client_max_body_size 10M;
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;

# Deteksi IP Asli dari Cloudflare
# Gunakan header dari Cloudflare untuk mendeteksi IP Asli pengunjung
set_real_ip_from 127.0.0.1; # Sesuaikan jika cloudflared pakai IP Docker (misal 172.x.x.x)
real_ip_header CF-Connecting-IP;

# Setup Rate Limiting berdasarkan IP Asli pengunjung
limit_req_zone $binary_remote_addr zone=kuma_limit:10m rate=30r/s;


# ==========================================
# 2. BLOK SERVER UTAMA (Hanya Port 80 - CF Tunnel)
# ==========================================
server {
        listen 80;
        listen [::]:80;
        server_name domainanda.com www.domainanda.com;
        
        # Keamanan akses: Hanya izinkan dari localhost (Cloudflare Tunnel daemon)
        allow 127.0.0.1;
        allow ::1;
        deny all;

        # --- SECURITY HEADERS ---
        # Catatan: Ubah "SAMEORIGIN" jika kamu ingin embed Status Page Kuma di iframe website lain
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

        # --- OPTIMASI KINERJA: Gzip Compression ---
        gzip on;
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

        # --- REVERSE PROXY UNTUK UPTIME KUMA ---
        location / {
                # burst=50 memberi ruang untuk lonjakan request awal saat dashboard dibuka
                # nodelay memastikan request tidak tertahan (laggy)
                limit_req zone=kuma_limit burst=50 nodelay;
                
                # Arahkan ke port Uptime Kuma (Ubah jika port berbeda)
                proxy_pass http://127.0.0.1:3001;
                
                # Header Proxy Standar
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                # WAJIB: Header WebSocket agar dashboard realtime Kuma tidak terputus
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";

                # Menonaktifkan proxy buffering untuk koneksi realtime yang lebih lancar
                proxy_buffering off;
        }

        # --- HARDENING ---
        # Blokir Akses ke File/Folder Tersembunyi (.env, .git)
        location ~ /\.(?!well-known).* {
                deny all;
                access_log off;
                log_not_found off;
        }
}
```
