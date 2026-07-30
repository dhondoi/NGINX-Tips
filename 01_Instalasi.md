# Cek Dependency

```cmd
apt depends nginx
```

# Install

```cmd
sudo apt install nginx -y
```

# Verifikasi

```cmd
systemctl status nginx
```

# Uninstall

```cmd
sudo systemctl stop nginx
sudo apt purge nginx nginx-common nginx-core -y
sudo apt autoremove -y
sudo rm -rf /etc/nginx
sudo rm -rf /var/www/html/index.nginx-debian.html
```
