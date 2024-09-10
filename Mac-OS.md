# **Setting Up LNbits with `phoenixd` on iOS and VPS**

## **On iOS (Running `phoenixd`)**

### **1. Set Up `phoenixd`**

#### **Download and Extract `phoenixd` Binary**

1. Visit the [phoenixd releases page](https://github.com/ACINQ/phoenixd/releases) and download the appropriate binary for macOS (x64 or arm64).
   
   - **For macOS x64:** `phoenix-0.3.4-macos-x64.zip`
   - **For macOS arm64:** `phoenix-0.3.4-macos-arm64.zip`

2. Extract the binary:

   ```sh
   unzip phoenix-0.3.4-macos-<architecture>.zip
   cd phoenix-0.3.4-macos-<architecture>
   ```

   Replace `<architecture>` with `x64` or `arm64` based on your macOS hardware.

#### **Set Up Keyring for `phoenixd`**

1. **Retrieve the Phoenix Key:**

   ```sh
   cat ~/.phoenix/phoenix.conf
   ```

   This file contains the path to your Phoenix key.

2. **Create Keyring Directory:**

   ```sh
   mkdir -p ~/.phoenix_key
   chmod 700 ~/.phoenix_key
   ```

3. **Copy the Phoenix Key File:**

   ```sh
   cp /path/to/phoenix_key ~/.phoenix_key/
   ```

4. **Set Environment Variable:**

   ```sh
   export PHOENIX_KEY_PATH=~/.phoenix_key/phoenix_key
   ```

   Ensure this environment variable is available in your shell or application environment.

#### **Run `phoenixd`**

Start `phoenixd` using the binary:

```sh
./phoenixd
```

Ensure `phoenixd` is running and accessible at [http://127.0.0.1:9740](http://127.0.0.1:9740).

## **On VPS (Running LNbits and Caddy)**

### **1. Install Dependencies**

#### **Update and Install System Packages**

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv unzip
```

#### **Install Poetry**

```sh
curl -sSL https://install.python-poetry.org | python3 -
export PATH="$HOME/.local/bin:$PATH"
```

Add the Poetry path to your shell profile:

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### **2. Set Up LNbits**

#### **Download and Set Up LNbits**

```sh
wget https://raw.githubusercontent.com/lnbits/lnbits/snapcraft/lnbits.sh
chmod +x lnbits.sh
./lnbits.sh
cd lnbits
```

#### **Configure Environment**

Copy the example environment file and modify it:

```sh
cp .env.example .env
nano .env
```

Update `.env` with:

```dotenv
LNBITS_DATA_FOLDER="./data"
# Uncomment and adjust the line if using PostgreSQL or CockroachDB:
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename"
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://<ios-device-ip>:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

Replace `<ios-device-ip>` with the IP address of your iOS device running `phoenixd`.

#### **Install Dependencies**

```sh
poetry install
```

#### **Run LNbits (Initial Test)**

```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

### **3. Set Up Caddy**

#### **Install Caddy**

```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

#### **Create Caddyfile**

```sh
sudo nano /etc/caddy/Caddyfile
```

Add the following content:

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

Replace `yourdomain.com` with your actual domain and ensure DNS is configured.

#### **Start Caddy**

```sh
sudo systemctl start caddy
sudo systemctl enable caddy
```

### **4. Create Systemd Service Files**

#### **Create `phoenixd` Service File**

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

Replace `/path/to/phoenixd` with the path where you extracted `phoenixd` and `youruser` with your username.

#### **Create LNbits Service File**

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

### **5. Reload Systemd and Start Services**

```sh
sudo systemctl daemon-reload
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

### **6. Verify Running Services**

#### **Create a Script to Check Running Services**

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

#### **Make the Script Executable and Run It**

```sh
chmod +x ~/check_services.sh
~/check_services.sh
```

### **Firewall Considerations**

#### **On the VPS**

1. **Allow Traffic for LNbits and Caddy**

   ```sh
   sudo ufw allow 5000/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

2. **Allow Incoming Traffic from iOS Device**

   ```sh
   sudo ufw allow from <ios-device-ip> to any port 9740
   ```

   Replace `<ios-device-ip>` with the IP address of your iOS device.

3. **Verify Firewall Rules**

   ```sh
   sudo ufw status
   ```

#### **On iOS**

1. **Network Configuration**

   Ensure your iOS device is on a network that allows outbound connections to the VPS on port 9740.

2. **VPNs or Security Apps**

   If using a VPN or security apps on iOS, ensure they are configured to allow connections to your VPS.

### **Testing Connectivity**

1. **From VPS to iOS Device**

   ```sh
   curl http://<ios-device-ip>:9740/
   ```

   or

   ```sh
   telnet <ios-device-ip> 9740
   ```

2. **From iOS Device to VPS**

   ```sh
   curl http://<vps-domain-or-ip>:5000/
   ```

Adjust firewall settings as needed based on your network configuration and security requirements.

---

