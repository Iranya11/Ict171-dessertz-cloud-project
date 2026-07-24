# Ict171-dessertz-cloud-project

**Student:** Iranya Dewmini

**Student ID:** 35900162

**Live site:** https://desertz.ddns.net

**Video explainer:** [https://drive.google.com/file/d/10LZhVpdUi3Pj2jmGauwzOPUTHc1rf2hP/view?usp=sharing]



---

# Stage 1 — Deploying the Virtual Machine
**Project:** Dessertz — ICT171 Cloud Server Project, 2026

This stage covers standing up the cloud hardware that Dessertz runs on.

## 1. Create the VM

Logged into the Azure Portal and searched for Virtual Machines.
Clicked **Create** → **Azure Virtual Machine** and configured:

| Setting            | Value                                        |
|---------------------|-----------------------------------------------|
| Name                | `171machine`                                  |
| Operating system    | Ubuntu Server 22.04 LTS                       |
| Size                | `Standard B2ats v2` (2 vCPUs, 1 GiB memory)   |
| Region              | Japan East                                    |
| Resource group      | `1711`                                        |

Under **Administrator account**, set up authentication (password or SSH key) and kept the credentials safe for the connection step.

Under **Networking**, opened the following inbound ports on the Network Security Group:

| Service | Port | Protocol | Why it's needed                              |
|---------|------|----------|-----------------------------------------------|
| SSH     | 22   | TCP      | Remote management of the server                |
| HTTP    | 80   | TCP      | Regular web traffic                            |
| HTTPS   | 443  | TCP      | Secure web traffic (added in Stage 4)          |

Without these rules, the VM would be unreachable both for SSH management and for site visitors.

The VM was created during the project proposal assignment and is confirmed **Running**, with a public IP address of **20.89.16.246** and a private IP of `10.1.1.4` on the `171machine-vnet/default` virtual network.

📸 *Screenshot: Azure VM overview page showing OS, region, and public IP*
<img width="1500" height="355" alt="image" src="https://github.com/user-attachments/assets/5dc926c9-2190-4bd9-a44b-ca2d40ccbf99" />

📸 *Screenshot: VM listed as "Running" in the Azure dashboard*
<img width="1455" height="620" alt="image" src="https://github.com/user-attachments/assets/046e349d-d345-4067-9495-08ce86bc5f3e" />

## 2. Connect and update

Connected over SSH from a local terminal:

```bash
ssh azureuser@20.89.16.246
```

On first connection, accepted the host fingerprint prompt (`yes`) and entered the password when prompted (nothing appears on screen while typing — this is normal).

Updated the system before installing anything else:

```bash
sudo apt update && sudo apt upgrade -y
```

`apt update` refreshes the local package index; `apt upgrade -y` installs available updates without prompting individually.

📸 *Screenshot: successful SSH connection*
<img width="1112" height="966" alt="image" src="https://github.com/user-attachments/assets/ab446a35-7fce-4b7d-90d7-6635c813d904" />

---

# Stage 2 — Installing and Configuring Apache

With the VM provisioned and updated, this stage installs the web server and deploys the Dessertz site files.

## 1. Install Apache2

```bash
sudo apt install apache2 -y
```

## 2. Start and enable the service

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl status apache2
```

`start` launches Apache immediately, `enable` makes sure it comes back up automatically after a reboot, and `status` confirms it's active.

📸 *Screenshot: `systemctl status apache2` showing active (running)*
<img width="1122" height="507" alt="image" src="https://github.com/user-attachments/assets/302dc9da-db90-4454-85c7-c6420aaf589e" />

## 3. Confirm the server is publicly reachable

Loaded `http://20.89.16.246` in a browser (not just tested locally). The default **Apache2 Ubuntu Default Page** confirmed the server was serving traffic before any custom content was added.

📸 *Screenshot: default Apache2 page loading at the public IP*
<img width="800" height="826" alt="image" src="https://github.com/user-attachments/assets/3fbc0622-fee0-4cc4-a0af-42910cc14fab" />

## 4. Deploy the Dessertz site files

Apache serves files from `/var/www/html/` by default, owned by `root`. Ownership was handed to the working user before uploading anything:

```bash
sudo chown -R azureuser:azureuser /var/www/html
```

From there, the Dessertz HTML/CSS files were transferred into `/var/www/html/` using `scp` from a Windows Terminal (PowerShell) session, replacing the default `index.html`:

```bash
scp -r ./dessertz-site/* azureuser@20.89.16.246:/var/www/html/
```

📸 *Screenshot: Dessertz site loading in the browser at the public IP*
<img width="1575" height="875" alt="image" src="https://github.com/user-attachments/assets/f82eac53-4e3f-4ee6-a95d-3278f9ae70da" />

---

# Stage 3 — Setting Up Dynamic DNS

A raw IP address isn't practical to remember or share, so the VM's public IP was mapped to a proper hostname using **No-IP's** free dynamic DNS service.

## 1. Create the hostname

1. Created a free account at [No-IP.com](https://www.noip.com/).
2. Clicked **Create Hostname**, chose the name `desertz`, and selected the free `.ddns.net` extension — giving the final hostname **desertz.ddns.net**.
3. Set the **Record Type** to **A (Host)** and entered the Azure VM's public IP address, `20.89.16.246`.
4. Saved the hostname and waited for the DNS record to propagate.

📸 *Screenshot: No-IP dashboard showing desertz.ddns.net pointed at the VM's IP*
<img width="1902" height="837" alt="image" src="https://github.com/user-attachments/assets/c0b8aabb-9580-40e4-831e-b042feab53c5" />

## 2. Confirm it resolves

Once propagation completed, `desertz.ddns.net` loaded the exact same content as the raw IP address.

📸 *Screenshot: desertz.ddns.net loading the Dessertz site in a browser*
<img width="1911" height="947" alt="image" src="https://github.com/user-attachments/assets/c97db6fa-5bb8-4622-86a1-d2770f758ea7" />

## 3. Keeping the record current

Azure public IPs default to **Dynamic**, meaning the IP can change if the VM is stopped and restarted — which would silently break the `desertz.ddns.net` A record above. This was checked in the public IP resource's Configuration blade (`171machine-ip`): the assignment is **Static**, so `20.89.16.246` stays fixed and the domain won't break for the rest of the project.

---

# Stage 4 — Securing the Server with HTTPS (SSL/TLS)

With the domain resolving correctly, the next step was encrypting traffic between visitors and the server. A free SSL/TLS certificate was installed using **Let's Encrypt** via the **Certbot** utility, so the site could be served securely over HTTPS rather than plain HTTP.

## 1. Confirm port 443 is open

The HTTPS rule (port 443, TCP) was already added to the Network Security Group back in Stage 1, in preparation for this step.

## 2. Install Certbot

```bash
sudo apt install certbot python3-certbot-apache -y
```

## 3. Generate and install the certificate

```bash
sudo certbot --apache -d desertz.ddns.net
```

Certbot prompted for an email address for renewal/security notices, agreement to the Let's Encrypt Terms of Service, and whether to redirect all HTTP traffic to HTTPS — the redirect option was selected so all traffic is automatically forced onto the secure connection.

## 4. Verify the certificate

```bash
sudo certbot certificates
```

Output confirmed the certificate is correctly issued and installed:

```
Certificate Name: desertz.ddns.net
  Serial Number: 5bb36451f93527ff6def9dacc21365345a5
  Key Type: RSA
  Domains: desertz.ddns.net
  Expiry Date: 2026-09-16 19:53:07+00:00 (VALID: 54 days)
  Certificate Path: /etc/letsencrypt/live/desertz.ddns.net/fullchain.pem
  Private Key Path: /etc/letsencrypt/live/desertz.ddns.net/privkey.pem
```

## 5. Confirm auto-renewal is active

Let's Encrypt certificates are valid for 90 days, so Certbot installs a systemd timer that checks for renewal automatically. This was confirmed active:

```bash
sudo systemctl status certbot.timer
```

The timer runs twice daily and has been active since the certificate was first installed (18 June 2026). The Certbot log (`/var/log/letsencrypt/letsencrypt.log`) shows repeated automatic renewal checks over the following weeks, each correctly reporting the certificate is "not yet due for renewal" — confirming the auto-renewal mechanism is working as expected ahead of the certificate's 16 September 2026 expiry.

## 6. Verify the secure connection

Navigating to `https://desertz.ddns.net` shows the padlock icon in the browser address bar, confirming the connection is encrypted end-to-end.

📸 *Screenshot: padlock/certificate details on desertz.ddns.net*
<img width="1545" height="845" alt="image" src="https://github.com/user-attachments/assets/e2d2dbfe-feb4-4ae6-8b74-e9334b5270e0" />

---

# Stage 5 — Automated Site Backup Script

To protect the Dessertz site content against accidental loss or server issues, a custom bash script was written to automatically back up `/var/www/html` on a schedule, with old backups cleaned up automatically to save disk space.

## 1. What the script does

1. **Snapshotting** — Compresses the entire contents of `/var/www/html` into a single timestamped `.tar.gz` archive.
2. **Logging** — Records the outcome of every run (success or failure), with a timestamp, to `/home/azureuser/backup.log`.
3. **Cleanup** — Automatically deletes any backup archive older than 7 days, keeping the VM's storage from filling up over time.

## 2. The script

Created at `/usr/local/bin/backup-dessertz.sh`:

```bash
#!/bin/bash
# Dessertz site backup script
# Backs up /var/www/html and removes backups older than 7 days

BACKUP_DIR="/home/azureuser/backups"
SITE_DIR="/var/www/html"
TIMESTAMP=$(date +%Y-%m-%d_%H-%M-%S)
LOG_FILE="/home/azureuser/backup.log"

mkdir -p "$BACKUP_DIR"

tar -czf "$BACKUP_DIR/dessertz-backup-$TIMESTAMP.tar.gz" -C "$SITE_DIR" .

if [ $? -eq 0 ]; then
    echo "$(date): Backup succeeded -> dessertz-backup-$TIMESTAMP.tar.gz" >> "$LOG_FILE"
else
    echo "$(date): Backup FAILED" >> "$LOG_FILE"
fi

find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
```

Made executable with:

```bash
sudo chmod +x /usr/local/bin/backup-dessertz.sh
```

## 3. Manual test run

The script was first run manually to confirm it worked correctly before automating it:

```bash
sudo /usr/local/bin/backup-dessertz.sh
ls -lh /home/azureuser/backups
cat /home/azureuser/backup.log
```

Output confirmed a successful backup:

```
total 36K
-rw-r--r-- 1 root root 33K Jul 24 09:47 dessertz-backup-2026-07-24_09-47-52.tar.gz
Fri Jul 24 09:47:52 UTC 2026: Backup succeeded -> dessertz-backup-2026-07-24_09-47-52.tar.gz
```

📸 *Screenshot: manual script run and log output*
<img width="940" height="194" alt="image" src="https://github.com/user-attachments/assets/962dd235-58e8-4837-979b-f742f852c7b6" />

## 4. Scheduling with cron

To run automatically without manual intervention, the script was scheduled via `crontab` to run daily at 2am:

```bash
crontab -e
```

Added:

```bash
0 2 * * * /usr/local/bin/backup-dessertz.sh
```

Confirmed the job was saved correctly:

```bash
crontab -l
```

📸 *Screenshot: `crontab -l` confirming the scheduled job*
<img width="940" height="655" alt="image" src="https://github.com/user-attachments/assets/28a5512b-f8c6-46d0-8ca3-4f28c2eee181" />

## 5. Verifiable output

Together, the log file (`/home/azureuser/backup.log`) and the growing set of timestamped archives in `/home/azureuser/backups` provide an ongoing, checkable record that the backup system is running as intended — both from a manual trigger and from the scheduled cron job.

## 🚧 Planned Improvements

The script currently covers `/var/www/html` only. Future additions could include:

- Storing backups somewhere off the VM (e.g. Azure Blob Storage), so a copy survives even if the server is lost entirely.
- Adding alerts (email or webhook) when a backup fails, instead of relying solely on the log.
- Including Apache's configuration files in the backup, not just the website content.
---

## 🍰 Dessertz and How It Functions

Dessertz is built around three core pillars that bring global dessert culture together in one place:

1. **Regional Explorer** — Lets visitors browse desserts by country or cultural region, surfacing the traditions and history behind each one.
2. **Recipe Library** — A searchable collection of step-by-step guides for recreating traditional confections at home, filterable by ingredient or origin.
3. **Cultural Stories** — Longer-form pieces exploring the history and meaning behind iconic desserts from around the world — the heritage and identity woven into each dish, not just the recipe.

The current deployment establishes the foundation for these three features — a static site hosted on Azure IaaS, running Ubuntu 22.04 and Apache — with the Regional Explorer, Recipe Library, and Cultural Stories modules marked as planned and in active development.

### Homepage view

<img width="1851" height="822" alt="image" src="https://github.com/user-attachments/assets/06b83798-fddf-4b42-9bc6-0e471de52629" />

## 📊 Platform Overview

| Feature            | Description                                              | Status    |
|--------------------|------------------------------------------------------------|-----------|
| Regional Explorer  | Discover desserts by country or region                    | Planned   |
| Recipe Library     | Step-by-step, searchable recreation guides                 | Planned   |
| Cultural Stories   | Deep dives into dessert history and meaning                 | Planned   |
| Infrastructure     | Azure IaaS, Ubuntu 22.04 LTS, Apache HTTP Server            | Live      |

### Capabilities section

<img width="1542" height="862" alt="image" src="https://github.com/user-attachments/assets/175f7361-a974-42d5-8b8e-e401c73159b7" />
<img width="1565" height="746" alt="image" src="https://github.com/user-attachments/assets/a499c731-10b2-47ea-9839-1b588c5e5955" />
<img width="1457" height="732" alt="image" src="https://github.com/user-attachments/assets/e778f0b3-a7ab-4935-9f62-6a21ffcbfbf2" />

## 🚧 Future of Dessertz

Dessertz is currently a static foundation site, with its main features — Regional Explorer, Recipe Library, and Cultural Stories — still in planning. Upcoming work will focus on turning these from planned modules into working features, starting with a browsable regional dessert index and expanding into the recipe and story content from there.

Beyond the technical build, Dessertz aims to make global food culture more discoverable — using dessert as a lens into heritage, community, and identity across different societies.

---
## 👥 Author & License

**Iranya Dewmini** (Student ID: `35900162`) — *ICT171 Student*

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.
