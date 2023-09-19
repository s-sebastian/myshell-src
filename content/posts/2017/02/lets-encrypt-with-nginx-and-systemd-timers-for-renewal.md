+++
date = "2017-02-21T11:15:43Z"
title = "Let's encrypt with Nginx and systemd timers for renewal"
tags = ["certbot", "nginx", "systemd"]
description = "Let's encrypt with Nginx and systemd timers for renewal"
categories = ["Linux"]
+++

[Let's Encrypt](https://letsencrypt.org/ "Let's Encrypt") is a free, automated, and open certificate authority utilizing the [ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment "ACME") protocol.

We'll use [certbot](https://certbot.eff.org/ "Certbot"), the official client and [systemd timers](https://www.freedesktop.org/software/systemd/man/systemd.timer.html "systemd.timer") to automate the renewal of our Let's Encrypt certificates.

#### Cerbot installation:

If you are using CentOS/RHEL 7, Certbot is available in [EPEL](https://fedoraproject.org/wiki/EPEL "EPEL") (Extra Packages for Enterprise Linux). In order to use Certbot, we must first enable the EPEL repository:

```sh-session
# yum install -y epel-release
```

Next, install Certbot itself:

```sh-session
# yum install -y certbot
```

#### Nginx:

Create webroot directory:

```sh-session
# mkdir -p /var/www/letsencrypt
```

Add this location to Nginx configuration file for your domain:

**/etc/nginx/conf.d/example.conf**

```nginx
...
location ^~ /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        default_type "text/plain";
        try_files $uri =404;
        }
...
```

#### Creating certificate with Certbot:

When using the webroot method the Certbot client places a challenge response inside `/var/www/letsencrypt/.well-known/acme-challenge/` which is used for validation. That's why we've created this Nginx location above so it's accessible via HTTP.

(You can use `--dry-run` switch to test the configuration first)

```sh-session
# certbot certonly --webroot -w /var/www/letsencrypt -d example.com -d www.example.com
```

If the certificate was created successfully you can update Nginx configuration:

**/etc/nginx/conf.d/example.conf**


```nginx
server {
        server_name example.com www.exmaple.com;
        listen 80;
	root /var/empty;
	return 301 https://example.com$request_uri;
}

server {
        server_name example.com www.exmaple.com;
	listen 443 ssl http2;
	root /var/www/example.com;
	index index.html index.htm;

        ssl_protocols TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_stapling on;
        ssl_stapling_verify on;
	ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  	ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

location ^~ /.well-known/acme-challenge/ {
	root /var/www/letsencrypt;
	default_type "text/plain";
	try_files $uri =404;
	}

}
```

#### Systemd:

We'll use certbot `--renew-hook` option to reload Nginx only if certificate is renewed successfully.

There is also a comment for systemd switch to do the same thing but it will reload Nginx every time certbot runs.

**/etc/systemd/system/certbot.service**

```systemd
[Unit]
Description=Renew Let's Encrypt certificates
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --renew-hook "/bin/systemctl --no-block reload nginx" --quiet --agree-tos
#ExecStopPost=/bin/systemctl --no-block reload nginx
```

Add a timer to renew the certificates daily, including a randomized delay so that requests for renewal are spread over the day.

**/etc/systemd/system/certbot.timer**

```systemd
[Unit]
Description=Daily renewal of Let's Encrypt's certificates

[Timer]
OnCalendar=daily
RandomizedDelaySec=1day
Persistent=true

[Install]
WantedBy=timers.target
```

Activate and enable the timer:

```sh-session
# systemctl daemon-reload
# systemctl start certbot.timer
# systemctl enable certbot.timer
```

You can verify that the timer has been started with this command:

```sh-session
# systemctl list-timers certbot.timer
NEXT                         LEFT       LAST                         PASSED       UNIT          ACTIVATES
Tue 2017-02-21 14:08:16 GMT  47min left Mon 2017-02-20 10:04:01 GMT  1 day 3h ago certbot.timer certbot.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

By default certificates should be renewed 30 days before the expiry date, you can change this option in the config file, for instance:

**/etc/letsencrypt/renewal/example.com.conf**

```ini
...
renew_before_expiry = 10 days
...
```
