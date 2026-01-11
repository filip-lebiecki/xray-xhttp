# 🛡️ Xray (VLESS) + XHTTP + NGINX Stealth Proxy

This repository documents the setup of a stealth proxy that blends into normal web traffic. By using **NGINX** as a reverse proxy, **Xray** (VLESS) as the core, and the **XHTTP** transport protocol, we can disguise VPN traffic as standard HTTPS requests to a web game or API.

To a Deep Packet Inspection (DPI) firewall, this setup looks like:

1. **DNS:** Resolves to a legitimate domain.
2. **TLS:** Handshakes look like a standard Chrome browser visiting a site.
3. **Traffic:** Looks like a user playing a web game or making standard API calls.

## 🏗️ Architecture

1. **Public Interface (Port 443):** NGINX listens for HTTPS traffic.
2. **Traffic Splitting:**
* **Root path (`/`)**: Serves a legitimate static website (e.g., a 2048 game).
* **Secret path (`/api/stream`)**: Forwards traffic to the local Xray backend.


3. **Xray Backend (Port 8080):** Accepts decrypted HTTP traffic from NGINX, unwraps the VLESS protocol, and routes it to the internet.

---

## 📋 Prerequisites

* A Linux VPS (Ubuntu/Debian recommended).
* A domain name pointed to your server's IP (via Cloudflare or other DNS).
* Ports **80** and **443** open on your firewall.

---

## 🚀 Server-Side Setup

### 1. Install Dependencies

Update your system and install NGINX and Certbot.

```bash
apt update && apt install nginx certbot python3-certbot-nginx -y

```

### 2. Setup the "Camouflage" Website

Remove the default NGINX page and download a static website template (e.g., a game).

```bash
cd /var/www/html
rm index.nginx-debian.html
wget https://raw.githubusercontent.com/filip-lebiecki/xray-xhttp/refs/heads/main/index.html

```

### 3. Obtain SSL Certificates

Use Certbot to get a free Let's Encrypt certificate. Replace `yourdomain.com` with your actual domain.

```bash
certbot certonly --nginx -d yourdomain.com

```

### 4. Configure NGINX (Base HTTPS)

Set up NGINX to handle HTTPS and redirect HTTP to HTTPS.

```bash
cd /etc/nginx/sites-available
wget https://raw.githubusercontent.com/filip-lebiecki/xray-xhttp/refs/heads/main/default_https_site_only
cd /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/defaultln -s /etc/nginx/sites-available/default_https_site_only default
nginx -t
systemctl reload nginx
```

### 5. Install Xray

Install the latest version of Xray using the official script.

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

```

### 6. Configure Xray

Generate a UUID for authentication:

```bash
xray uuid
# Copy the output, you will need it for the config below

cd /usr/local/etc/xray
rm config.json
wget https://raw.githubusercontent.com/filip-lebiecki/xray-xhttp/refs/heads/main/config.json
```

*Note: Ensure `path` matches what we will configure in NGINX.*

Restart Xray:

```bash
systemctl restart xray

```

### 7. Configure NGINX Reverse Proxy

Update NGINX to forward the secret path to Xray.

```bash
cd /etc/nginx/sites-available
wget https://raw.githubusercontent.com/filip-lebiecki/xray-xhttp/refs/heads/main/default_https
cd /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/default_https default
nginx -t
systemctl reload nginx
```

Test and reload NGINX:

```bash
nginx -t
systemctl reload nginx

```

---

## 💻 Client Setup

You can use **v2RayN** (Windows), **v2RayNG** (Android), or the raw **Xray CLI**.

### Client Config (config.json)

```json
{
  "inbounds": [
    {
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "yourdomain.com",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID_HERE",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "yourdomain.com",
          "fingerprint": "chrome"
        },
        "xhttpSettings": {
          "path": "/api/stream",
          "mode": "packet-up"
        }
      }
    }
  ]
}

```

### v2RayN GUI Settings

If using a GUI, map the settings as follows:

* **Address:** `yourdomain.com`
* **Port:** `443`
* **Protocol:** `VLESS`
* **UUID:** (Paste your generated UUID)
* **Flow:** Empty (or packet-up if available)
* **Encryption:** `none`
* **Transport:** `xhttp` or `http` (depending on client version)
* **Path:** `/api/stream`
* **TLS:** `TLS`
* **SNI:** `yourdomain.com`
* **Fingerprint (uTLS):** `chrome`

---

## 🔍 Verification

1. **Check NGINX Logs:**
On the server, run:
```bash
tail -f /var/log/nginx/access.log

```


You should see `POST /api/stream` requests appearing as normal HTTP traffic.
2. **Check IP:**
On your client machine, configure your browser to use the SOCKS proxy (`127.0.0.1:1080`) and visit `https://ifconfig.me`. You should see your VPS IP address.

---

## 🧠 Theory: Why this works

* **VLESS:** A lightweight, stateless protocol for tunneling. It handles authentication but leaves encryption to the transport layer.
* **XHTTP:** Wraps VLESS frames into standard HTTP POST (upload) and GET (download) requests. This allows the traffic to pass through strict firewalls and CDNs that only allow valid HTTP.
* **TLS + uTLS:** Encrypts the XHTTP stream. We use `uTLS` with a `chrome` fingerprint to mimic the exact handshake of a Google Chrome browser, preventing DPI from flagging the connection based on TLS ClientHello signatures.
