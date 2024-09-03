
# Setting up LNbits with `phoenixd` as a Funding Source on macOS

---

## 1. Install Dependencies

### Install Homebrew
If you don't have Homebrew installed, install it first:

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install PostgreSQL
```sh
brew install postgresql@14
brew services start postgresql@14
```

### Install Poetry
```sh
brew install poetry
```

## 2. Set Up LNbits

### Clone LNbits Repository
```sh
git clone https://github.com/lnbits/lnbits.git
cd lnbits
```

### Configure Environment
Copy the example environment file and modify it:

```sh
cp .env.example .env
nano .env
```

Update `.env` with the following configuration:

```sh
LNBITS_DATABASE_URL="postgres://postgres:postgres@localhost:5432/lnbits"
```

Other configurations may also be necessary depending on your setup.

### Install Dependencies and Add `psycopg2-binary`
```sh
poetry install
poetry add psycopg2-binary
```

### Run LNbits (Initial Test)
```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## 3. Set Up phoenixd Using Binaries

### Download and Extract Phoenixd Binary

Visit the [phoenixd releases page](https://github.com/ACINQ/phoenixd/releases) and download the appropriate binary for your macOS architecture:

- **For macOS x64:** `phoenix-0.3.4-macos-x64.zip`
- **For macOS arm64:** `phoenix-0.3.4-macos-arm64.zip`

After downloading, extract the binary:

```sh
unzip phoenix-0.3.4-macos-<architecture>.zip
cd phoenix-0.3.4-macos-<architecture>
```

Replace `<architecture>` with `x64` or `arm64` depending on your macOS hardware.

### Set Up Keyring for Phoenixd

`phoenixd` uses `phoenix_key` for key management. The binary should already be configured to use a secure keyring, but ensure that the keyring is set up correctly:

1. **Generate or Load the Key:**
   - Ensure you have your `phoenix_key` stored securely.
   - If it's your first time setting up, you might need to generate a new key using the instructions provided in the `phoenixd` documentation or a compatible key manager.

2. **Configure Environment for Keyring Access:**

Ensure that any environment variables or configurations required for `phoenix_key` access are set up correctly. This typically involves setting up a secure directory for the keyring.

```sh
export PHOENIX_KEY_PATH=/path/to/your/phoenix_key
```

Ensure this path is secure and accessible only by the `phoenixd` process.

### Run phoenixd (Initial Test)
Start `phoenixd` using the binary:

```sh
./phoenixd
```

Check if `phoenixd` is running and accessible at [http://127.0.0.1:9740](http://127.0.0.1:9740).

## 4. Configure Reverse Proxy with Caddy

### Install Caddy
```sh
brew install caddy
```

### Create Caddyfile
```sh
sudo nano /usr/local/etc/Caddyfile
```

Add the following to the Caddyfile:

```sh
yourdomain.com {
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

### Start Caddy
```sh
sudo caddy start
```

## 5. Create LaunchAgents for macOS

### Create phoenixd LaunchAgent

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

### Create LNbits LaunchAgent

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

### Load LaunchAgents

```sh
launchctl load ~/Library/LaunchAgents/com.user.phoenixd.plist
launchctl load ~/Library/LaunchAgents/com.user.lnbits.plist
```

## 6. Verify Running Services

### Create a Script to Check Running Services

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

### Make the Script Executable

```sh
chmod +x ~/check_services.sh
```

### Run the Script

```sh
~/check_services.sh
```

---
