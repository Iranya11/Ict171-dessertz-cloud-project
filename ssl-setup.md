# SSL/TLS Setup

The Dessertz server runs on Apache 2 (Ubuntu 22.04 LTS, Azure IaaS VM). HTTPS is enabled using **Certbot**, which requests and installs a free certificate from **Let's Encrypt** and configures Apache automatically.

Certbot was chosen because it's free, widely supported, integrates directly with Apache via the `python3-certbot-apache` plugin, and handles renewal automatically — no manual cert management required.

Let's Encrypt validates domain ownership by checking that the domain resolves to your server's public IP. (see [DNS Setup](desertz.ddns.net)) — if `desertz.ddns.net` doesn't yet point to `20.89.16.246`, Certbot's validation will fail.

## Installation

```bash
sudo apt update
sudo apt install certbot python3-certbot-apache -y

## Requesting the certificate
sudo certbot --apache -d desertz.ddns.net


Certbot will prompt for an email address (for renewal/expiry notices), ask you to agree to the Let's Encrypt terms, and ask whether to redirect HTTP traffic to HTTPS — select the redirect option so all traffic is served securely.

## What Certbot changes automatically

Certbot edits the Apache config to add a new SSL virtual host (typically `/etc/apache2/sites-available/000-default-le-ssl.conf`), pointing to the issued certificate and key:

<VirtualHost *:443>
    ServerName desertz.ddns.net
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/desertz.ddns.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/desertz.ddns.net/privkey.pem
</VirtualHost>

It also enables the required Apache modules (`ssl`, `headers`) and opens port 443.

## Verifying it works

Browser check: visiting `https://desertz.ddns.net` shows a padlock icon with no warnings.

Command-line check:
curl -vI https://desertz.ddns.net

This should return `HTTP/2 200` (or `HTTP/1.1 200`) along with certificate details showing `issuer: Let's Encrypt` and a valid expiry date.

> **[TODO: paste your actual `curl -vI` output here, or a screenshot of the padlock/certificate details in the browser — this is the "proof it works" evidence the rubric is looking for]**

## Renewal

Let's Encrypt certificates expire every 90 days. Certbot installs a systemd timer that handles renewal automatically — no manual action needed. You can confirm it's active with:

```bash
sudo systemctl status certbot.timer
```

A dry run can be used to test that renewal will succeed without actually renewing:

```bash
sudo certbot renew --dry-run
```

> **[TODO: confirm the timer is enabled on your VM, or note if you're handling renewal manually]**
