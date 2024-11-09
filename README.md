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
sudo apt install php-fpm php-mysql php php-curl php-gd php-intl php-zip 
sudo add-apt-repository ppa:ondrej/php
sudo apt-get install php7.4-fpm
sudo systemctl enable php7.4-fpm
sudo systemctl start php7.4-fpm
```

### 3. Configure Nginx
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
    root /var/www/your_domain;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
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
     - main
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