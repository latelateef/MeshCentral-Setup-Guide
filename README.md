# MeshCentral on Google Cloud — Detailed README

> This README describes step-by-step instructions to deploy MeshCentral on a Google Cloud (GCE) Ubuntu VM, configure it to run as a service, secure it, and troubleshoot common issues.

---

## Table of Contents

1. Overview & prerequisites
2. Create a Google Cloud VM (GUI and `gcloud` examples)
3. Connect to the VM (SSH)
4. System preparation & Node.js installation
5. Install MeshCentral and first run
6. Edit configuration for WAN mode and SSL
7. Run MeshCentral as a systemd service
8. Firewall & network notes (GCP firewall + UFW)
9. Troubleshooting & logs
10. After setup: access & creating first account

---

## 1. Overview & prerequisites

What this guide assumes:

* You have a Google Cloud account with permission to create VMs and firewall rules.
* Basic familiarity with SSH, Linux command line, and editing files with `nano` or `vim`.
* You will use an Ubuntu LTS image (example uses Ubuntu 22.04 LTS).
* You want MeshCentral to be reachable from the internet on ports 80 and 443 (HTTP/HTTPS).

Files and folders MeshCentral creates on first run (example):

* `meshcentral-data/` — configuration, database and site files
* `meshcentral-files/` — uploaded files and agent installers
* `meshcentral-backups/` — server backups (if configured)

---

## 2. Create a Google Cloud VM

### Google Cloud Console (UI)

1. Go to **Compute Engine > VM instances** > **Create instance**.
2. Choose a name (e.g. `meshcentral-server`).
3. Choose region & zone.
4. Machine type: `e2-medium` or similar (depending on expected usage).
5. Boot disk: **Ubuntu 22.04 LTS** (or 20.04 LTS).
6. Under **Firewall**, check **Allow HTTP traffic** and **Allow HTTPS traffic**.
7. Create the VM.

---

## 3. Connect to the VM (SSH)

From the Cloud Console click **SSH**

---

## 4. System preparation & Node.js (LTS) installation

After SSHing in, run these steps:

```bash
# Update system packages
sudo apt upgrade -y

# Install curl if missing
sudo apt install -y curl build-essential

# Install Node.js LTS (example uses 18.x)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify versions
node -v
npm -v
```

---

## 5. Install MeshCentral and first run

Create a working directory and install MeshCentral:

```bash
# Create folder and enter it
mkdir -p ~/meshcentral && cd ~/meshcentral

# Install MeshCentral locally (no global install required)
sudo npm install meshcentral

# First run to generate data and certs (LAN mode initially)
sudo node node_modules/meshcentral
```

On first run you will see MeshCentral generate certificates and folders such as `meshcentral-data/`, and logs similar to:

```
Generating certificates, may take a few minutes...
Generating root certificate...
MeshCentral HTTP redirection server running on port 80.
MeshCentral HTTPS server running on port 443.
Server has no users, next new account will be site administrator.
```

Press `Ctrl+C` to stop the server after the first run — this allows editing configuration before exposing the server to WAN.

---

## 6. Edit configuration for WAN mode and SSL

Open the generated config file and edit relevant fields:

```bash
sudo nano ~/meshcentral/meshcentral-data/config.json
```

A recommended minimal `settings` excerpt:

```json
{
  "settings": {
    "cert": "your.domain.example",
    "WANonly": true,
    "sessionKey": "A-Strong-Random-Session-Key",
    "port": 443,
    "redirPort": 80
  },
  "domains": {
    "": {
      "title": "MyMeshCentral",
      "newAccounts": true,
      "userNameIsEmail": true
    }
  }
}
```

Key notes:

* `cert`: set to your domain name (recommended) or the external IP if you don’t have a domain.
* `WANonly: true` exposes the server on the WAN (internet).
* `sessionKey`: choose a long, random passphrase. This protects cookies/sessions and is required.

After saving config, start MeshCentral again to load the new settings:

```bash
sudo node node_modules/meshcentral
```

You should now see `WAN mode` in the server output if configured correctly.

---

## 7. Run MeshCentral as a systemd service

Create a systemd service so MeshCentral runs in the background and restarts automatically.

1. Create service file:

```bash
sudo nano /etc/systemd/system/meshcentral.service
```

2. Paste (replace `/home/yourusername/meshcentral` and `yourusername`):
Example: ram@computername:~/meshcentral -> ram is the username and other one is computername

```ini
[Unit]
Description=MeshCentral Server
After=network.target

[Service]
Type=simple
User=yourusername
WorkingDirectory=/home/yourusername/meshcentral
ExecStart=/usr/bin/node /home/yourusername/meshcentral/node_modules/meshcentral
Restart=always

[Install]
WantedBy=multi-user.target
```

3. Fix ownership and permissions (replace `<username>`):

```bash
sudo chown -R <username>:<username> /home/<username>/meshcentral
```

4. Grant Node permission to bind to low ports (80/443) without running as root:

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/node
```

5. Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable meshcentral
sudo systemctl start meshcentral
sudo systemctl status meshcentral
```

If you later change the config file, restart the service:

```bash
sudo systemctl restart meshcentral
```

---

## 8. Firewall & network notes (GCP firewall + UFW)

* **GCP firewall**: Ensure there is a firewall rule allowing inbound TCP 80 and 443 for the VM (see Section 2). Firewall rules in GCP take precedence for external access.
* **UFW (on the VM)**: If `ufw` is enabled locally, allow ports:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH
sudo ufw enable
```

* Confirm ports are listening on the VM:

```bash
sudo ss -tlnp | grep node
# or
sudo lsof -iTCP -sTCP:LISTEN -P -n | grep node
```

---

## 9. Troubleshooting & logs

* View real-time logs from systemd:

```bash
sudo journalctl -u meshcentral -f
```

* Common problems:

  * **Port bind error**: ensure `setcap` was applied and service is not running as root. Check `/usr/bin/node` path.
  * **Permissions issues**: ensure `meshcentral-data` and entire `~/meshcentral` folder are owned by the service user.

---

## 10. After setup: access & creating first account

* Open your browser to `https://EXTERNAL_IP/` or `https://your.domain.example/`.
* The first account you create will become the site administrator (MeshCentral will mention "Server has no users, next new account will be site administrator").
* From the web UI you can create groups, generate agent installers (Windows EXE, Linux packages), and manage devices.
