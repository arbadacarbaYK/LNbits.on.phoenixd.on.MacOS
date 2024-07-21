# LNbits with Phoenixd on Ubuntu VPS
To set up LNbits with phoenixd on a free VPS running Ubuntu Linux, follow these steps:

## 1. Install Dependencies

### Update and Install System Packages:**
```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv postgresql
```

### Install PostgreSQL:**
```sh
sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
sudo -u postgres psql -c "CREATE DATABASE lnbits OWNER postgres;"
```

### Install Poetry:**
```sh
curl -sSL https://install.python-poetry.org | python3 -
export PATH="$HOME/.local/bin:$PATH"
```

## 2. Set Up LNbits

### Clone LNbits Repository:**
```sh
git clone https://github.com/lnbits/lnbits.git
cd lnbits
```

### Configure Environment:**
```sh
cp .env.example .env
nano .env
```
Update `.env` with:
```sh
LNBITS_DATABASE_URL="postgres://postgres:postgres@localhost:5432/lnbits"
```

### Install Dependencies and Add `psycopg2-binary`:**
```sh
poetry install
poetry add psycopg2-binary
```

### Run LNbits (Initial Test):**
```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

## 3. Set Up phoenixd

### Install Java Development Kit (JDK):**
```sh
sudo apt install -y openjdk-17-jdk
```

### Clone phoenixd Repository:**
```sh
git clone https://github.com/ACINQ/phoenixd.git
cd phoenixd
```

### Build phoenixd:**
```sh
./gradlew package
```

### Run phoenixd (Initial Test):**
```sh
./build/install/phoenixd/bin/phoenixd
```

## 4. Configure Reverse Proxy with Caddy

### Install Caddy:**
```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -fsSL https://getcaddy.com | bash
```

### Create Caddyfile:**
```sh
sudo nano /etc/caddy/Caddyfile
```
Add the following:
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

### Start Caddy:**
```sh
sudo systemctl start caddy
sudo systemctl enable caddy
```

## 5. Create Systemd Service Files

### Create phoenixd Service File:**
```sh
sudo nano /etc/systemd/system/phoenixd.service
```
Add the following content:
```sh
[Unit]
Description=phoenixd
After=network.target

[Service]
ExecStart=/path/to/phoenixd/build/install/phoenixd/bin/phoenixd
Restart=always
User=youruser

[Install]
WantedBy=multi-user.target
```

### Create LNbits Service File:**
```sh
sudo nano /etc/systemd/system/lnbits.service
```
Add the following content:
```sh
[Unit]
Description=LNbits
After=network.target

[Service]
ExecStart=/usr/local/bin/poetry run lnbits --port 5000 --host 0.0.0.0
WorkingDirectory=/path/to/lnbits
Restart=always
User=youruser
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### Reload Systemd and Start Services:**
```sh
sudo systemctl daemon-reload
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## 6. Verify Running Services

### Create a Script to Check Running Services:**
```sh
nano ~/check_services.sh
```
Add:
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

* **Make the Script Executable and Run It:**
```sh
chmod +x ~/check_services.sh
~/check_services.sh
```
