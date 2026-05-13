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

## Let's Encrypt Certificate

### Generate certificate

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d popuri.fm -d www.popuri.fm
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


## Assets Caching

```nginx
server {

    ...

    location ~* \.(css|js|png|jpg|jpeg|gif|svg|ico|webp)$ {
        expires 7d;
        add_header Cache-Control "public";
    }
}
```

## Security headers
```nginx
server {

    ...

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'self';" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

## Gzip compression for performance
```nginx
server {

    ...
    
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```