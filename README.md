# LEMP-Stack Setup with WordPress on AWS VPS (Ubuntu 24.04)

This guide provides a step-by-step setup for a LEMP stack (Linux, Nginx, MySQL, PHP) on an Ubuntu 22.04 VPS using AWS. It includes WordPress installation, SSL/TLS with Let’s Encrypt, and server optimizations for performance.

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
sudo apt install php8.1-fpm php-mysql
```

### 5. Configure Nginx to Use the PHP Processor
- Create the root web directory
```
sudo mkdir /var/www/lamp-stack
```
- Assign ownership of the directory with the $USER environment variable, which will reference your current system user
```
sudo chown -R $USER:$USER /var/www/lamp-stack
```
- open a new configuration file in Nginx’s sites-available directory using your preferred command-line editor. Here, i’ll use vim:
```
sudo vim /etc/nginx/sites-available/lamp-stack
```
- Add the following configuration
```
server {
    listen 80;
    server_name your_domain www.your_domain;
    root /var/www/lamp-stack;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

}
```
- Activate your configuration by linking to the configuration file from Nginx’s sites-enabled directory:
```
sudo ln -s /etc/nginx/sites-available/lamp-stack /etc/nginx/sites-enabled/
```
- Unlink the default configuration file from the /sites-enabled/ directory:
```
sudo unlink /etc/nginx/sites-enabled/default
```
- To ensure there are no syntax errors in the configuration, run:
```
sudo nginx -t
```
- Reload Nginx to Apply Changes:
```
sudo systemctl reload nginx
```



### 6. WordPress Setup
- Download and extract WordPress into the Nginx web root
```
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
sudo mv wordpress/* /var/www/devops
```
- Set Permissions
```
sudo chown -R www-data:www-data /var/www/devops/
sudo find /var/www/devops -type d -exec chmod 755 {} \;
sudo find /var/www/devops -type f -exec chmod 644 {} \;
```

### 8. SSL/TLS Configuration
- Install Certbot
- Obtain SSL certificate from Let’s Encrypt and configure Nginx
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com
```

### 9. Performance Optimization
- Enable Gzip compression in Nginx
- Configure Nginx for caching static files
```
sudo nano /etc/nginx/nginx.conf
```
- Add the following lines in the http block:
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

    # Static file caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        access_log off;
    }

    # Enable Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1000;
    gzip_proxied any;
    gzip_vary on;
}
```

``` 

### 12.Test
```
https://your_domain.com
```

### Additional Security and Best Practices
- Use strong passwords for all services
- Grant minimal privileges for the WordPress MySQL user
- Keep software up-to-date