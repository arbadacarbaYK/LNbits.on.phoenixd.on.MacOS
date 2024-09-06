# **LNbits with Phoenixd on a Debian VPS (SQLite Version)**

## **1. Install Dependencies**

### **Update and Install System Packages**

Ensure your system is up-to-date and install the necessary packages:

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git python3-pip python3-venv unzip
```

### **Install Poetry**

Install Poetry for managing Python dependencies:

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

### **Clone LNbits Repository**

Clone the LNbits repository and set it up:

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

Use Poetry to install LNbits dependencies:

```sh
poetry install
```

### **Run LNbits (Initial Test)**

Run LNbits to ensure it starts correctly:

```sh
poetry run lnbits --port 5000 --host 0.0.0.0
```

You should be able to access LNbits at [http://127.0.0.1:5000](http://127.0.0.1:5000).

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
   ./phoenix-0.3.4-linux-x64/phoenixd
   cd phoenixd
   ```

### **Run phoenixd (Initial Test)**

Run phoenixd to ensure it starts correctly:

```sh
./phoenixd
```

## **4. Configure HTTPS and Reverse Proxy with Caddy**

### **Install Caddy**

Install Caddy using the following commands:

```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### **Create Caddyfile**

Create and edit the Caddyfile:

```sh
sudo nano /etc/caddy/Caddyfile
```

Add the following configuration:

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

Enable and start the Caddy service:

```sh
sudo systemctl start caddy
sudo systemctl enable caddy
```

## **5. Configure phoenixd Key Management**

Ensure you manage the `phoenixd` keys correctly:

1. **Generate a New Key Pair**:

   Use `phoenixd` to generate a new key pair:

   ```sh
   ./phoenixd --generate-keys
   ```

   This command will output a new public/private key pair. Save these keys securely, as they will be required for LNbits and phoenixd to communicate securely.

2. **Store Keys Securely**:

   Place the keys in a secure directory accessible to `phoenixd`. Update your LNbits configuration to use these keys.

   Update `.env` in the LNbits configuration directory with:

   ```env
   PHOENIXD_PRIVATE_KEY="/path/to/phoenixd_private.key"
   PHOENIXD_PUBLIC_KEY="/path/to/phoenixd_public.key"
   ```

## **6. Create Systemd Service Files**

### **Create phoenixd Service File**

Create a systemd service file for phoenixd:

```sh
sudo nano /etc/systemd/system/phoenixd.service
```

Add the following content:

```ini
[Unit]
Description=phoenixd
After=network.target

[Service]
ExecStart=/path/to/phoenixd/phoenixd --private-key /path/to/phoenixd_private.key
Restart=always
User=youruser

[Install]
WantedBy=multi-user.target
```

### **Create LNbits Service File**

Create a systemd service file for LNbits:

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

Reload systemd and enable the services:

```sh
sudo systemctl daemon-reload
sudo systemctl start phoenixd
sudo systemctl enable phoenixd
sudo systemctl start lnbits
sudo systemctl enable lnbits
```

## **7. Verify Running Services**

### **Create a Script to Check Running Services**

Create a script to check if the services are running:

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

Make the script executable and run it to verify the services:

```sh
chmod +x ~/check_services.sh
~/check_services.sh
```

---
