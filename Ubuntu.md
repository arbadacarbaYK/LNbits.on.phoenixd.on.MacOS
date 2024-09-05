# Setting up LNbits with phoenixd on Ubuntu VPS (SQLite Version)

## **1. Install Dependencies**

### **Update and Install System Packages**
```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv unzip
```

### **Install Poetry**
```sh
curl -sSL https://install.python-poetry.org | python3 -
export PATH="$HOME/.local/bin:$PATH"
```

Add the Poetry path to your shell profile:
```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## **2. Set Up LNbits**

### **Download and Set Up LNbits**
```sh
wget https://raw.githubusercontent.com/lnbits/lnbits/snapcraft/lnbits.sh
chmod +x lnbits.sh
./lnbits.sh
cd lnbits
```

### **Configure Environment**
Copy the example environment file and edit it:
```sh
cp .env.example .env
nano .env
```

Update `.env` with:
```env
# Database: to use SQLite, specify LNBITS_DATA_FOLDER
#           to use PostgreSQL, specify LNBITS_DATABASE_URL=postgres://...
#           to use CockroachDB, specify LNBITS_DATABASE_URL=cockroachdb://...
# for both PostgreSQL and CockroachDB, you'll need to install
#   psycopg2 as an additional dependency
LNBITS_DATA_FOLDER="./data"
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
# Enable HTTPS support behind a proxy
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

### **Install Dependencies**
```sh
poetry install
```

### **Run LNbits (Initial Test)**
```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## **3. Set Up phoenixd Using Pre-Built Binaries**

### **Download and Extract phoenixd Binary**

1. **Download the appropriate binary for your system**:
   ```sh
   wget https://github.com/ACINQ/phoenixd/releases/download/v0.3.4/phoenix-0.3.4-linux-x64.zip
   ```

2. **Extract the downloaded file**:
   ```sh
   sudo apt install -y unzip
   unzip phoenix-0.3.4-linux-x64.zip
   chmod +x phoenix-0.3.4-linux-x64/phoenixd
   ```

3. **Run phoenixd**:
   ```sh
   ./phoenix-0.3.4-linux-x64/phoenixd
   ```

### **Set up Keyring for Phoenixd**

1. **Find the Phoenix Key**:
   ```sh
   cat ~/.phoenix/phoenix.conf
   ```
   - This file contains the path to your Phoenix key.

2. **Secure the Keyring Directory**:
   ```sh
   mkdir -p ~/.phoenix_key
   chmod 700 ~/.phoenix_key
   ```

3. **

3. **Place the phoenix_key File**:
   ```sh
   cp /path/to/phoenix_key ~/.phoenix_key/
   ```

4. **Set Environment Variable for Phoenix Key Path**:
   ```sh
   export PHOENIX_KEY_PATH=~/.phoenix_key/phoenix_key
   ```

## **4. Configure HTTPS and Reverse Proxy with Caddy**

### **Install Caddy**
```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -fsSL https://getcaddy.com | sudo bash
```

### **Create Caddyfile**
```sh
sudo nano /etc/caddy/Caddyfile
```

Add the following:
```caddyfile
yourdomain.com {
  # Automatically get SSL certificates with HTTPS
  tls your-email@example.com
  
  # Handle Server-Sent Events (SSE) for payments
  handle /api/v1/payments/sse* {
    reverse_proxy 127.0.0.1:5000 {
      header_up X-Forwarded-Host {host}
      transport http {
        keepalive off
        compression off
      }
    }
  }
  
  # Reverse proxy for LNbits
  reverse_proxy 127.0.0.1:5000 {
    header_up X-Forwarded-Host {host}
  }
}
```

### **Start Caddy**
```sh
sudo systemctl start caddy
sudo systemctl enable caddy
```

## **5. Create Systemd Service Files**

### **Create phoenixd Service File**
```sh
sudo nano /etc/systemd/system/phoenixd.service
```

Add the following content:
```ini
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

Replace `/path/to/phoenixd` with the actual path where you extracted `phoenixd`, and `youruser` with your actual username.

### **Create LNbits Service File**
```sh
sudo nano /etc/systemd/system/lnbits.service
```

Add the following content:
```ini
[Unit]
Description=LNbits
After=network.target

[Service]
ExecStart=/home/youruser/.local/bin/poetry run lnbits --port 5000 --host 0.0.0.0
WorkingDirectory=/path/to/lnbits
Restart=always
User=youruser
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### **Reload Systemd and Start Services**
```sh
sudo systemctl daemon-reload
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## **6. Verify Running Services**

### **Create a Script to Check Running Services**
```sh
nano ~/check_services.sh
```

Add the following content:
```sh
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

### **Make the Script Executable and Run It**
```sh
chmod +x ~/check_services.sh
~/check_services.sh
```

---
