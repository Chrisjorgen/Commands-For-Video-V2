# 🐦 Pelican Panel Setup Guide

A comprehensive guide for setting up Pelican Panel with Oracle Cloud Infrastructure and Cloudflare.

## 📋 Prerequisites

- A server to install the panel on
- A domain registered in Cloudflare
- An Oracle Account (preferably Pay as You Go)

> **Note**: Some commands may require `sudo`. Use it whenever applicable to your situation.

## 🚀 Installation Steps

### 1. Get Ubuntu Server ISO

Download from: [Ubuntu Server Download Page](https://ubuntu.com/download/server)

### 2. Create VM and Install Ubuntu Server

*Steps vary based on your setup (Guide uses Proxmox)*

### 3. Update and Setup VM

#### 3.1 Update VM
```bash
sudo apt upgrade -y && sudo apt update -y
```

#### 3.2 Install Panel Dependencies
```bash
sudo apt -y install php8.3 php8.3-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip,intl,sqlite3} mariadb-server nginx tar unzip git netcat-openbsd resolvconf wireguard
```

### 4. Setup Oracle VPS

#### Configuration
- **Image**: Canonical Ubuntu 24.05
- **Shape**: Ampere VM.Standard.A1.Flex
  - 2 cores
  - 12GB Memory

#### 4.1 Initial Setup

##### 4.1.1 Update VPS
```bash
sudo su
apt update -y && apt upgrade -y
```

##### 4.1.2 Install and Configure UFW
```bash
apt install ufw -y
ufw allow 22 && ufw allow 25565 && ufw allow 58000
ufw enable
```

##### 4.1.3 Install and Setup Wireguard
```bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

##### 4.1.4 Install and Setup Caddy-l4

###### Install xcaddy Dependencies
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Add GPG key
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg 

# Add repository
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list 

# Install xcaddy
apt update -y
apt install -y xcaddy
```

###### Install Go
```bash
wget https://go.dev/dl/go1.23.3.linux-arm64.tar.gz
tar -C /usr/local -xzf go1.23.3.linux-arm64.tar.gz  

# Add Go to PATH
export PATH=$PATH:/usr/local/go/bin
source ~/.profile

# Verify installation
go version
```

###### Build and Configure Caddy-l4
```bash
xcaddy build --with github.com/mholt/caddy-l4
mv caddy /usr/local/bin/
caddy -v 

# Setup configuration directory
mkdir -p /etc/caddy
```

Create Caddyfile (`/etc/caddy/Caddyfile`):
```
{
    layer4 {
        :25565 {
            route {
                proxy {
                    upstream 10.66.66.2:25565
                }
            }
        }
    }
}
```

Create systemd service (`/etc/systemd/system/caddy.service`):
```ini
[Unit]
Description=Caddy web server
Documentation=https://caddyserver.com/docs
After=network.target

[Service]
ExecStart=/usr/local/bin/caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
ExecReload=/usr/local/bin/caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
ExecStop=/usr/local/bin/caddy stop
Restart=on-failure
LimitNOFILE=1048576
LimitNPROC=512

[Install]
WantedBy=multi-user.target
```

Enable and start Caddy:
```bash
systemctl daemon-reload
systemctl enable caddy
systemctl start caddy
systemctl status caddy
```

### 5. Setup Pelican VM

#### 5.1 Connect to Wireguard Tunnel

Create configuration file (`/etc/wireguard/wg0.conf`) and add your VPS configuration.

Create systemd service (`/etc/systemd/system/wg-quick@.service`):
```ini
[Unit]
Description=WireGuard via wg-quick(8) for %I
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/wg-quick up %i
ExecStartPost=/usr/bin/curl ifconfig.io
ExecStop=/usr/bin/wg-quick down %i
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start Wireguard:
```bash
sudo systemctl daemon-reload
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
sudo systemctl status wg-quick@wg0.service
```

#### 5.2 Setup SSL Certificates
```bash
sudo apt install -y python3-certbot-nginx
sudo certbot -d example.com --manual --preferred-challenges dns certonly

# Add auto-renewal cron job
sudo crontab -e
# Add: 0 23 * * * certbot renew --quiet --deploy-hook "systemctl restart nginx"
```

#### 5.3 Install Pelican Panel
Follow the official documentation at [pelican.dev/docs](https://pelican.dev/docs)

#### 5.4 Setup Cloudflare Tunnel
Use the curl command provided in your Cloudflare Dashboard. Use the domain from step 5.2 as your Origin Server Name.

#### 5.5 Apply Configuration
Reboot both machines:
```bash
# On VPS
reboot

# On VM
sudo reboot
```

### 6. Additional Configuration

#### 6.1 Adding New Eggs
Get new eggs from: [pelican-eggs GitHub repository](https://github.com/pelican-eggs)

#### 6.2 Configure UDP Ports

Example configuration for Project Zomboid:

```bash
# Allow new ports in UFW
ufw allow 16261 && ufw allow 16262
```

Update Caddyfile (`/etc/caddy/Caddyfile`):
```
{
    layer4 {
        :25565 {
            route {
                proxy {
                    upstream 10.66.66.2:25565
                }
            }
        }
        
        udp/:16261 {
            route {
                proxy {
                    upstream udp/10.66.66.2:16261
                }
            }
        }
        
        udp/:16262 {
            route {
                proxy {
                    upstream udp/10.66.66.2:16262
                }
            }
        }
    }
}
```

### 🔧 DNS Configuration

To use a domain for server access:

1. Create A Record:
   - Name: `mc` (or your preference)
   - IPv4 Address: VPS Public IP

2. Create SRV Record:
   - Name: `_minecraft._tcp.mc`
   - Priority: 10
   - Weight: 10
   - Port: 25565
   - Target: `mc.domain.com`

---

## 📝 Notes
- Wireguard IP is your primary allocation for external access
- Local IP can be added as an additional allocation for local access
- Remember to reboot both machines after major configuration changes
