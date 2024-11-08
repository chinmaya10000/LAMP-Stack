# LAMP Setup with WordPress on AWS VPS (Ubuntu 24.04)

This guide provides a step-by-step setup for a LAMP stack (Linux, Nginx, MySQL, PHP) on an Ubuntu 24.04 VPS using AWS. It includes WordPress installation, SSL/TLS with Let’s Encrypt, and server optimizations for performance.

## Steps

### 1. Server Initialization
- SSH into the VPS
- Update system packages
```
sudo apt-get update -y
```

### 2. Nginx Installation
- Install and start Nginx
```
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```