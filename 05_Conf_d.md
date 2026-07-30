# Hardening V.1

```nginx
# Sembunyikan Versi Nginx
server_tokens off;

# Batasi ukuran body request (misal: upload file maks 10MB)
client_max_body_size 10M;

# Batasi ukuran header request
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;

# Membuat zona bernama 'one' dengan alokasi memori 10MB, maks 10 request/detik per IP
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

```
