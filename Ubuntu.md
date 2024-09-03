### **LNbits with Phoenixd on an Ubuntu VPS**

We will build from binaries because some ppl had problems with their own glib version.
So to set up LNbits with phoenixd on a free VPS running Ubuntu Linux, follow these steps:

---

## **1. Install Dependencies**

### **Update and Install System Packages**
```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv postgresql unzip
```

### **Install PostgreSQL**
```sh
sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
sudo -u postgres psql -c "CREATE DATABASE lnbits OWNER postgres;"
```

### **Install Poetry**
```sh
curl -sSL https://install.python-poetry.org | python3 -
export PATH="$HOME/.local/bin:$PATH"
```

## **2. Set Up LNbits**

### **Clone LNbits Repository**
```sh
git clone https://github.com/lnbits/lnbits.git
cd lnbits
```

### **Configure Environment**
```sh
cp .env.example .env
nano .env
```

Update `.env` with:
```sh
LNBITS_DATABASE_URL="postgres://postgres:postgres@localhost:5432/lnbits"
```

### **Install Dependencies and Add `psycopg2-binary`**
```sh
poetry install
poetry add psycopg2-binary
```

### **Run LNbits (Initial Test)**
```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

Check if LNbits is running at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## **3. Set Up phoenixd Using Pre-Built Binaries**

### **Download and Extract phoenixd Binary**
1. **Navigate to the phoenixd release page** on GitHub: [phoenixd Releases](https://github.com/ACINQ/phoenixd/releases)
2. **Download the appropriate binary for your system** (e.g., `phoenix-0.3.4-linux-x64.zip`).
3. **Extract the downloaded file:**
   ```sh
   unzip phoenix-0.3.4-linux-x64.zip -d phoenixd
   cd phoenixd
   ```

### **Set Up Keyring for Phoenixd**

`phoenixd` requires a `phoenix_key` for secure key management. Follow these steps to ensure the keyring is properly configured:

1. **Generate or Import Phoenix Key:**
   - If you do not already have a `phoenix_key`, generate a new one or import an existing one.
   - To generate a new key, use the appropriate key management tool compatible with `phoenixd`.

2. **Secure the Keyring Directory:**

Create a directory to securely store the `phoenix_key`:

```sh
mkdir -p ~/.phoenix_key
chmod 700 ~/.phoenix_key
```

3. **Place the `phoenix_key` File:**

Copy your `phoenix_key` to the newly created directory:

```sh
cp /path/to/your/phoenix_key ~/.phoenix_key/
```

4. **Set Environment Variable for Phoenix Key Path:**

Ensure that the `phoenix_key` path is set as an environment variable before starting `phoenixd`:

```sh
export PHOENIX_KEY_PATH=~/.phoenix_key/phoenix_key
```

### **Run phoenixd (Initial Test)**
```sh
./phoenixd
```

Check if phoenixd is running and accessible at [http://127.0.0.1:9740](http://127.0.0.1:9740).

## **4. Configure Reverse Proxy with Caddy**

### **Install Caddy**
```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -fsSL https://getcaddy.com | bash
```

### **Create Caddyfile**
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

Replace `/path/to/phoenixd` with the actual path where you extracted `phoenixd`, and replace `youruser` with your actual username.

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
