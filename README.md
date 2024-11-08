# LAMP Setup with WordPress on AWS VPS (Ubuntu 24.04)

This guide provides a step-by-step setup for a LAMP stack (Linux, Nginx, MySQL, PHP) on an Ubuntu 24.04 VPS using AWS. It includes WordPress installation, SSL/TLS with Let’s Encrypt, and server optimizations for performance.

## Steps

### 1. Server Initialization
- SSH into the VPS
- Update system packages
```
sudo apt-get update && sudo apt-get upgrade -y
```

### 2. Nginx Installation
- Install and start Nginx
```
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 3. Deploy and Configure Database
- Install MySQL, secure the installation.
```
sudo apt install mysql-server -y
sudo mysql_secure_installation
```
- Create a dedicated user and database for WordPress
```
sudo mysql -u root
CREATE DATABASE wordpress_db;
CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 4. PHP Installation
- Install PHP and extensions required for WordPress
```
sudo apt install php-fpm php-mysql -y
```

### 5. WordPress Setup
- Download and extract WordPress into the Nginx web root
```
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
```
- Set Permissions
```
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo find /var/www/html/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/html/wordpress -type f -exec chmod 644 {} \;
```

### 6. Configure Nginx for WordPress
- Configure Nginx for WordPress
```
sudo nano /etc/nginx/sites-available/wordpress
```
- Add the following configuration
```
server {
    listen 80;
    server_name your_domain.com;
    root /var/www/html/wordpress;

    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
