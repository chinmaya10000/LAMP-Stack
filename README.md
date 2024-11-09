# LEMP-Stack Setup with WordPress on AWS VPS (Ubuntu 22.04)

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

### 3. Install MySql
- Install MySQL, secure the installation.
```
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

### 4. PHP Installation
- Install PHP and extensions required for WordPress
```
sudo apt install php8.1-fpm php-mysql
```

### 5. Configure Nginx to Use the PHP Processor
- Create the root web directory
```
sudo mkdir /var/www/lemp-stack
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
server {
    listen 80;
    server_name lamp-stack.ddns.net www.lamp-stack.ddns.net;
    root /var/www/lemp-stack;

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
- Activate your configuration by linking to the configuration file from Nginx’s `sites-enabled` directory:
```
sudo ln -s /etc/nginx/sites-available/lemp-stack /etc/nginx/sites-enabled/
```
- Unlink the default configuration file from the `/sites-enabled/` directory:
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
sudo vim /var/www/lemp-stack/info.php
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

## 6. Create a dedicated user and database for WordPress
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

### 7. Now we can create the PHP script that will connect to MySQL and query for our content. 
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

### 9. WordPress Setup
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

### 10. SSL/TLS Configuration
- Install Certbot
- Obtain SSL certificate from Let’s Encrypt and configure Nginx
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com
```

### 11. Performance Optimization
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

### 12.Test
```
https://lamp-stack.ddns.net
```

### Additional Security and Best Practices
- Use strong passwords for all services
- Grant minimal privileges for the WordPress MySQL user
- Keep software up-to-date