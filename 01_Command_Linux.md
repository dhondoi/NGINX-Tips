Perintah,Fungsi

`sudo systemctl start nginx`, Menjalankan Nginx

`sudo systemctl stop nginx`, Menghentikan Nginx

`sudo systemctl restart nginx`, Memuat ulang Nginx secara keseluruhan

`sudo systemctl reload nginx`, Memuat ulang konfigurasi tanpa downtime

`sudo nginx -t` ,Menguji apakah ada kesalahan sintaks pada konfigurasi

shorthand
```bash
sudo nginx -t && sudo systemctl reload nginx && sudo systemctl status nginx
```
buat symlink
```bash
sudo ln -s /etc/nginx/sites-available/domainanda.com /etc/nginx/sites-enabled/
```
