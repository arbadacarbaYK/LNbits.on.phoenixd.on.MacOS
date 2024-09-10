# Setting up LNbits with phoenixd on Debian VPS (SQLite Version)

## **1. Install Dependencies**

### **Update and install system packages**
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

#### **For Both Services on the Same VPS**

Copy the example environment file and edit it:
```
cp .env.example .env
nano .env
```

Update `.env` with:
```dotenv
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

#### **For Separate Local `phoenixd`**

If `phoenixd` is running on a different machine or IP address, update `.env` to:
```dotenv
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://<phoenixd-ip>:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```
Replace `<phoenixd-ip>` with the actual IP address or hostname of the machine running `phoenixd`.

### **Install dependencies**
```
poetry install
```

### **Run LNbits (Initial Test)**
```
poetry run lnbits --port 5000 --host 0.0.0.0
```

Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## **3. Set up phoenixd using pre-built binaries**

### **Download and extract phoenixd Binary**

1. **Download the appropriate binary for your system**:
   ```
   wget https://github.com/ACINQ/phoenixd/releases/download/v0.3.4/phoenix-0.3.4-linux-x64.zip
   ```

2. **Extract the downloaded file**:
   ```
   sudo apt install -y unzip
   unzip phoenix-0.3.4-linux-x64.zip
   chmod +x phoenix-0.3.4-linux-x64/phoenixd
   ```

3. **Run phoenixd**:
   ```
   ./phoenix-0.3.4-linux-x64/phoenixd
   ```

### **Set up Keyring for Phoenixd**

1. **Find the Phoenix Key**:
   ```
   cat ~/.phoenix/phoenix.conf
   ```
   - This file contains the path to your Phoenix key.

2. **Secure the Keyring Directory**:
   ```
   mkdir -p ~/.phoenix_key
   chmod 700 ~/.phoenix_key
   ```

3. **Place the phoenix_key File**:
   ```
   cp /path/to/phoenix_key ~/.phoenix_key/
   ```

4. **Set Environment Variable for Phoenix Key Path**:
   ```
   export PHOENIX_KEY_PATH=~/.phoenix_key/phoenix_key
   ```

## **4. Configure HTTPS and reverse proxy with Caddy**

### **Install Caddy**
```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### **Create Caddyfile**
```
sudo nano /etc/caddy/Caddyfile
```

Add the following:
```
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
```
sudo systemctl start caddy
sudo systemctl enable caddy
```

## **5. Create systemd service files**

### **Create phoenixd service file**
```
sudo nano /etc/systemd/system/phoenixd.service
```

Add the following content:
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

Replace `/path/to/phoenixd` with the actual path where you extracted `phoenixd`, and `youruser` with your actual username.

### **Create LNbits service file**
```
sudo nano /etc/systemd/system/lnbits.service
```

Add the following content:
```
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

### **Reload Systemd and start services**
```
sudo systemctl daemon-reload
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## **6. Verify/Monitor running Services**

### **Create a script to check running services**
```
nano ~/check_services.sh
```

Add the following content:
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

### **Make the Script Executable and Run It**
```
chmod +x ~/check_services.sh
~/check_services.sh
```

## **7. Start-skript to start all services in the right order**

### **Create the script**
```
nano ~/start_services.sh
```

### **Add the following content**
```
#!/bin/bash

# Start phoenixd first
echo "Starting phoenixd..."
sudo systemctl start phoenixd
sleep 5  # Wait for phoenixd to fully start

# Start Caddy next
echo "Starting Caddy..."
sudo systemctl start caddy
sleep 5  # Wait for Caddy to fully start and set up HTTPS

# Finally, start LNbits
echo "Starting LNbits..."
sudo systemctl start lnbits

echo "All services started successfully in the correct order."
```

### **Make the script executable**
```
chmod +x ~/start_services.sh
```

### **Run the script and start all services**
```
./start_services.sh
```

Adjust the `sleep` times if necessary to give more or less time for the services to start.

---
