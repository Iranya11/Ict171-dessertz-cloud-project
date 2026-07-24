# Ict171-dessertz-cloud-project
**Student:** Iranya Dewmini
**Student ID:** 35900162
**Live site:** https://desertz.ddns.net
**Video explainer:** [link here once recorded]

# 1: Provisioning the Virtual Machine
**Author:** Iranya Dewmini · **Student ID:** 35900162
**Project:** Dessertz — ICT171 Cloud Server Project, 2026

This stage covers standing up the cloud hardware that Dessertz runs on.

## Stage 1. Create the VM
 Logged into the Azure Portal and searched for Virtual Machines.
 Clicked Create a Azure Virtual Machine and configured:

   | Setting | Value |
   |---|---|
   | Name | `171machine` |
   | Operating system | Ubuntu Server 22.04 LTS |
   | Size | `Standard B2ats v2` (2 vCPUs, 1 GiB memory) |
   | Region | `Japan East` |
   | Resource group | `1711` |

3. Under **Administrator account**, set up authentication (password or SSH key) and kept the credentials safe for the connection step.

4. Under **Networking**, opened the following inbound ports on the Network Security Group:

   | Service | Port | Protocol | Why it's needed |
   |---|---|---|---|
   | SSH | 22 | TCP | Remote management of the server |
   | HTTP | 80 | TCP | Regular web traffic |
   | HTTPS | 443 | TCP | Secure web traffic (added in Stage 4) |

Without these rules, the VM would be unreachable both for SSH management and for site visitors.

The VM was created during the project proposal assignment and is confirmed **Running**, with a public IP address of **20.89.16.246** and a private IP of `10.1.1.4` on the `171machine-vnet/default` virtual network.

📸 *Screenshot: Azure VM overview page showing OS, region, and public IP*
`<img width="1500" height="355" alt="image" src="https://github.com/user-attachments/assets/5dc926c9-2190-4bd9-a44b-ca2d40ccbf99" />


📸 *Screenshot: VM listed as "Running" in the Azure dashboard*
`<img width="1455" height="620" alt="image" src="https://github.com/user-attachments/assets/046e349d-d345-4067-9495-08ce86bc5f3e" />


## 2. Connect and update

Connected over SSH from a local terminal:

```
ssh azureuser@20.89.16.246
```

On first connection, accepted the host fingerprint prompt (`yes`) and entered the password when prompted (nothing appears on screen while typing — this is normal).

Updated the system before installing anything else:

```
sudo apt update && sudo apt upgrade -y
```

`apt update` refreshes the local package index; `apt upgrade -y` installs available updates without prompting individually.

📸 *Screenshot: successful SSH connection*
<img width="1112" height="966" alt="image" src="https://github.com/user-attachments/assets/ab446a35-7fce-4b7d-90d7-6635c813d904" />


**Next:** [Web Server Setup →](02-web-server-setup.md)

# Stage 2 — Installing and Configuring Apache

With the VM provisioned and updated, this stage installs the web server and deploys the Dessertz site files.

## 1. Install Apache2
sudo apt install apache2 -y

. Start and enable the service
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl status apache2
`start` launches Apache immediately, `enable` makes sure it comes back up automatically after a reboot, and `status` confirms it's active.

📸 *Screenshot: `systemctl status apache2` showing active (running)*
`<img width="1122" height="507" alt="image" src="https://github.com/user-attachments/assets/302dc9da-db90-4454-85c7-c6420aaf589e" />


## 3. Confirm the server is publicly reachable

Loaded `http://20.89.16.246` in a browser (not just tested locally). The default **Apache2 Ubuntu Default Page** confirmed the server was serving traffic before any custom content was added.

📸 *Screenshot: default Apache2 page loading at the public IP*
`<!-- insert screenshot here -->`

## 4. Deploy the Dessertz site files

Apache serves files from `/var/www/html/` by default, owned by `root`. Ownership was handed to the working user before uploading anything:

```
sudo chown -R azureuser:azureuser /var/www/html
```

From there, the Dessertz HTML/CSS files were transferred into `/var/www/html/` using `<!-- e.g. SFTP client such as FileZilla, or scp -->`, replacing the default `index.html`.

📸 *Screenshot: Dessertz site loading in the browser at the public IP*
`<!-- insert screenshot here -->`

---

**Previous:** [← VM Setup](01-vm-setup.md) · **Next:** [DNS Setup →](03-dns-setup.md)

# Stage 3 — Setting Up Dynamic DNS

**Author:** Iranya Dewmini · **Student ID:** 35900162

A raw IP address isn't practical to remember or share, so the VM's public IP was mapped to a proper hostname using **No-IP's** free dynamic DNS service.

## 1. Create the hostname

1. Created a free account at [No-IP.com](https://www.noip.com/).
2. Clicked **Create Hostname**, chose the name `desertz`, and selected the free `.ddns.net` extension — giving the final hostname **desertz.ddns.net**.
3. Set the **Record Type** to **A (Host)** and entered the Azure VM's public IP address, `20.89.16.246`.
4. Saved the hostname and waited for the DNS record to propagate.

📸 *Screenshot: No-IP dashboard showing desertz.ddns.net pointed at the VM's IP*
`<!-- insert screenshot here -->`

## 2. Confirm it resolves

Once propagation completed, `desertz.ddns.net` loaded the exact same content as the raw IP address.

```
ping desertz.ddns.net
```

📸 *Screenshot: desertz.ddns.net loading the Dessertz site in a browser*
`<!-- insert screenshot here -->`

## 3. Keeping the record current

**Important:** in the Azure Portal, under the VM's network interface / public IP resource, check the **Assignment** setting. Azure public IPs default to **Dynamic**, meaning the IP can change if the VM is stopped and restarted — which would silently break the `desertz.ddns.net` A record above.

- If it's still **Dynamic**, change it to **Static** in the public IP resource's Configuration blade (`171machine-ip`) so `20.89.16.246` stays fixed for the rest of the project.
- If it's already **Static**, note that here for the marker as confirmation the domain won't break mid-semester.

`<!-- state which of the two applies once you've checked -->`

---

**Previous:** [← Web Server Setup](02-web-server-setup.md) · **Next:** [SSL/TLS Setup →](04-ssl-setup.md)
