# **Setting Up LNbits with `phoenixd` as a Funding Source on macOS**

## **1. Install Dependencies**

### **Install Homebrew**
```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### **Install Dependencies**
```sh
brew install python@3 unzip
```

### **Install Poetry**
```sh
brew install poetry
```

## **2. Set Up LNbits**

### **Download and Run LNbits Setup Script**
```sh
wget https://raw.githubusercontent.com/lnbits/lnbits/snapcraft/lnbits.sh
chmod +x lnbits.sh
./lnbits.sh
cd lnbits
```

### **Configure Environment**
Copy the example environment file and modify it:

```sh
cp .env.example .env
nano .env
```

Update `.env` with the following configuration:

```env
LNBITS_DATA_FOLDER="./data"
# LNBITS_DATABASE_URL="postgres://user:password@host:port/databasename" (Commented out for SQLite)
# Enable HTTPS support behind a proxy
FORWARDED_ALLOW_IPS="*"
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD="your_phoenixd_key"
```

### **Install Dependencies**

Since we're using SQLite, you don't need `psycopg2-binary`:

```sh
poetry install
```

### **Run LNbits (Initial Test)**
Run LNbits to ensure it starts correctly:

```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

You should be able to access LNbits at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## **3. Set Up phoenixd Using Binaries**

### **Download and Extract Phoenixd Binary**

Visit the [phoenixd releases page](https://github.com/ACINQ/phoenixd/releases) and download the appropriate binary for your macOS architecture:

- **For macOS x64:** `phoenix-0.3.4-macos-x64.zip`
- **For macOS arm64:** `phoenix-0.3.4-macos-arm64.zip`

After downloading, extract the binary:

```sh
unzip phoenix-0.3.4-macos-<architecture>.zip
cd phoenix-0.3.4-macos-<architecture>
```

Replace `<architecture>` with `x64` or `arm64` depending on your macOS hardware.

### **Set Up Keyring for Phoenixd**

`phoenixd` uses `phoenix_key` for key management. Ensure the keyring is set up correctly:

1. **Retrieve the Phoenix Key:**
   - The `phoenix_key` can be found by running the following command:
   ```sh
   cat ~/.phoenix/phoenix.conf
   ```

2. **Configure Environment for Keyring Access:**
   Set up the environment variable to point to your `phoenix_key`:

   ```sh
   export PHOENIX_KEY_PATH=/path/to/your/phoenix_key
   ```

   Ensure this path is secure and accessible only by the `phoenixd` process.

### **Run phoenixd (Initial Test)**
Start `phoenixd` using the binary:

```sh
./phoenixd
```

Check if `phoenixd` is running and accessible at [http://127.0.0.1:9740](http://127.0.0.1:9740).

## **4. Configure HTTPS and Reverse Proxy with Caddy**

### **Install Caddy**
```sh
brew install caddy
```

### **Create Caddyfile**
```sh
sudo nano /usr/local/etc/Caddyfile
```

Add the following to the Caddyfile:

```sh
yourdomain.com {
  # Automatically get SSL certificates with HTTPS
  tls your-email@example.com  # Use a valid email for Let's Encrypt
  
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

Replace `yourdomain.com` with your actual domain and ensure you have DNS configured to point to your server.

### **Start Caddy**
```sh
sudo caddy start
```

Check if your LNbits instance is accessible via HTTPS at `https://yourdomain.com`.

## **5. Create LaunchAgents for macOS**

### **Create phoenixd LaunchAgent**

Create `~/Library/LaunchAgents/com.user.phoenixd.plist` with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.phoenixd</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/yourusername/phoenix-0.3.4-macos-<architecture>/phoenixd</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Replace `<architecture>` with `x64` or `arm64` as needed.

### **Create LNbits LaunchAgent**

Create `~/Library/LaunchAgents/com.user.lnbits.plist` with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.lnbits</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/poetry</string>
        <string>run</string>
        <string>lnbits</string>
        <string>--port</string>
        <string>5000</string>
        <string>--host</string>
        <string>0.0.0.0</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>WorkingDirectory</key>
    <string>/Users/yourusername/lnbits</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PYTHONUNBUFFERED</key>
        <string>1</string>
    </dict>
</dict>
</plist>
```

### **Load LaunchAgents**

```sh
launchctl load ~/Library/LaunchAgents/com.user.phoenixd.plist
launchctl load ~/Library/LaunchAgents/com.user.lnbits.plist
```

## **6. Verify Running Services**

### **Create a Script to Check Running Services**

Create a new script:

```sh
nano ~/check_services.sh
```

Add the following content:

```sh
#!/bin/bash

# Check LNbits service
lnbits_status=$(launchctl list | grep com.user.lnbits)
if [[ -n "$lnbits_status" ]]; then
    echo "LNbits is running."
else
    echo "LNbits is not running."
fi

# Check phoenixd service
phoenixd_status=$(launchctl list | grep com.user.phoenixd)
if [[ -n "$phoenixd_status" ]]; then
    echo "phoenixd is running."
else
    echo "phoenixd is not running."
fi

# Check Caddy service
caddy_status=$(ps aux | grep caddy | grep -v grep)
if [[ -n "$caddy_status" ]]; then
    echo "Caddy is running."
else
    echo "Caddy is not running."
fi
```

### **Make the Script Executable**

```sh
chmod +x ~/check_services.sh
```

### **Run the Script**

```sh
~/check_services.sh
```

---
