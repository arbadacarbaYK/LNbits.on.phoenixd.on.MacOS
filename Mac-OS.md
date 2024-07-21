
# Setting up LNbits with `phoenixd` as a Funding Source on macOS

## 1. Install Dependencies

### Homebrew
If you don't have Homebrew installed, you can install it with:
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
```sh
cp .env.example .env
nano .env
```
Update `.env` with:
- Set `LNBITS_DATABASE_URL="postgres://postgres:postgres@localhost:5432/lnbits"`
- Other required configurations as needed.

### Install Dependencies and Add psycopg2-binary
```sh
poetry install
poetry add psycopg2-binary
```

### Run LNbits (Initial Test)
```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```
Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## 3. Set Up phoenixd

### Clone phoenixd Repository
```sh
git clone https://github.com/ACINQ/phoenixd.git
cd phoenixd
```

### Build and Run phoenixd
Choose the appropriate build command based on your macOS architecture:

#### Native MacOS x64:
```sh
./gradlew packageMacOSX64
```

#### Native MacOS arm64:
```sh
./gradlew packageMacOSArm64
```

### Run phoenixd (Initial Test)
```sh
./phoenixd
```
Check if phoenixd is running at [http://127.0.0.1:9740](http://127.0.0.1:9740).

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
Create `~/Library/LaunchAgents/com.user.phoenixd.plist` with:
```sh
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.phoenixd</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/yourusername/projects/phoenixd/phoenixd</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### Create LNbits LaunchAgent

Create `~/Library/LaunchAgents/com.user.lnbits.plist` with:
```sh
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

### Create a script to check running services
```sh
nano ~/check_services.sh
```

Add the following content:
```sh
#!/bin/bash
```

# Check LNbits service
```sh
lnbits_status=$(launchctl list | grep com.user.lnbits)
if [[ -n "$lnbits_status" ]]; then
    echo "LNbits is running."
else
    echo "LNbits is not running."
```

# Check phoenixd service
```sh
phoenixd_status=$(launchctl list | grep com.user.phoenixd)
if [[ -n "$phoenixd_status" ]]; then
    echo "phoenixd is running."
else
    echo "phoenixd is not running."
```

# Check Caddy service
```sh
caddy_status=$(ps aux | grep caddy | grep -v grep)
if [[ -n "$caddy_status" ]]; then
    echo "Caddy is running."
else
    echo "Caddy is not running."
```

### Make the script executable
```sh
chmod +x ~/check_services.sh
```

### Run the script
```sh
~/check_services.sh
```
