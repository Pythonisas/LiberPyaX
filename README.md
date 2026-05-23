# LiberPyaX

Backend-first starter notes for a Libervia-based deployment.

## Resources

- Libervia website: https://libervia.org
- NGI issue reference: https://github.com/ngi-nix/projects/issues/4

## KISS install guide (Debian 13 + Nginx)

This guide keeps scope on backend setup only.

### 1) Base system prep

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git nginx certbot python3-certbot-nginx ufw
sudo timedatectl set-timezone UTC
```

Create an admin user if needed and keep root SSH disabled in production.

### 2) Basic firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

### 3) DNS basics (required before HTTPS)

At your DNS provider, create records that point to your Debian server public IP:

- `A` record: `example.org` -> `YOUR_SERVER_IPV4`
- `A` record: `xmpp.example.org` -> `YOUR_SERVER_IPV4` (if you separate hostnames)
- Optional `AAAA` records for IPv6

Wait for DNS propagation, then verify:

```bash
dig +short example.org
```

### 4) Install and enable Nginx

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

### 5) Reverse proxy skeleton for backend

Create a simple Nginx server block:

```nginx
server {
    listen 80;
    server_name example.org;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save as `/etc/nginx/sites-available/liberpyax.conf`, then enable it:

```bash
sudo ln -s /etc/nginx/sites-available/liberpyax.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 6) TLS with Let's Encrypt

```bash
sudo certbot --nginx -d example.org
```

Certbot updates Nginx config and installs auto-renewal timers. Verify:

```bash
sudo systemctl list-timers | grep certbot
```

### 7) Backend service notes

Run Libervia/Sat backend as a dedicated system user and bind it to localhost (for example `127.0.0.1:8080`), letting Nginx expose public HTTP/HTTPS.

Typical production pattern:

- app process managed by `systemd`
- app bound to localhost only
- Nginx handles TLS, headers, and public ingress

### 8) Minimal post-install checks

```bash
curl -I http://127.0.0.1:8080
curl -I https://example.org
sudo journalctl -u nginx --no-pager -n 100
```

If backend check fails locally, fix the app service first. If local works but domain fails, check DNS and Nginx server_name/routing.
