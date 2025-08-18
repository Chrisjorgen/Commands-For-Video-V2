# DIY Game Server Setup
A comprehensive guide for setting up a DIY Game Server using Pelican Panel with Oracle Cloud and Cloudflare.

## 📋 Prerequisites

- A server to install the panel on
- A domain registered in Cloudflare
- An Oracle Account (preferably Pay as You Go) [Documentation for Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

> **Note**: Some commands may require `sudo`. Use it whenever applicable to your situation.

## 🚀 Installation Steps

### 1. Get Ubuntu Server ISO

Download from: [Ubuntu Server Download Page](https://ubuntu.com/download/server)

### 2. Create VM and Install Ubuntu Server

*Steps vary based on your setup (Guide uses Proxmox)*

### 3. Update and Setup VM

#### 3.1 Update VM
```bash
sudo apt update -y && sudo apt upgrade -y
```
#### Add php8.4 repository

##### Ubuntu
```bash
sudo LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php 
sudo apt update
```

#### 3.2 Install Panel Dependencies
```bash
sudo apt -y install php8.4 php8.4-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip,intl,sqlite3} mariadb-server nginx tar unzip git netcat-openbsd resolvconf wireguard
```

### 4. Setup Oracle VPS

#### Configuration
- **Image**: Canonical Ubuntu 24.04
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

###### Install Dependencies for xcaddy
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Add xcaddy GPG key for secure package downloads:
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg 

# Add the xcaddy repository to the system's sources list:
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list 

# Install xcaddy
apt update -y
apt install -y xcaddy
```

###### Download and Extract go (Check if Version is Up-to-Date)
```bash
wget https://go.dev/dl/go1.23.3.linux-arm64.tar.gz
tar -C /usr/local -xzf go1.23.3.linux-arm64.tar.gz  

# Add Go to your system's PATH to make it accessible from any directory
export PATH=$PATH:/usr/local/go/bin

# Apply the PATH changes by sourcing your profile:
source ~/.profile

# Check if go was installed successfully
go version 
```

###### Install Caddy-l4
```bash
xcaddy build --with github.com/mholt/caddy-l4
mv caddy /usr/local/bin/
caddy -v 

# Setup Caddy-l4
mkdir -p /etc/caddy
nano /etc/caddy/Caddyfile
```

Add to Caddyfile:
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

```bash
caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
nano /etc/systemd/system/caddy.service
```

Add to caddy.service:
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

```bash
systemctl daemon-reload
systemctl enable caddy
systemctl start caddy
systemctl status caddy
```

##### 4.1.5 Going Back to Home Directory if Necessary 
```bash
cd /home/ubuntu
```

##### 4.1.6 Open Wireguard Client Configuration File
```bash
ls
nano wg0-"your config name here".conf
```

### 5. Setup Pelican VM

#### 5.1 Connect to the Wireguard Tunnel
```bash
sudo nano /etc/wireguard/wg0.conf
```
(copy and paste in the configuration in the VPS)
```bash
sudo nano /etc/systemd/system/wg-quick@.service
```

Add to wg-quick@.service:
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

```bash
sudo systemctl daemon-reload
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
sudo systemctl status wg-quick@wg0.service
```

#### 5.2 Making Certificates for Panel
```bash
sudo apt install -y python3-certbot-nginx
sudo certbot -d example.com --manual --preferred-challenges dns certonly
sudo crontab -e
0 23 * * * certbot renew --quiet --deploy-hook "systemctl restart nginx"
```

#### 5.3 Installing Pelican Panel

This step requires you to follow the documentation here: [https://pelican.dev/docs](https://pelican.dev/docs) along with the video.

#### 5.4 Setup Cloudflare Tunnel

Use sudo with the curl command given in the Cloudflare Dashboard `sudo curl -L --output cloudflared.dev [...]`
Origin Server Name is the domain you used to make the certificates in #5.2

#### 5.5 Reboot both Machines to Apply Pending Configurations
```bash
# VPS
reboot

# VM
sudo reboot
```

#### 5.6 Install Pelican Wings and Making Node

This step requires you to follow the documentation here: [https://pelican.dev/docs](https://pelican.dev/docs) along with the video.

#### 5.7 Making and Trying first Server (Minecraft)

Your Wireguard IP is always your primary allocation if you want other users to access your server. You can add your local IP as an additional allocation for local access if you want to.

#### 5.8 Making A and SRV DNS Records to Use Domain to Join the Server

##### A Record: Name -> mc (or something else you want) | IPv4 Address -> Public IP of VPS
##### SRV Record: _minecraft._tcp.mc - 10 - 10 - 25565 - mc.domain.com (If Following Above)

### 6. Where to Get new eggs and How to Proxy UDP (Project Zomboid used as an example)

#### 6.1 Download and Add new eggs

Current github repository for eggs: [https://github.com/pelican-eggs](https://github.com/pelican-eggs)

#### 6.2 Setup New Ingress Rules for New Ports and Configure Caddy and UFW to use them:

```bash
ufw allow 16261 && ufw allow 16262
nano /etc/caddy/Caddyfile
```

(Replace "port" with your UDP port)

```
udp/:port {
    route {
        proxy {
            upstream udp/10.66.66.2:port
        }
    }
}
```

Below is how the complete config would look like:

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

Finally, reboot both machines once again.
