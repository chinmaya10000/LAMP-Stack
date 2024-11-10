# Automated WordPress Deployment with Nginx, LEMP Stack, and GitHubActions
To set up an automated deployment process for a WordPress website using Nginx as the web server, LEMP (Linux, Nginx, MySQL, PHP) stack, and GitHub Actions as the CI/CD automation tool.This GitHub Actions workflow automates the continuous integration and deployment (CI/CD) process. while following best practices for security, performance, and automation.

## Steps

### 1. Server Provisioning
- I am using AWS as the Cloud Provider
- Provision an EC2 instance on AWS with a secure Linux distribution (e.g., Ubuntu 24.04).
- Configure security groups to allow necessary incoming traffic and restrict unnecessary access.
- Generate and configure SSH key pairs for secure remote acces.

### 2. Nginx, MySQL/MariaDB, and PHP Setup
- Install and configure Nginx as the web server.
```
sudo apt-get update -y
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```
- If firewall is active/going to enable we need to allow `NGINX` with firewall we using here `ufw`.
```
sudo ufw app list
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'
sudo ufw allow ssh
sudo ufw status
sudo ufw enable
sudo systemctl restart nginx
```
- Visit public ip to see nginx welcome page :
```
http://your_serverip
```
- Install and setup PHP :
```
sudo add-apt-repository ppa:ondrej/php
sudo apt update && apt install php8.2-fpm php8.2-common php8.2-mysql php8.2-xml php8.2-xmlrpc php8.2-curl php8.2-gd php8.2-imagick php8.2-cli php8.2-dev php8.2-imap php8.2-mbstring php8.2-soap php8.2-zip php8.2-bcmath -y
sudo sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/g' /etc/php/8.2/fpm/php.ini
sudo service php8.2-fpm restart
```
```
sed -i 's/memory_limit = 128M/memory_limit = 512M/g' /etc/php/8.2/fpm/php.ini
sed -i 's/post_max_size = 8M/post_max_size = 128M/g' /etc/php/8.2/fpm/php.ini
sed -i 's/max_file_uploads = 20/max_file_uploads = 30/g' /etc/php/8.2/fpm/php.ini
sed -i 's/max_execution_time = 30/max_execution_time = 900/g' /etc/php/8.2/fpm/php.ini
sed -i 's/max_input_time = 60/max_input_time = 3000/g' /etc/php/8.2/fpm/php.ini
sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 128M/g' /etc/php/8.2/fpm/php.ini
service php8.2-fpm restart
```

