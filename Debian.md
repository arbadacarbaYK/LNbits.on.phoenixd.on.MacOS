# **Setting Up LNbits with `phoenixd`**

You’ll find two scenarios covered:
- `phoenixd` running locally but `LNbits`/`Caddy` on an external Debian VPS
- `phoenixd`, `LNbits`, and `Caddy` running locally

## **1. Install Dependencies (Local and VPS)**

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

#### **Scenario 1: All Services Locally (on Debian)**

Copy the example environment file and edit it:
```
cp .env.example .env
nano .env
```
Update `.env` with:
```
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

#### **Scenario 2: `phoenixd` Locally, `LNbits` and `Caddy` on External VPS**

On your local machine (where `phoenixd` is running), ensure the following in `.env`:
```
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://<phoenixd-ip>:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```
Replace `<phoenixd-ip>` with the IP address or hostname of your local machine running `phoenixd`.

## **3. Set Up phoenixd Using Pre-built Binaries**

### **Download and Extract phoenixd Binary**
1. **Download the binary**:
   ```
   wget https://github.com/ACINQ/phoenixd/releases/download/v0.3.4/phoenix-0.3.4-linux-x64.zip
   ```
2. **Extract the file**:
   ```
   sudo apt install -y unzip
   unzip phoenix-0.3.4-linux-x64.zip
   chmod +x phoenix-0.3.4-linux-x64/phoenixd
   ```
3. **Run phoenixd**:
   ```
   ./phoenix-0.3.4-linux-x64/phoenixd
   ```

### **Set Up Keyring for Phoenixd**
1. **Find the Phoenix Key**:
   ```
   cat ~/.phoenix/phoenix.conf
   ```
2. **Secure the Keyring Directory**:
   ```
   mkdir -p ~/.phoenix_key
   chmod 700 ~/.phoenix_key
   ```
3. **Place the phoenix_key File**:
   ```
   cp /path/to/phoenix_key ~/.phoenix_key/
   ```
4. **Set Environment Variable**:
   ```
   export PHOENIX_KEY_PATH=~/.phoenix_key/phoenix_key
   ```

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

#### **Scenario 1: All Services Locally**
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

#### **Scenario 2: `phoenixd` Locally, `LNbits` and `Caddy` on External VPS**

On the external VPS (where `LNbits` and `Caddy` are running):
```
sudo nano /etc/caddy/Caddyfile
```
Add:
```
yourdomain.com {
  tls your-email@example.com

  handle /api/v1/payments/sse* {
    reverse_proxy <local-machine-ip>:5000 {
      header_up X-Forwarded-Host {host}
      transport http {
        keepalive off
        compression off
      }
    }
  }

  reverse_proxy <local-machine-ip>:5000 {
    header_up X-Forwarded-Host {host}
  }
}
```
Replace `<local-machine-ip>` with the IP address of your local machine running `phoenixd`.

### **Start Caddy**
```
sudo systemctl start caddy
sudo systemctl enable caddy
```

## **5. Create Systemd Service Files**

### **Create phoenixd Service File**
```
sudo nano /etc/systemd/system/phoenixd.service
```
Add:
```
[Unit]
Description=phoenixd
After=network.target

[Service]
ExecStart=/path/to/phoenixd/phoenixd
Restart=always
User=youruser
Environment=PHOENIX_KEY_PATH=/home/youruser/.phoenix_key/phoenix_key
WorkingDirectory=/path/to/phoenixd

[Install]
WantedBy=multi-user.target
```
Replace paths and user as needed.

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
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## **6. Firewall Configuration**

### **Scenario 1: All Services Locally**

For local firewall settings, allow necessary ports:
```
sudo ufw allow 5000/tcp
sudo ufw allow 9740/tcp
sudo ufw enable
```

### **Scenario 2: `phoenixd` Locally, `LNbits` and `Caddy` on External VPS**

On the local machine (where `phoenixd` is running):
```
sudo ufw allow 9740/tcp
sudo ufw enable
```

On the external VPS (where `LNbits` and `Caddy` are running):
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## **7. Verify/Monitor Running Services**

### **Create a Script to Check Running Services**

#### **Scenario 1: All Services Locally**
```
nano ~/check_services.sh
```
Add:
```
#!/bin/bash
if systemctl is-active --quiet phoenixd; then
    echo "phoenixd is running."
else
    echo "phoenixd is not running."
fi

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

#### **Scenario 2: `phoenixd` Locally, `LNbits` and `Caddy` on External VPS**

On the local machine:
```
ssh user@remote-vps-ip 'systemctl is-active --quiet lnbits && echo "LNbits is running." || echo "LNbits is not running."'
ssh user@remote-vps-ip 'systemctl is-active --quiet caddy && echo "Caddy is running." || echo "Caddy is not running."'
```

## **8. Start Script to Start All Services**

### **Scenario 1: All Services Locally**
```
nano ~/

start_services_local.sh
```
Add:
```
#!/bin/bash
echo "Starting phoenixd..."
sudo systemctl start phoenixd
sleep 5

echo "Starting LNbits..."
sudo systemctl start lnbits
sleep 5

echo "Starting Caddy..."
sudo systemctl start caddy
```

### **Scenario 2: `phoenixd` Locally, `LNbits` and `Caddy` on External VPS**

On the local machine:
```
nano ~/start_services_local.sh
```
Add:
```
#!/bin/bash
echo "Starting phoenixd..."
sudo systemctl start phoenixd
```

On the external VPS:
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

---
