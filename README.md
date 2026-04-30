## Frappe & ERPNext v16 Installation Ubuntu

## Prerequisites

1. Ubuntu 25.04 
2. python3.14
3. pip 25.3+
4. node 24
5. yarn 1.22+
6. redis 6+
7. mariadb 11.8



## Step 1: Update

```bash
sudo apt update -y && sudo apt upgrade -y

## Step 2: Create a Frappe Userr
It’s recommended not to run Bench as root.

```bash
sudo adduser frappe_user
sudo usermod -aG sudo frappe_user
su frappe_user

cd /home/frappe_user

You can also use your existing user; just replace frappe_user with your username.

##Step 3: Install Dependencies

```bash
sudo apt install -y git python3-dev python3-setuptools python3-pip \
python3.14-venv software-properties-common curl build-essential \
pkg-config libmysqlclient-dev python3.14-dev

##Step 4: Install MariaDB
```bash
sudo apt install -y mariadb-server
sudo mariadb-secure-installation

During setup, recommended options:

    Enter current password for root: ENTER

    Switch to unix_socket authentication? Y
    Change root password? Y
    Remove anonymous users? Y
    Disallow root login remotely? N
    Remove test database? Y
    Reload privilege tables? Y

##Step 5: Configure MariaDB
```bash
sudo nano /etc/mysql/my.cnf

Add under [mysqld]:
```bash
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
[mysql]
default-character-set = utf8mb4

Restart MariaDB:
```bash
sudo service mysql restart

##Step 6: Install Redis
```bash
sudo apt install -y redis-server

##Step 7: Install Node.js 24 & Yarn (Frappe v16 requirement)
#Remove old Node.js (if any)
```bash
sudo apt remove -y nodejs npm
sudo apt autoremove -y

#Install prerequisites
```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg build-essential

#Install Node.js 24
```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

Verify:
```bash
node -v  # should show v24.x.x
npm -v

#Install Yarn globally
```bash
sudo npm install -g yarn
yarn -v  # should show 1.22.x

##Step 8: Install PDF Rendering Tools
```bash
sudo apt install -y libcairo2 libpango-1.0-0 libpangocairo-1.0-0 \
libgdk-pixbuf-2.0-0 libffi-dev shared-mime-info

##Step 9: Install Bench via pipx
```bash
sudo apt install -y pipx
pipx ensurepath
source ~/.bashrc
pipx install frappe-bench
pipx install uv
pipx install honcho

##Step 10: Initialize Frappe Bench
Use Python 3.14 and Frappe v16 RC:
```bash
bench init frappe-bench \
  --frappe-branch v16.0.0 \
  --python python3.14

#This resolves Python & Node/Yarn dependencies correctly.

##Step 11: Create a New Site
```bash
cd frappe-bench
bench new-site yoursite_name
  
  Enter MySQL root user: root (or your MySQL user)
  MySQL root password: your MySQL password
  Set Frappe Administrator password when prompted

##Step 12: Get ERPNext & HRMS Apps (v16 RC)
#ERPNext & HRMS
```bash
# ERPNext
bench get-app erpnext --branch v16.0.0
# HRMS
bench get-app hrms --branch v16.0.0

##Step 13: Install Apps on Your Site
```bash
bench --site yoursite_name install-app erpnext
bench --site yoursite_name install-app hrms

#Set current site:
```bash
bench use yoursite_name

##Step 14: Start Development Server or skip and set Production step 15
```bash
bench start

#Open your browser:
```bash
http://127.0.0.1:8000

##Step 15 & Production Setup ()

##Enterprise Production Setup & Security (ERPNext v16 / Ubuntu)

#Frappe Production Setup Guide

#Step 1: Stop Development Mode
Before setting up production, ensure you are not running the site manually. Go to your terminal where bench start is running and press Ctrl + C to stop it. Ensure no Python processes are holding the ports.

##Step 2: System Preparation & Cleaning
Ubuntu often pre-installs Apache, which conflicts with Nginx on Port 80. We remove it to prevent “Address already in use” errors.
```bash
#Remove Apache to prevent conflicts
sudo systemctl disable --now apache2
sudo apt remove apache2 -y
#Install the runtime engines
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y nginx supervisor redis-server fail2ban

##Step 3: Configuration Generation
Run these commands as your bench user. We ask Frappe to generate the correct configuration files for your specific site.
```bash
#Create Nginx routing rules
bench setup nginx 
#Create Supervisor worker instructions
bench setup supervisor

##Step 4: System Linking
We manually plug your generated files into the operating system using Symbolic Links.
```bash
Note: Replace /home/frappe_user in the commands below with your actual home directory path (e.g., /home/frappe_user).

```bash
#Link the Web Config (-sf forces overwrite of old links)
sudo ln -sf /home/frappe_user/frappe-bench/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf  
#Link the Worker Config
sudo ln -sf /home/frappe_user/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf

##Step 5: Nginx Log Format Fix (Ubuntu 25 Config)
Ubuntu’s default Nginx is missing the “main” log format Frappe expects.
  1. Edit the global config:
  ```bash
  sudo nano /etc/nginx/nginx.conf

  2. Scroll to the http { block. Paste the following exactly once above the access_log line:
  Nginx
  ```bash
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

  3. Save (Ctrl+O, Enter) and Exit (Ctrl+X).

  4. Test the syntax:
  ```bash
  sudo nginx -t
  (Proceed only if it says “syntax is ok”)

##Step 6: Permissions & Assets
If permissions are too tight, Nginx cannot read CSS/JS files.
```bash
# Allow Nginx to traverse your home folder
chmod o+x /home/frappe_user
# Compile Production Assets (CSS/JS)
cd ~/frappe-bench 
bench build
# Reset Ownership (Fixes Supervisor "Spawn Errors")
# Replace 'frappe_user:frappe_user' with your actual user:group
sudo chown -R frappe_user:frappe_user /home/frappe_user/frappe-bench

##Step 7: Routing & Activation
We set the default site and remove the Nginx welcome page.
```bash
# Set your site as default (Replace with actual name)
bench use your_site_name
# DELETE DEFAULT NGINX PAGE
sudo rm /etc/nginx/sites-enabled/default 
# Regenerate Nginx config with new default
bench setup nginx 
# Reload Nginx
sudo service nginx reload 
# Restart Supervisor
sudo service supervisor restart
sudo supervisorctl reload

##Step 8: Security (Firewall)
Configure UFW to secure your server while allowing essential traffic.  
```bash
# Install UFW
sudo apt install ufw
# Set Default Rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
# Allow Critical Ports
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
# Enable Firewall
sudo ufw enable

##Step 9: Verification
Check service status (Should all be RUNNING).
```bash
sudo supervisorctl status

#Access your site: Open http://localhost (or your IP address) in your browser.



















































