### 3. Configure Nginx
- Remove directory 
```
rm /etc/nginx/sites-available/default
rm /etc/nginx/sites-enabled/default
```
- Create the root web directory
```
sudo mkdir /var/www/lemp-stack/
cd /var/www/lemp-stack/
mkdir mkdir cache
sudo mkdir wordpress
cd public_html
```
- Assign ownership of the directory with the `$USER` environment variable, which will reference your current system user
```
sudo chown -R $USER:$USER /var/www/lemp-stack
```
- open a new configuration file in Nginx’s `sites-available` directory using your preferred command-line editor. Here, i’ll use `vim`:
```
sudo vim /etc/nginx/sites-available/lemp-stack
```
- Add the following configuration
```
fastcgi_cache_path /var/www/lemp-stack/cache levels=1:2 keys_zone=example.com:100m inactive=60m;

server {
  listen 80;

  root /var/www/lemp-stack/wordpress/public_html;
  index index.php index.html index.htm;

  server_name  lamp-stack.ddns.net www.lamp-stack.ddns.net;
  client_max_body_size 0;

 error_page 401 403 404 /404.html;
  error_page 500 502 503 504 /50x.html;


###############
set $skip_cache 0;

# POST requests and urls with a query string should always go to PHP
if ($request_method = POST) {
     set $skip_cache 1;
}

#if ($query_string != "")

if ($query_string = "unapproved*") {
     set $skip_cache 1;
}

if ( $cookie_woocommerce_items_in_cart = "1" ){
     set $skip_cache 1;
}

# Don't cache URIs containing the following segments
if ($request_uri ~* "/wp-admin/|/xmlrpc.php|/wp-.*.php|index.php|sitemap(_index)?.xml")  {
     set $skip_cache 1;
}
if ($request_uri ~* "/(cart|checkout|my-account)/*$") {
     set $skip_cache 1;
}

# Don't use the cache for logged-in users or recent commenters
if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_no_cache|wordpress_logged_in") {
     set $skip_cache 1;
}

###############

location / {
    try_files $uri $uri/ /index.php?q=$uri&$args;
  }

  location ~* \.php$ {
    if ($uri !~ "^/uploads/") {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        }
    include         fastcgi_params;
    fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
    fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
  
################
fastcgi_cache_bypass $skip_cache;
fastcgi_no_cache $skip_cache;
fastcgi_cache example.com;
fastcgi_cache_valid 200 301 24h;
###############


 }
location = /favicon.ico { 
    log_not_found off;
    access_log off;
  }
  
  location = /robots.txt {
    log_not_found off;
    access_log off;
    allow all; 
  }
  
  location ~* .(css|gif|ico|jpeg|jpg|js|png)$ {
    expires 30d;
    log_not_found off;
  }
}
```
### 4. Enable Gzip
- Enable Gzip compression in Nginx
```
sudo vim /etc/nginx/nginx.conf 
```
- Include the following content in this file:
```
# Enable Gzip compression.
gzip on;

# Disable Gzip on IE6.
gzip_disable "msie6";

# Allow proxies to cache both compressed and regular version of file.
# Avoids clients that don't support Gzip outputting gibberish.
gzip_vary on;

# Compress data, even when the client connects through a proxy.
gzip_proxied any;

# The level of compression to apply to files. A higher compression level increases
# CPU usage. Level 5 is a happy medium resulting in roughly 75% compression.
gzip_comp_level 5;

# Compress the following MIME types.
gzip_types
	application/atom+xml
	application/javascript
	application/json
	application/ld+json
	application/manifest+json
	application/rss+xml
	application/vnd.geo+json
	application/vnd.ms-fontobject
	application/x-font-ttf
	application/x-web-app-manifest+json
	application/xhtml+xml
	application/xml
application/x-javascript


	font/opentype
font/ttf
font/eot
font/otf


	image/bmp
	image/svg+xml
	image/x-icon
	text/cache-manifest
	text/css
	text/plain
	text/vcard
	text/vnd.rim.location.xloc
	text/vtt
	text/x-component
	text/x-cross-domain-policy;


##
# Cache Settings
##
fastcgi_cache_key "$scheme$request_method$host$request_uri";
add_header Fastcgi-Cache $upstream_cache_status;
fastcgi_cache_use_stale error timeout invalid_header http_500;
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
```
- Activate your configuration by linking to the configuration file from Nginx’s `sites-enabled` directory:
```
sudo ln -s /etc/nginx/sites-available/lemp-stack /etc/nginx/sites-enabled/
```
- To ensure there are no syntax errors in the configuration, run:
```
sudo nginx -t
```
- Reload Nginx to Apply Changes:
```
sudo systemctl reload nginx
```
- Our new website is now active, but the web root `/var/www/lemp-stack/wordpress/public_html` is still empty. Create an `index.html` file in that location so that we can test that our new server block works as expected:
```
sudo vim /var/www/lemp-stack/wordpress/public_html/index.html
```
- Include the following content in this file:
```
<html>
  <head>
    <title>LEMP-Stack website</title>
  </head>
  <body>
    <h1>Hello World!</h1>

    <p>This is the landing page of <strong>LEMP Stack</strong>.</p>
  </body>
</html>
```
- test
```
http://lamp-stack.ddns.net
```
- We can do this by creating a test PHP file in your document root.
```
sudo vim /var/www/lemp-stack/wordpress/public_html
```
- Add the following lines into the new file.
```
<?php
phpinfo();
```
- Test 
```
http://lamp-stack.ddns.net/info.php
```

## 5. Install and configure mysqldb
- Install mysql 
```
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

- Configure Database
```
sudo mysql -u root
CREATE DATABASE wordpress_db;
CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY 'Lemp@321';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 6. WordPress Setup
- Download and extract WordPress into the Nginx web root
```
cd /var/www/lemp-stack/wordpress/public_html
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
sudo mv -v wordpress/* /var/www/lemp-stack/wordpress/public_html
rmdir wordpress
rm latest.tar.gz
```
- Set Permissions
```
sudo chown -R www-data:www-data /var/www/
sudo find /var/www/ -type d -exec chmod 755 {} \;
sudo find /var/www/ -type f -exec chmod 644 {} \;
```
- Restart the server:
```
sudo systemctl restart nginx
sudo systemctl restart php8.2-fpm.service
sudo systemctl restart mysql
```

### 7. SSL/TLS Configuration
- Install Certbot
- Obtain SSL certificate from Let’s Encrypt and configure Nginx
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com
```


### 8. open a browser and go to:
```
https://lamp-stack.ddns.net
```

# WordPress EC2 Deployment with GitHub Actions
This repository contains automation for deploying a WordPress website using LEMP stack (Linux, Nginx, MySQL, PHP) on AWS EC2.

# WordPress LEMP Stack Deployment

This repository contains automation for deploying a WordPress website using LEMP stack (Linux, Nginx, MySQL, PHP) on AWS EC2.

## Prerequisites

- AWS EC2 instance running Ubuntu
- GitHub account
- MySQL database (setup instructions below)
- Domain name 

## Initial Server Setup

Before running the GitHub Actions workflow, you need to set up your database. SSH into your EC2 instance and follow these steps:

1. Install MySQL:
 ```bash
   sudo apt update
   sudo apt install mysql-server
