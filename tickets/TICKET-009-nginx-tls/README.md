# TICKET-009: Nginx + TLS Deployment

**Status:** Resolved  
**Environment:** Ubuntu 26.04 LTS, VirtualBox VM, 2 vCPU / 2GB RAM, connecting from a WSL (Ubuntu) host on Windows  
**Date:** 10/22/2026  
**Skills practiced:** Nginx, TLS/SSL, web server configuration

## 1. Problem Statement
Needed to deploy a web service with encrypted (HTTPS) traffic instead of
plaintext HTTP.

## 2. Investigation
This was a build ticket, but the first verification returned
`403 Forbidden` instead of `200 OK`, so I investigated: 

Output of `$ls -l /var/www/html/`:

<pre>
total 4
-rw-r--r-- 1 root root 615 Oct 14 19:38 index.nginx-debian.html
</pre>

Output of `$sudo tail -n 5 /var/log/nginx/error.log`:

<pre>
2026/10/22 20:37:44 [error] 4154#4154: *1 directory index of "/var/www/html/" is forbidden, client: 127.0.0.1, server: , request: "HEAD / HTTP/1.1", host: "localhost"
</pre>

## 3. Root Cause
My config replaced Ubuntu's default site file and dropped its `index`
directive. Ubuntu's placeholder page is `index.nginx-debian.html`, not
`index.html`, so Nginx found no index file and, with directory listing off
by default, returned 403. TLS and the redirect were working; only the
content lookup failed.

## 4. Fix
Generated a self-signed TLS certificate and configured Nginx to serve
HTTPS on port 443 with an automatic redirect from HTTP (port 80), and added
`index index.html index.nginx-debian.html;` to the HTTPS server block:

<pre>
$sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key -out /etc/ssl/certs/selfsigned.crt \
  -subj "/CN=web-01.local"
</pre>

And the server block config:

<pre>
$sudo tee /etc/nginx/sites-available/default <<'done'
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/selfsigned.key;
    root /var/www/html;
    index index.html index.nginx-debian.html;
}
server {
    listen 80;
    return 301 https://$host$request_uri;
}
done
</pre>

## 5. Verification

Output of `$sudo nginx -t`:

<pre>
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
</pre>

Output of `$curl -k -I https://localhost`:

<pre>
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Thu, 22 Oct 2026 17:46:36 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 14 Oct 2026 16:38:26 GMT
Connection: keep-alive
ETag: "6acfb002-267"
Accept-Ranges: bytes
</pre>

## 6. Prevention / Notes
Documented the redirect block so all future sites on this box default to
HTTPS-only. Always run `sudo nginx -t` before restarting, and when
overwriting a distro default config, check what the original contained
(like the `index` line) before discarding it.

## 7. What I'd do differently in production
I'd use Let's Encrypt (via certbot) for a real publicly trusted
certificate instead of self-signed, and automate its renewal.
