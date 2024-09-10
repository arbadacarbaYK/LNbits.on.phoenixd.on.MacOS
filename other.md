# **Changes Needed for Other Distros**

### **1. CentOS / RHEL (Red Hat Enterprise Linux)**

**Package Manager**: `yum` or `dnf` (for newer versions)

- **Install System Packages**: Use `yum` or `dnf` for package installation.
  ```
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install Caddy**: Use the official repository for Caddy.
  ```
  sudo dnf install -y yum-utils
  sudo dnf config-manager --add-repo https://repo.caddyserver.com/stable/caddy.repo
  sudo dnf install caddy
  ```
---

### **2. Fedora**

**Package Manager**: `dnf`

- **Install System Packages**:
  ```
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install Caddy**:
  ```
  sudo dnf install -y caddy
  ```

- **Configure phoenixd Key Management**

---

### **3. AlmaLinux / Rocky Linux (CentOS successors)**

**Package Manager**: `dnf`

- **Install System Packages**:
  ```
  sudo dnf update -y
  sudo dnf install -y curl git python3-pip python3-venv postgresql-server postgresql-contrib
  ```

- **Install Caddy**:
  ```
  sudo dnf install -y caddy
  ```

---
