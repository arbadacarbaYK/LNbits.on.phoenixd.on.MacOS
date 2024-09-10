# **Changes Needed for Other Distros**

### **1. CentOS / RHEL (Red Hat Enterprise Linux)**

**Package Manager**: `yum` or `dnf` (for newer versions)

- **Install System Packages**: Use `yum` or `dnf` for package installation.
  ```sh
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install PostgreSQL**: Initialize and start the PostgreSQL service.
  ```sh
  sudo postgresql-setup --initdb
  sudo systemctl start postgresql
  sudo systemctl enable postgresql

  sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
  sudo -u postgres psql -c "CREATE DATABASE lnbits OWNER postgres;"
  ```

- **Install Poetry**: Same as other systems.
  ```sh
  curl -sSL https://install.python-poetry.org | python3 -
  export PATH="$HOME/.local/bin:$PATH"
  ```

- **Install Caddy**: Use the official repository for Caddy.
  ```sh
  sudo dnf install -y yum-utils
  sudo dnf config-manager --add-repo https://repo.caddyserver.com/stable/caddy.repo
  sudo dnf install caddy
  ```

- **Configure phoenixd Key Management**

  1. **Generate a New Key Pair**:
     ```sh
     ./phoenixd --generate-keys
     ```
     Save the public/private key pair securely.

  2. **Store Keys Securely**:
     Place the keys in a secure directory, accessible to `phoenixd`, and update your LNbits `.env` configuration file:
     ```sh
     nano /path/to/lnbits/.env
     ```

     Update with:
     ```env
     PHOENIXD_PRIVATE_KEY="/path/to/phoenixd_private.key"
     PHOENIXD_PUBLIC_KEY="/path/to/phoenixd_public.key"
     ```

- **Service Files**: Use the same systemd service files as on Ubuntu/Debian.

---

### **2. Fedora**

**Package Manager**: `dnf`

- **Install System Packages**:
  ```sh
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install PostgreSQL**: Initialize and start PostgreSQL.
  ```sh
  sudo postgresql-setup --initdb
  sudo systemctl start postgresql
  sudo systemctl enable postgresql

  sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
  sudo -u postgres psql -c "CREATE DATABASE lnbits OWNER postgres;"
  ```

- **Install Poetry**: Same as on other systems.
  ```sh
  curl -sSL https://install.python-poetry.org | python3 -
  export PATH="$HOME/.local/bin:$PATH"
  ```

- **Install Caddy**:
  ```sh
  sudo dnf install -y caddy
  ```

- **Configure phoenixd Key Management**

  1. **Generate a New Key Pair**:
     ```sh
     ./phoenixd --generate-keys
     ```
     Save the public/private key pair securely.

  2. **Store Keys Securely**:
     Place the keys in a secure directory, accessible to `phoenixd`, and update your LNbits `.env` configuration file:
     ```sh
     nano /path/to/lnbits/.env
     ```

     Update with:
     ```env
     PHOENIXD_PRIVATE_KEY="/path/to/phoenixd_private.key"
     PHOENIXD_PUBLIC_KEY="/path/to/phoenixd_public.key"
     ```

- **Service Files**: Use the same systemd service files as on Ubuntu/Debian.

---

### **3. AlmaLinux / Rocky Linux (CentOS successors)**

**Package Manager**: `dnf`

- **Install System Packages**:
  ```sh
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install PostgreSQL**: Initialize and start PostgreSQL.
  ```sh
  sudo postgresql-setup --initdb
  sudo systemctl start postgresql
  sudo systemctl enable postgresql

  sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
  sudo -u postgres psql -c "CREATE DATABASE lnbits OWNER postgres;"
  ```

- **Install Poetry**: Same as on other systems.
  ```sh
  curl -sSL https://install.python-poetry.org | python3 -
  export PATH="$HOME/.local/bin:$PATH"
  ```

- **Install Caddy**:
  ```sh
  sudo dnf install -y caddy
  ```

- **Configure phoenixd Key Management**

  1. **Generate a New Key Pair**:
     ```sh
     ./phoenixd --generate-keys
     ```
     Save the public/private key pair securely.

  2. **Store Keys Securely**:
     Place the keys in a secure directory, accessible to `phoenixd`, and update your LNbits `.env` configuration file:
     ```sh
     nano /path/to/lnbits/.env
     ```

     Update with:
     ```env
     PHOENIXD_PRIVATE_KEY="/path/to/phoenixd_private.key"
     PHOENIXD_PUBLIC_KEY="/path/to/phoenixd_public.key"
     ```

- **Service Files**: Use the same systemd service files as on Ubuntu/Debian.

---
