# Ubuntu Nginx Setup Documentation

## 1. OS Installation

Install **Ubuntu Server 20.04 LTS**.

### File Partition Scheme

| Mount Point | Size    | File System |
| ----------- | ------- | ----------- |
| /boot/efi   | 1024 MB | EFI System  |
| swap        | 8200 MB | swap        |
| /home       | 50 GB   | ext4        |
| /opt        | 50 GB   | ext4        |
| / (root)    | 350+ GB | ext4        |
| Other disks | any     | ext4        |

---

## 2. Install Required Software

### Update Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Install PHP 7.4 & Extensions

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install -y php7.4 php7.4-fpm php7.4-mysql php7.4-curl php7.4-mbstring php7.4-xml php7.4-bcmath php7.4-zip php7.4-imagick
```

### Install Nginx

```bash
sudo apt install -y nginx
```

### Install MySQL 8.0

```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

### Install phpMyAdmin

```bash
sudo apt install -y phpmyadmin
```

> During installation, select **apache2** (ignore, as we’re using nginx), then configure manually.

```bash
sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin
```

---

## 3. MySQL Configuration

### Grant All Privileges

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'yourpassword' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### Change MySQL Password

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'newpassword';
FLUSH PRIVILEGES;
```

---

## 4. Setup SSL

### Place Certificates

Put the following files in `/etc/nginx/ssl/`:

* `erp.crt` (certificate)
* `erp.key` (private key)
* `ca-bundle.crt` (CA bundle)

### Combine Certificate and CA Bundle

```bash
cd /etc/nginx/ssl
sudo sh -c 'cat erp.crt ca-bundle.crt > ssl-bundle.crt'
```

---

## 5. Nginx Configuration

### Example: `/etc/nginx/conf.d/default.conf`

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name erp.neways3.com www.erp.neways3.com;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    server_name erp.neways3.com www.erp.neways3.com;

    ssl_certificate /etc/nginx/ssl/ssl-bundle.crt;
    ssl_certificate_key /etc/nginx/ssl/erp.key;

    root /home/server_erp/erp.neways3;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location /phpmyadmin {
        alias /var/www/html/phpmyadmin/;
        index index.php index.html;
    }

    location ~ ^/phpmyadmin/(.+\.php)$ {
        alias /var/www/html/phpmyadmin/$1;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /usr/share/phpmyadmin/$1;
    }

    location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
        alias /var/www/html/phpmyadmin/$1;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### Test and Reload

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 6. Enable Services on Boot

```bash
sudo systemctl enable nginx
sudo systemctl enable php7.4-fpm
sudo systemctl enable mysql
```

---

## 7. External Info & Utilities

### Check Hostname in MySQL

```sql
SELECT user, host FROM mysql.user;
```

### Create a New File

```bash
sudo touch filename.txt
```

Or create and open with nano:

```bash
sudo nano newfile.txt
```

### Set File Permissions

```bash
sudo chown youruser:youruser file.txt
sudo chmod 644 file.txt
```

---

## 8. Common Issues

* **SSL Key Mismatch**: Ensure the private key matches the certificate.
* **Imagick Missing**: Install using `sudo apt install php-imagick` and restart PHP.
* **FTP Put Disk Full**: Check disk usage with `df -h`.

---

## 9. Useful Commands

### Restart Services

```bash
sudo systemctl restart nginx
sudo systemctl restart php7.4-fpm
sudo systemctl restart mysql
```

### Check PHP Modules

```bash
php -m | grep imagick
php -m | grep curl
```

### Check Open Ports

```bash
sudo lsof -i -P -n | grep LISTEN
```

### Monitor Logs

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## Done!

You're now running a production-ready Ubuntu server with Nginx, PHP, MySQL, phpMyAdmin, and SSL.
