# **Setting Up LNbits with `phoenixd` on Ubuntu VPS (SQLite Version)**

## **1. Install Dependencies**

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

#### **For Both Services on the Same VPS**

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

#### **For Separate Local `phoenixd`**

If `phoenixd` is running on a different machine or IP address, update `.env` to:
```
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://<phoenixd-ip>:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```
Replace `<phoenixd-ip>` with the actual IP address or hostname of the machine running `phoenixd`.

## **3. Set Up phoenixd Using Pre-built Binaries**

### **Download and Extract phoenixd Binary**

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

4. **Set Environment Variable for Phoenix Key Path**:
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

Replace `yourdomain.com` with your actual domain and ensure DNS is configured to point to your VPS static IP.

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

### **Create LNbits Service File**
```
sudo nano /etc/systemd/system/lnbits.service
```

Add the following content:
```
[Unit]
Description=LNbits
# Uncomment these lines if you have a dependency like lnd
#Wants=lnd.service
#After=lnd.service

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

## **6. Verify/Monitor Running Services**

### **Create a Script to Check Running Services on VPS**

If `LNbits` and `Caddy` are on the same VPS, use the following script:
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

### **For Separate Machines**

If `LNbits` and `Caddy` are on a different VPS, you can check their status remotely using SSH or a monitoring tool. Here's how you might adjust the script for remote checks:

1. **Check Services Remotely**:

   On your local machine, use SSH to check services on the remote VPS:
   ```
   ssh user@remote-vps-ip 'systemctl is-active --quiet phoenixd && echo "phoenixd is running." || echo "phoenixd is not running."'
   ssh user@remote-vps-ip 'systemctl is-active --quiet lnbits && echo "LNbits is running." || echo "LNbits is not running."'
   ssh user@remote-vps-ip 'systemctl is-active --quiet caddy && echo "Caddy is running." || echo "Caddy is not running."'
   ```

2. **Create a Combined Check Script**:

   ```
   nano ~/check_remote_services.sh
   ```

   Add the following content:
   ```
   #!/bin/bash

   # Replace with your remote VPS IP and user
   REMOTE_USER="user"
   REMOTE_IP="remote-vps-ip"

   echo "Checking services on remote VPS..."

   ssh $REMOTE_USER@$REMOTE_IP 'systemctl is-active --quiet phoenixd && echo "phoenixd is running." || echo "phoenixd is not running."'
   ssh $REMOTE_USER@$REMOTE_IP 'systemctl is-active --quiet lnbits && echo "LNbits is running." || echo "LNbits is not running."'
   ssh $REMOTE_USER@$REMOTE_IP 'systemctl is-active --quiet caddy && echo "Caddy is running." || echo "Caddy is not running."'
   ```

### **Make the Script Executable and Run It**
```
chmod +x ~/check_services.sh
chmod +x ~/check_remote_services.sh
~/check_services.sh
~/check_remote_services.sh
```

## **7. Start Script to Start All Services in the Right Order**

### **When `phoenixd` is Local and `LNbits`/`Caddy` are on a Separate VPS**

In this setup, you'll need to ensure that `phoenixd` starts first on the local VPS, followed by `LNbits` and `Caddy` on the separate VPS. Here's how to adapt the script for this configuration:

### **1. Create the Start Script on the Local VPS (where `phoenixd` is running)**
```
nano ~/start_services_local.sh
```

### **Add the Following Content**
```
#!/bin/bash

# Start phoenixd first (local VPS)
echo "Starting phoenixd..."
sudo systemctl start phoenixd
sleep 5  # Wait for phoenixd to fully start

# Notify the remote VPS to start LNbits and Caddy
echo "Triggering remote VPS to start LNbits and Caddy..."
ssh user@remote-vps-ip 'bash -s' << 'EOF'
#!/bin/bash

# Start Caddy (remote VPS)
echo "Starting Caddy..."
sudo systemctl start caddy
sleep 5  # Wait for Caddy to fully start and set up HTTPS

# Finally, start LNbits
echo "Starting LNbits..."
sudo systemctl start lnbits

echo "All services on remote VPS started successfully."
EOF

echo "All services started successfully on local and remote VPS."
```

Replace `user@remote-vps-ip` with the appropriate SSH user and IP address of the remote VPS where `LNbits` and `Caddy` are running.

### **2. Make the Script Executable**
```
chmod +x ~/start_services_local.sh
```

### **3. Run the Script on the Local VPS**
```
~/start_services_local.sh
```

### **When `LNbits`/`Caddy` are on the Remote VPS**

### **1. Create a Start Script on the Remote VPS (where `LNbits` and `Caddy` are running)**
```
nano ~/start_services_remote.sh
```

### **Add the Following Content**
```
#!/bin/bash

# Start Caddy first (remote VPS)
echo "Starting Caddy..."
sudo systemctl start caddy
sleep 5  # Wait for Caddy to fully start and set up HTTPS

# Finally, start LNbits
echo "Starting LNbits..."
sudo systemctl start lnbits

echo "All services started successfully on remote VPS."
```

### **2. Make the Script Executable**
```
chmod +x ~/start_services_remote.sh
```

### **3. Run the Script on the Remote VPS**
```
~/start_services_remote.sh
```

Adjust the `sleep` times if necessary to give more or less time for the services to start.

---
