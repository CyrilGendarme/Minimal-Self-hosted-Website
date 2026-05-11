# Minimal Self-Hosted Website on a VPS

## Open Required Ports

```bash
ufw allow OpenSSH
ufw allow 80
ufw allow 443
sudo ufw allow 3389/tcp  # Only if using Remote Desktop
ufw enable
````

---

## Install and Start Nginx

```bash
apt update && apt upgrade -y
apt install -y nginx ufw

systemctl enable nginx
systemctl start nginx
```

---

## Create a "Hello World" Website

### Create directory

```bash
mkdir -p /var/www/hello
```

### Create HTML file

```bash
nano /var/www/hello/index.html
```

Paste:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Hello VPS</title>
  <link rel="stylesheet" href="style.css">
</head>

<body>
  <h1>Hello from VPS</h1>
</body>
</html>
```

---

### Create CSS file

```bash
nano /var/www/hello/style.css
```

Paste:

```css
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #0f172a;
  color: #e2e8f0;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  text-align: center;
}

h1 {
  font-size: 3rem;
  color: #38bdf8;
}
```

---

## Self-Signed SSL Certificate (Quick Proof of Concept)

> For production, use Let's Encrypt with a domain name.

### Generate certificate

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/selfsigned_poc_hello_world_webpage.key \
-out /etc/ssl/certs/selfsigned_poc_hello_world_webpage.crt
```

---

## Configure Nginx for IP Access

```bash
nano /etc/nginx/sites-available/hello
```

Paste:

For http:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/hello;
    index index.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

For https (SSL):
```nginx
server {
    listen 443 default_server;
    listen [::]:443 default_server;

    root /var/www/hello;
    index index.html;

    server_name _;

    ssl_certificate /etc/ssl/certs/selfsigned_poc_hello_world_webpage.crt;
    ssl_certificate_key /etc/ssl/private/selfsigned_poc_hello_world_webpage.key;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
---

## Enable Site

```bash
ln -s /etc/nginx/sites-available/hello /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default

systemctl restart nginx
```

---

## Access Your Website

Open in your browser:

```
https://YOUR_IP
```

> Your browser will show a **"Not Secure"** warning — this is expected with a self-signed certificate.

## Redirect http to https :

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name domain.com www.domain.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    server_name domain.com www.domain.com;

    ssl_certificate /etc/ssl/certs/selfsigned_poc_hello_world_webpage.crt;
    ssl_certificate_key /etc/ssl/private/selfsigned_poc_hello_world_webpage.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    # LIGHT HOMEPAGE
    root /var/www/domain;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```