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
- Install and configure mysql server :
```
sudo apt install mysql-server 
sudo mysql_secure_installation
sudo systemctl restart mysql
sudo systemctl enable mysql
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
fastcgi_cache_path /var/www/example.com/cache levels=1:2 keys_zone=example.com:100m inactive=60m;

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
- Our new website is now active, but the web root `/var/www/lamp-stack` is still empty. Create an `index.html` file in that location so that we can test that our new server block works as expected:
```
sudo vim /var/www/lemp-stack/index.html
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

## 4. Create a dedicated user and database for WordPress
- Configure Database
```
sudo mysql -u root
CREATE DATABASE wordpress_db;
CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY 'Lemp@321';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
- We can test if the new user has the proper permissions by logging in to the MySQL console again
```
mysql -u wordpress_user -p
```
- Next, we’ll create a test table named `todo_list`. From the `MySQL` console
```
CREATE TABLE wordpress_db.todo_list (
	item_id INT AUTO_INCREMENT,
	content VARCHAR(255),
	PRIMARY KEY(item_id)
);
```
- Insert a few rows of content in the test table.
```
INSERT INTO wordpress_db.todo_list (content) VALUES ("My first important item");
```
- To confirm that the data was successfully saved to your table
```
SELECT * FROM wordpress_db.todo_list;
exit
```

### 5. Now we can create the PHP script that will connect to MySQL and query for our content. 
- Create a new PHP file in our custom web root directory using your preferred editor. i’ll use vim for that
```
sudo vim /var/www/lemp-stack/todo_list.php
```
- The following PHP script connects to the MySQL database and queries for the content of the `todo_list` table, exhibiting the results in a list. If there’s a problem with the database connection, it will throw an exception. Add the following content to your `todo_list.php` script:
```
<?php
$user = "wordpress_user";
$password = "Lemp@321";
$database = "wordpress_db";
$table = "todo_list";

try {
  $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
  echo "<h2>TODO</h2><ol>"; 
  foreach($db->query("SELECT content FROM $table") as $row) {
    echo "<li>" . $row['content'] . "</li>";
  }
  echo "</ol>";
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}
```

### 6. WordPress Setup
- Download and extract WordPress into the Nginx web root
```
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
sudo mv wordpress/* /var/www/lamp-stack
```
- Set Permissions
```
ps -ef | grep nginx
sudo chown -R www-data:www-data /var/www/lemp-stack/
```

### 7. Performance Optimization
- Enable Gzip compression in Nginx
- Configure Nginx for caching static files
```
sudo nano /etc/nginx/nginx.conf
```
- Add the following lines in the http block:
```
server {
    listen 80;
    server_name lamp-stack.ddns.net;
    root /var/www/lamp-stack;
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
- Restart the server:
```
sudo systemctl restart nginx
```

### 8. SSL/TLS Configuration
- Install Certbot
- Obtain SSL certificate from Let’s Encrypt and configure Nginx
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com
```


### 9.Test
```
https://lamp-stack.ddns.net
```

# GitHub Actions Workflow
This workflow is designed to be super-easy to integrate into your own WordPress project or any other PHP-based project. It’s designed to ensure code quality and deploy changes to both staging and production environments from the master branch.

## Steps

### 1. Create a GitHub repository
- Go to repository >> Settings >>Actions secrets and variables >> add below secrets
```
HOST: The hostname or IP address of the production server.
USERNANE: ubuntu
SSH_PRIVATE_KEY : private key of your server
```

### 2. GitHub Actions Workflow
- clikc on actions button on our git repo, click on set up a workflow yourself button. create a `main.yaml` file
- this deployment file i am trying to build themes by installing dependacies and building then after that i am syncing the directory with rsync utility to sych with my wordpress site location files and send securly.
```
name: Push Code to EC2

on:
 push:
   branches:
     - master
jobs:
 push_to_ec2:
   runs-on: ubuntu-latest
   steps:
     - name: Checkout Repository
       uses: actions/checkout@v2
     - name: Set up SSH
       uses: webfactory/ssh-agent@v0.8.0
       with:
         ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
     - name: Install dependencies
       run: |
         # Add your commands to install project dependencies
         # For example, if you are using Node.js:
         # npm install
     - name: Build the project
       run: |
         # Add your commands to build the project
         # For example, if you are using Node.js:
         # npm run build
     - name: Install rsync
       run: sudo apt-get update && sudo apt-get install -y rsync
     - name: Add EC2 instance host key to known hosts
       run: |
         ssh-keyscan -H $HOST >> $HOME/.ssh/known_hosts
       env:
         HOST: ${{ secrets.HOST }}
     - name: Push code to EC2
       run: |
         # Set IP address and directory paths
         EC2_IP="$HOST"
         SOURCE_DIR="$GITHUB_WORKSPACE/"
         DEST_DIR="/var/www/lemp-stack/wordpress/"
         # Use rsync with SSH key authentication and compression
         rsync -avz --delete-after --exclude='.git' -e "ssh -i $SSH_AUTH_SOCK" "$SOURCE_DIR" "${{ secrets.USERNAME }}@$EC2_IP:$DEST_DIR"
       env:
         HOST: ${{ secrets.HOST }}
         USERNAME: ${{ secrets.USERNAME }}
         SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Additional Security and Best Practices
- Use strong passwords for all services
- Grant minimal privileges for the WordPress MySQL user
- Keep software up-to-date