3. Secure MySQL installation:
```bash
   sudo mysql_secure_installation
4. Create WordPress database and user:
```bash
   sudo mysql
   CREATE DATABASE wordpress_db;
   CREATE USER 'wordpress_user'@'localhost' IDENTIFIED 'Lemp@321';
   GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;


### 2. GitHub Actions Setup
- Go to your repository Settings > Secrets and Variables > Actions
- Add the following secrets:
```bash
EC2_SSH_KEY: Your EC2 instance's private SSH key
EC2_IP: Your EC2 instance's public IP address
DB_NAME: WordPress database name (e.g., wordpress)
DB_USER: Database username (e.g., wordpressuser)
DB_PASSWORD: Database password (the one you set during database creation)
DB_HOST: Database host (use 'localhost' if database is on the same server)


And here's the updated workflow file (.github/workflows/main.yml):

```yaml
name: Deploy WordPress LEMP Stack

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.EC2_SSH_KEY }}

      - name: Add EC2 to known hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.EC2_IP }} >> ~/.ssh/known_hosts

      - name: Deploy to EC2
        env:
          EC2_IP: ${{ secrets.EC2_IP }}
          DB_NAME: ${{ secrets.DB_NAME }}
          DB_USER: ${{ secrets.DB_USER }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          DB_HOST: ${{ secrets.DB_HOST }}
        run: |
          ssh ubuntu@$EC2_IP << 'ENDSSH'
            # Update system packages
            sudo apt update
            sudo apt install -y nginx php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip

            # Create web root directory if it doesn't exist
            sudo mkdir -p /var/www/lemp-stack/wordpress/public_html

            # Download and setup WordPress
            cd /var/www/lemp-stack/wordpress/public_html
            if [ ! -d wordpress ]; then
              sudo wget https://wordpress.org/latest.tar.gz
              sudo tar -xzf latest.tar.gz
              sudo rm latest.tar.gz
            fi

            # Configure wp-config.php
            cd wordpress
            if [ ! -f wp-config.php ]; then
              sudo cp wp-config-sample.php wp-config.php
              sudo sed -i "s/database_name_here/${DB_NAME}/" wp-config.php
              sudo sed -i "s/username_here/${DB_USER}/" wp-config.php
              sudo sed -i "s/password_here/${DB_PASSWORD}/" wp-config.php
              sudo sed -i "s/localhost/${DB_HOST}/" wp-config.php
              
              # Add WordPress salts
              SALTS=$(curl -s https://api.wordpress.org/secret-key/1.1/salt/)
              sudo sed -i "/#@-/,/#@+/c\\$SALTS" wp-config.php

              # Add extra security configurations
              echo "define('DISALLOW_FILE_EDIT', true);" >> wp-config.php
              echo "define('WP_AUTO_UPDATE_CORE', true);" >> wp-config.php
            fi

            # Set correct permissions
            sudo chown -R www-data:www-data /var/www/
            sudo find /var/www/ -type d -exec chmod 755 {} \;
            sudo find /var/www/ -type f -exec chmod 644 {} \;

            # Configure Nginx
            sudo tee /etc/nginx/sites-available/lemp-stack << 'EOF'
            server {
                listen 80;
                root /var/www/lemp-stack/wordpress/public_html;
                index index.php index.html index.htm;
                server_name lamp-stack.ddns.net;

                # Security headers
                add_header X-Frame-Options "SAMEORIGIN" always;
                add_header X-XSS-Protection "1; mode=block" always;
                add_header X-Content-Type-Options "nosniff" always;

                location / {
                    try_files $uri $uri/ /index.php?$args;
                }

                location ~ \.php$ {
                    include snippets/fastcgi-php.conf;
                    fastcgi_pass unix:/var/run/php/php-fpm.sock;
                    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                    include fastcgi_params;
                }

                location ~ /\.ht {
                    deny all;
                }

                # Media: images, icons, video, audio, HTC
                location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|mp4|ogg|ogv|webm|htc)$ {
                    expires 1M;
                    access_log off;
                    add_header Cache-Control "public";
                }

                # CSS and Javascript
                location ~* \.(?:css|js)$ {
                    expires 1y;
                    access_log off;
                    add_header Cache-Control "public";
                }
            }
            EOF

            # Enable the site and remove default
            sudo rm -f /etc/nginx/sites-enabled/default
            sudo ln -sf /etc/nginx/sites-available/lemp-stack /etc/nginx/sites-enabled/

            # Test Nginx configuration
            sudo nginx -t

            # Restart services
            sudo systemctl restart php-fpm
            sudo systemctl restart nginx
          ENDSSH

      - name: Verify deployment
        run: |
          curl -I http://${{ secrets.EC2_IP }}


### Additional Security and Best Practices
- Use strong passwords for all services
- Grant minimal privileges for the WordPress MySQL user
- Keep software up-to-date