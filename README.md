# LNbits.on.phoenixd.onMacOS# Setting Up LNbits and Phoenixd on macOS

## Prerequisites

0. Download Phoenixd for your specific Mac-Chip from https://github.com/ACINQ/phoenixd

Native MacOS x64

    ```sh
./gradlew packageMacOSX64
    ```

Native MacOS arm64

    ```sh
./gradlew packageMacOSArm64
    ```

1. **Install Homebrew**:
    ```sh
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    ```

2. **Install PostgreSQL**:
    ```sh
    brew install postgresql
    brew services start postgresql
    ```

3. **Install Python and Poetry**:
    ```sh
    brew install python
    curl -sSL https://install.python-poetry.org | python3 -
    ```

## Setting Up LNbits

1. **Clone the LNbits Repository**:
    ```sh
    git clone https://github.com/lnbits/lnbits.git
    cd lnbits
    ```

2. **Copy and Modify Environment File**:
    ```sh
    cp .env.example .env
    nano .env
    ```
    Set the following variables:
    ```env
    LNBITS_DATABASE_URL=postgres://postgres:<password>@localhost:5432/lnbits
    ```

3. **Install Dependencies and Run LNbits**:
    ```sh
    poetry install
    poetry run lnbits
    ```

## Setting Up PostgreSQL

1. **Create PostgreSQL User and Database**:
    ```sh
    psql postgres
    CREATE USER lnbits WITH PASSWORD 'yourpassword';
    CREATE DATABASE lnbits OWNER lnbits;
    ```

## Setting Up Phoenixd

1. **Run Phoenixd**:
    ```sh
    ./phoenixd &
    ```

2. **Create Phoenixd Launch Agent File**:
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.user.phoenixd</string>
        <key>ProgramArguments</key>
        <array>
            <string>/Users/yourusername/Desktop/phoenixd/phoenixd</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <true/>
    </dict>
    </plist>
    ```
    Save this as `~/Library/LaunchAgents/com.user.phoenixd.plist`.

3. **Load the Launch Agent**:
    ```sh
    launchctl load ~/Library/LaunchAgents/com.user.phoenixd.plist
    ```

## Setting Up Caddy

1. **Install Caddy**:
    ```sh
    brew install caddy
    ```

2. **Create a Caddyfile**:
    ```sh
    sudo nano /etc/Caddyfile
    ```
    Add the following content:
    ```Caddyfile
    toxic.lightning-pirates.com {
      handle /api/v1/payments/sse* {
        reverse_proxy 127.0.0.1:5000 {
          header_up X-Forwarded-Host toxic.lightning-pirates.com
          transport http {
            keepalive off
            compression off
          }
        }
      }
      reverse_proxy 127.0.0.1:5000 {
        header_up X-Forwarded-Host toxic.lightning-pirates.com
      }
    }
    ```

3. **Start Caddy**:
    ```sh
    sudo caddy start
    ```

## Setting Up LNbits as a Launch Agent

1. **Create LNbits Launch Agent File**:
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.user.lnbits</string>
        <key>ProgramArguments</key>
        <array>
            <string>/Users/yourusername/.local/bin/poetry</string>
            <string>run</string>
            <string>lnbits</string>
        </array>
        <key>EnvironmentVariables</key>
        <dict>
            <key>PYTHONUNBUFFERED</key>
            <string>1</string>
        </dict>
        <key>WorkingDirectory</key>
        <string>/Users/yourusername/Desktop/lnbits/phoenixd/lnbits</string>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <true/>
    </dict>
    </plist>
    ```
    Save this as `~/Library/LaunchAgents/com.user.lnbits.plist`.

2. **Load the Launch Agent**:
    ```sh
    launchctl load ~/Library/LaunchAgents/com.user.lnbits.plist
    ```

## Verify All Services

To check if all services are running, use the following script:

```sh
#!/bin/bash

services=("com.user.phoenixd" "com.user.lnbits")
all_running=true

for service in "${services[@]}"; do
    status=$(launchctl list | grep "$service")
    if [[ -z $status ]]; then
        echo "$service is NOT running"
        all_running=false
    else
        echo "$service is running"
    fi
done

if $all_running; then
    echo "All services are running fine."
else
    echo "Some services are not running. Please check the details above."
fi
