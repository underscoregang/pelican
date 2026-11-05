# Pelican Panel — Secure Installation Guide

> Ubuntu 24.04 LTS · NGINX · PHP · HTTPS · private configuration
>
> **Documentation-safe:** all hostnames, addresses, IDs and credentials are placeholders. Never commit `.env`, node tokens, TLS private keys, database credentials, or real infrastructure details to a public repository.

## Architecture and prerequisites

- Dedicated, patched Ubuntu 24.04 VPS with a sudo-capable administrator; avoid daily root SSH logins.
- DNS record `panel.example.com` pointing to the server; substitute your actual hostname **locally**, not in published docs.
- Minimum capacity depends on workloads; Panel and Wings can share a machine for small deployments.
- Allow inbound 80/tcp for ACME HTTP validation and 443/tcp for the Panel. Restrict SSH (22/tcp) to trusted IPs. Do not expose database ports.
- Use the **current official Pelican release instructions** for supported PHP versions, packages and release asset names: https://pelican.dev/docs/panel/getting-started/

## 1. Prepare the host

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl ca-certificates unzip tar nginx certbot python3-certbot-nginx ufw
sudo systemctl enable --now nginx
```

Use SSH keys, disable password-based SSH only after confirming key login works, and configure automatic security updates according to your operations policy. If remote, **allow SSH before enabling UFW**:

```bash
sudo ufw allow from YOUR_TRUSTED_PUBLIC_IP to any port 22 proto tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
# Enable only after verifying SSH access and cloud firewall rules:
# sudo ufw enable
```

## 2. Install supported PHP and database dependencies

Pelican's PHP requirements can change between releases. Check the official installation page first. The example below uses PHP 8.5, but replace it with the version supported by your installed Panel release:

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update
sudo apt install -y php8.5-fpm php8.5-cli php8.5-{gd,mbstring,bcmath,xml,curl,zip,intl,sqlite3,mysql}
php -v
```

For a small instance, choose SQLite in the installer if supported by your release. For MariaDB/PostgreSQL, create a **dedicated least-privilege database user** and bind the database to localhost or a private network; never use the database root account for the application.

## 3. Install Panel release

Confirm the current release URL and checksum/signature where provided by the publisher before running commands:

```bash
sudo mkdir -p /var/www/pelican
cd /var/www/pelican
curl -fL https://github.com/pelican/panel/releases/latest/download/panel.tar.gz -o /tmp/pelican-panel.tar.gz
sudo tar -xzf /tmp/pelican-panel.tar.gz -C /var/www/pelican
```

**Do not blindly run Composer or Artisan commands from outdated guides.** Follow the installation instructions for the downloaded release. If Composer is required, install it from the official source with its published installer verification procedure: https://getcomposer.org/download/ . Run only the documented setup commands, then configure the application via the web installer.

Set ownership and writable paths:

```bash
sudo chown -R root:www-data /var/www/pelican
sudo chown -R www-data:www-data /var/www/pelican/storage /var/www/pelican/bootstrap/cache
sudo chmod -R u+rwX,g+rwX /var/www/pelican/storage /var/www/pelican/bootstrap/cache
sudo test -f /var/www/pelican/public/index.php && echo 'Panel entrypoint exists'
```

Do not make the application directory world-writable. Keep `.env` unreadable to untrusted users; ensure the PHP worker can read it as required.

## 4. NGINX — HTTP bootstrap

Create `/etc/nginx/sites-available/pelican.conf` (substitute your hostname locally):

```nginx
server {
    listen 80;
    server_name panel.example.com;
    root /var/www/pelican/public;
    index index.php;
    client_max_body_size 100m;

    location / { try_files $uri $uri/ /index.php?$query_string; }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;
    }
    location ~ /\.(?!well-known).* { deny all; }
}
```

Check the PHP-FPM socket matches the installed PHP version. Then:

```bash
sudo ln -sfn /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
sudo nginx -t && sudo systemctl reload nginx
```

Do not remove other enabled sites unless you know they are unused.

## 5. HTTPS before administrator setup

After DNS resolves and TCP 80 is reachable:

```bash
sudo certbot --nginx -d panel.example.com
sudo certbot renew --dry-run
```

Confirm HTTPS works, that HTTP redirects to HTTPS, and that the certificate matches the hostname. Access `https://panel.example.com/installer` and follow the release's installation wizard. Use a unique administrator password and MFA if available. Remove/disable installer access when the release documentation instructs.

## 6. Background jobs and post-install checks

Use the queue worker and scheduler instructions **for your specific Pelican version**; do not guess Artisan subcommands. Confirm services restart after reboot and that application logs are writable. Test:

```bash
sudo nginx -t
sudo systemctl is-active nginx
sudo systemctl is-active php8.5-fpm
curl -I https://panel.example.com/
```

A protected remote API can legitimately return 401 when called **without** credentials:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://panel.example.com/api/remote/servers
```

A 401 indicates the route is reachable but unauthenticated; it does **not** prove Wings credentials work.

## 7. Hardening and maintenance

- Keep OS, Panel, PHP, NGINX and database patched. Test upgrades and back up first.
- Restrict SSH and database access; use provider-level firewall **and** UFW.
- Use a separate least-privilege DB account; store secrets outside version control.
- Back up the database, `.env`/application key and server configurations securely; encrypt and test restores.
- Avoid publishing full `nginx -T`, `.env`, API headers, or logs containing credentials.
- Check NGINX, PHP-FPM and application logs for 404/500 errors. If CSS is missing, inspect browser Network requests and release asset manifest rather than blindly reinstalling.
- When using a reverse proxy, configure trusted proxies per Pelican's official docs; never enable insecure TLS verification.

## Troubleshooting

| Symptom | Verify |
|---|---|
| Connection refused | DNS, NGINX running, ports 80/443 and cloud firewall |
| NGINX 404 | Correct `server_name`, `/var/www/pelican/public`, `try_files`, enabled site |
| Unstyled installer | Missing assets, permissions, mixed content, asset URLs |
| 502 Bad Gateway | PHP-FPM status and matching socket |
| Remote API 401 | Expected without authentication; check token if authenticated request fails |
| Remote API 404 | Inspect application logs and node credentials; do not assume firewall issue |

**Official docs:** https://pelican.dev/docs/ · https://certbot.eff.org/ · https://docs.docker.com/engine/security/
