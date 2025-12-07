# 🌐 NGINX Proxy Manager Setup Guide for Proxmox (LXC Container)

This guide provides a detailed, step-by-step process for deploying **NGINX Proxy Manager (NPM)** in a Proxmox LXC container using Docker and Docker Compose. This setup is ideal for managing SSL certificates and reverse proxying services across your home network.

## 🚀 Quick Start (Basic Access)

For a basic setup and access to the admin panel:

1.  Create the container with the recommended **Container Specifications**.
2.  Follow **STEPS 1–6**.
3.  Access the admin interface via `http://YOUR_IP_ADDRESS:81`.
4.  Change the default credentials immediately.

---

## ⚙️ Container Specifications

| Component | Recommendation | Notes |
| :--- | :--- | :--- |
| **General** | Hostname | Call it however you see fit (e.g. NGINX-Server). |
| **General** | ✓ Enable Unpriviliged | Recommended. |
| **General** | ✓ Enable Nesting | Required. |
| **Template** | Debian 12 | Recommended. |
| **Disk** | 4 GB | Recommended for OS and NPM configuration files. |
| **CPU** | 1 core | Sufficient for handling proxy requests. |
| **Memory** | 1 GB (1024MB) | Recommended. |
| **Network** | `vmbr0` | Use a static IP or DHCP reservation. (e.g. 192.168.20.x/24) |
| **DNS** | 8.8.8.8 or 1.1.1.1 | Recommended. |

---

## 🖥️ STEP 1 - Initial Container Setup

**Perform these actions from inside the container console.**

```bash
apt update && apt upgrade -y
apt install -y curl wget gnupg2

# Optional: Set timezone for SSL certificates
# timedatectl set-timezone YOUR_TIMEZONE
````
## ⚙️ STEP 2 - Set Static IP Address (Using netplan) ##

This configuration uses ```netplan``` for static IP assignment. Adjust the IP and gateway details for your network.
```bash
# Check your network interface
ip a

# Edit network configuration
nano /etc/netplan/00-installer-config.yaml
```

Add or replace the configuration with your network details:
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:  # Replace with your interface name
      dhcp4: no
      addresses:
        - YOUR_IP_ADDRESS/24  # Replace with your desired IP
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
      routes:
        - to: 0.0.0.0/0
          via: YOUR_GATEWAY_IP  # Replace with your gateway
          on-link: true
```
Apply the changes:

Ctrl + o → Enter → Ctrl + x

Apply network configuration:
```Bash
netplan generate
netplan apply
systemctl restart systemd-networkd
```

## 🐳 STEP 3 - Install Docker & Docker Compose ## 
NPM is deployed via Docker Compose to simplify dependencies and setup.
```bash
# Install Docker
apt install -y docker.io
```
```bash
# Install Docker Compose
wget [https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64](https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64) -O /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
````

**Verify installations**
```bash
docker --version
docker-compose --version
```

**Enable and start Docker**
```bash
systemctl enable docker
systemctl start docker
````

**Optional: Add current user to docker group**
```bash
usermod -aG docker $USER
```

## 🚀 STEP 4 - Install NGINX Proxy Manager ##
We will use ```docker-compose``` to deploy both the NGINX Proxy Manager app and its MariaDB database.
```bash
# Create directory for NPM
mkdir -p /opt/nginx-proxy-manager
cd /opt/nginx-proxy-manager
```
Create docker-compose.yml
```bash
nano docker-compose.yml
```
Add the following YAML configuration:
```bash
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'   # HTTP traffic
      - '81:81'   # Admin interface
      - '443:443' # HTTPS traffic
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: 'jc21/mariadb-aria:latest'
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./mysql:/var/lib/mysql
```
Ctrl + o → Enter → Ctrl + x

## 🔒 STEP 5 - Firewall Setup (UFW) ##
Install and configure UFW to allow access on the proxy and admin ports.
```bash
apt install ufw -y

# Allow necessary ports
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
ufw allow 81/tcp comment 'NPM Admin'

ufw enable
ufw status
```

## ▶️ STEP 6 - Start NGINX Proxy Manager ##
Start the containers using the ```docker-compose.yml``` file.

Start NPM (runs in detached mode -d)
```bash
docker-compose up -d
```
Verify containers are running
(Both 'app' and 'db' should show 'Up')
```bash
docker-compose ps
```
## 💻 STEP 7 - Access NGINX Proxy Manager ##
Access the admin interface via your browser:
- URL: ```http://YOUR_IP_ADDRESS:81```
- Default Email: admin@example.com
- Default Password: changeme

**FIRST LOGIN ACTION**: Change the default admin email and password **immediately**!

## ✨ NEXT STEPS: Configuration and Usage ##
After this initial setup is complete, you can now set up your own proxy hosts with SSL Certificates to your own liking!

My advice is to use the question mark icons on the different Hosts page options and Certificates page to get a better understanding of what they do and what their purpose is.

## 💡 Troubleshooting Common Issues ##

| Issue | Fix |
| :--- | :--- |
| **Can't access port 81** | Check firewall: ```ufw status```, ensure port 81 is allowed. |
| **Docker containers not starting** | Check logs: ```docker-compose logs app```|
| **SSL certificates failing** | Ensure port 80 is accessible for Let's Encrypt validation (via router and firewall).|
| **Database connection errors** | Restart containers: ```docker-compose down && docker-compose up -d```|

✅ **Verification Checklist**
- [ ] Static IP configured correctly
- [ ] Docker and Docker Compose installed
- [ ] NPM containers running (docker-compose ps)
- [ ] Firewall allows ports 80, 443, 81
- [ ] Can access http://IP:81
- [ ] Changed default credentials
