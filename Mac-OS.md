# **Setting Up LNbits with `phoenixd` on iOS**

You’ll find two scenarios covered:

- phoenixd running locally on iOS but LNbits/Caddy on an external Debian VPS
- phoenixd, LNbits, and Caddy running locally on iOS

You’ll find two scenarios covered:
- `phoenixd` running on iOS but `LNbits`/`Caddy` on an external Debian VPS
- `phoenixd`, `LNbits`, and `Caddy` running locally with `phoenixd` on iOS

## **1. Install Dependencies (Local Debian VPS)**

### **Update and Install System Packages**
```
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv unzip
```

### **Install Poetry**
```
curl -sSL https://install.python-poetry.org | python3 -
export PATH="$HOME/.local/bin:$PATH"
```
Add the Poetry path to your shell profile:
```
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## **2. Set Up LNbits**

### **Download and Set Up LNbits**
```
wget https://raw.githubusercontent.com/lnbits/lnbits/snapcraft/lnbits.sh
chmod +x lnbits.sh
./lnbits.sh
cd lnbits
```

### **Configure Environment**

#### **Scenario 1: `phoenixd` on iOS, `LNbits` and Caddy on External Debian VPS**

On your Debian VPS (where `LNbits` and `Caddy` are running), ensure the following in `.env`:
```
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://<ios-device-ip>:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```
Replace `<ios-device-ip>` with the IP address or hostname of your iOS device running `phoenixd`.

#### **Scenario 2: `phoenixd`, `LNbits`, and Caddy Locally with `phoenixd` on iOS**

If you run `LNbits` and `Caddy` locally on Debian and `phoenixd` on iOS:
```
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

## **3. Set Up phoenixd Using Pre-built Binaries**

Since `phoenixd` is running on iOS, no installation is required on Debian. Ensure `phoenixd` is accessible via the network from the Debian VPS.

## **4. Configure HTTPS and Reverse Proxy with Caddy**

### **Install Caddy**
```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### **Create Caddyfile**

#### **Scenario 1: `phoenixd` on iOS, `LNbits` and Caddy on External Debian VPS**
```
sudo nano /etc/caddy/Caddyfile
```
Add:
```
yourdomain.com {
  tls your-email@example.com

  handle /api/v1/payments/sse* {
    reverse_proxy <ios-device-ip>:5000 {
      header_up X-Forwarded-Host {host}
      transport http {
        keepalive off
        compression off
      }
    }
  }

  reverse_proxy <ios-device-ip>:5000 {
    header_up X-Forwarded-Host {host}
  }
}
```
Replace `<ios-device-ip>` with the IP address of your iOS device running `phoenixd`.

#### **Scenario 2: `phoenixd`, `LNbits`, and Caddy Locally**

On the local Debian machine:
```
sudo nano /etc/caddy/Caddyfile
```
Add:
```
yourdomain.com {
  tls your-email@example.com

  handle /api/v1/payments/sse* {
    reverse_proxy 127.0.0.1:5000 {
      header_up X-Forwarded-Host {host}
      transport http {
        keepalive off
        compression off
      }
    }
  }

  reverse_proxy 127.0.0.1:5000 {
    header_up X-Forwarded-Host {host}
  }
}
```

### **Start Caddy**
```
sudo systemctl start caddy
sudo systemctl enable caddy
```

## **5. Create Systemd Service Files**

### **Create LNbits Service File**
```
sudo nano /etc/systemd/system/lnbits.service
```
Add:
```
[Unit]
Description=LNbits

[Service]
WorkingDirectory=/home/lnbits/lnbits
ExecStart=/home/lnbits/.local/bin/poetry run lnbits
User=lnbits
Restart=always
TimeoutSec=120
RestartSec=30
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### **Reload Systemd and Start Services**
```
sudo systemctl daemon-reload
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## **6. Firewall Configuration**

### **Scenario 1: `phoenixd` on iOS, `LNbits` and Caddy on External Debian VPS**

On the Debian VPS (where `LNbits` and `Caddy` are running):
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### **Scenario 2: `phoenixd`, `LNbits`, and Caddy Locally with `phoenixd` on iOS**

For local firewall settings, allow necessary ports:
```
sudo ufw allow 5000/tcp
sudo ufw enable
```

## **7. Verify/Monitor Running Services**

### **Create a Script to Check Running Services**

#### **Scenario 1: `phoenixd` on iOS, `LNbits`, and Caddy on External Debian VPS**
```
nano ~/check_services.sh
```
Add:
```
#!/bin/bash
if systemctl is-active --quiet lnbits; then
    echo "LNbits is running."
else
    echo "LNbits is not running."
fi

if systemctl is-active --quiet caddy; then
    echo "Caddy is running."
else
    echo "Caddy is not running."
fi
```

#### **Scenario 2: `phoenixd`, `LNbits`, and Caddy Locally with `phoenixd` on iOS**

On the local machine:
```
systemctl is-active --quiet lnbits && echo "LNbits is running." || echo "LNbits is not running."
systemctl is-active --quiet caddy && echo "Caddy is running." || echo "Caddy is not running."
```

## **8. Start Script to Start All Services**

### **Scenario 1: `phoenixd` on iOS, `LNbits`, and Caddy on External Debian VPS**

On the Debian VPS:
```
nano ~/start_services_vps.sh
```
Add:
```
#!/bin/bash
echo "Starting LNbits..."
sudo systemctl start lnbits
sleep 5

echo "Starting Caddy..."
sudo systemctl start caddy
```

### **Scenario 2: `phoenixd`, `LNbits`, and Caddy Locally with `phoenixd` on iOS**

On the local machine:
```
nano ~/start_services_local.sh
```
Add:
```
#!/bin/bash
echo "Starting LNbits..."
sudo systemctl start lnbits
sleep 5

echo "Starting Caddy..."
sudo systemctl start caddy
```

---
