+++
categories = ["Linux"]
date = "2012-07-12T22:57:00+01:00"
description = "CentOS/RedHat: Secured phpMyAdmin on Nginx with SSL only"
tags = ["nginx", "phpmyadmin"]
title = "CentOS/RedHat: Secured phpMyAdmin on Nginx with SSL only"
slug = "centosredhat-secured-phpmyadmin-on-nginx-with-ssl-only"
+++

#### Introduction:

This tutorial shows how to setup phpMyAdmin running on Nginx with SSL only. As a extra security measure we’ll limit access to the server by IP and we’ll use web authentication.

#### Variables:

```
OS: CentOS 6.3 64bit
Vhost for phpMyAdmin: pma.example.com
```

We need some additional repositories. You can also use “yum-priorities” package available on CentOS – the Yum Priorities plugin can be used to enforce ordered protection of repositories, by associating priorities to repositories. Visit the Yum Priorities CentOS Wiki for more information.

#### Installing extra repositories:

*Extra Packages for Enterprise Linux (EPEL)*

```sh-session
# rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-7.noarch.rpm
```

*Remi repository*

```sh-session
# rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
```

*Nginx repository*

```sh-session
# rpm  -Uvh http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
```

Enable Remi Repository:

**/etc/yum.repos.d/remi.repo**

```ini
[remi]
...
enabled=1
...
```

We need to exclude some packages from CentOS base repository:

**/etc/yum.repos.d/CentOS-Base.repo**

```ini
[base]
 ...
 exclude=php* mysql*
 ...
 [updates]
 ...
 exclude=php* mysql*
 ...
 [extras]
 ...
 exclude=php* mysql*
 ...
```

We need to also exclude Nginx package from EPEL repository:

**/etc/yum.repos.d/epel.repo**

```ini
[epel]
...
exclude=nginx*
...
```

#### Install required packages:

```sh-session
yum install -y nginx php-fpm phpmyadmin php-pecl-apc  crypto-utils
```

YUM will take care of all dependencies.

#### PHP-FPM configuration (PHP-FPM is running on UNIX File Sockets to avoid TCP overhead):

**/etc/php-fpm.d/www.conf**

```ini
[www]
 listen = /var/run/php-fpm/$pool.sock
 listen.allowed_clients = 127.0.0.1
 listen.owner = nginx
 listen.group = nginx
 listen.mode = 0640
 user = nginx
 group = nginx
 pm = dynamic
 pm.max_children = 50
 pm.start_servers = 5
 pm.min_spare_servers = 5
 pm.max_spare_servers = 35
 slowlog = /var/log/php-fpm/www-slow.log
 php_admin_value[error_log] = /var/log/php-fpm/www-error.log
 php_admin_flag[log_errors] = on
```

#### Nginx configuration:

**/etc/nginx/nginx.conf**

```nginx
user  nginx;
 worker_processes  4;

 pid        /var/run/nginx.pid;

events {
 worker_connections  1024;
 }

http {
 include       /etc/nginx/mime.types;
 default_type  application/octet-stream;

 log_format  main  '$remote_addr $host $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $ssl_cipher $request_time';
 access_log  /var/log/nginx/access.log  main buffer=32k;
 error_log  /var/log/nginx/error.log warn;

## Timeouts
 keepalive_timeout       60 60;

## General Options
 charset                 utf-8;
 ignore_invalid_headers  on;
 max_ranges              0;
 recursive_error_pages   on;
 sendfile                on;
 server_tokens           off;
 source_charset          utf-8;

## Global SSL options
 ssl_prefer_server_ciphers on;
 ssl_session_cache shared:SSL:10m;

## Compression
 gzip              on;
 gzip_static       on;
 gzip_buffers      16 8k;
 gzip_vary         on;
 gzip_types text/plain text/css application/json application/x-javascript application/xml application/xml+rss text/javascript image/png image/jpeg image/jpg image/gif text/x-component text/richtext image/svg+xml text/xsd text/xsl text/xml image/x-icon;

 ## HTTP to HTTPS redirect
 server {
 listen [::]:80;

 root  /var/empty;
 server_name pma.exmaple.com;
 rewrite ^ https://pma.example.com$request_uri permanent;
}

server {

add_header  Cache-Control "public, must-revalidate, proxy-revalidate";
 add_header Pragma "public";
 listen      listen [::]:443;
 root        /usr/share/phpMyAdmin;
 index index.php index.html index.htm;
 server_name pma.example.com;

## SSL Certs
 ssl on;
 ssl_certificate /etc/pki/tls/certs/pma.example.com.crt;
 ssl_certificate_key /etc/pki/tls/private/pma.example.com.key;

}

location ^~ / {
 try_files $uri $uri/ /index.php?$args;
 ## Restricted Access directory
 auth_basic      "Authorized Access Only!";
 auth_basic_user_file  .users;
 ## Hosts restrictions
 allow 192.168.1.1;  #Specify your IP here
 deny all;

location ~ \.php$ {
 fastcgi_pass unix:/var/run/php-fpm/www.sock;
 fastcgi_index index.php;
 fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
 include fastcgi_params;
 fastcgi_param HTTPS on;
 }

}

location /usr/share/phpMyAdmin/libraries {
 deny all;
 }
 location /usr/share/phpMyAdmin/setup/frames {
 deny all;
 }
 location /usr/share/phpMyAdmin/setup/lib {
 deny all;
 }

## deny access to .htaccess files
 location ~ /\. {
 access_log off;
 log_not_found off;
 deny  all;
 }
include /etc/nginx/conf.d/*.conf;

}
```

#### Create self-signed SSL certificate (This will start the genkey graphical user interface):

```
# genkey pma.example.com --days 365
```

Add the following line to phpMyAdmin config file to disable version check as this breaks the SSL connection:

**/etc/phpMyAdmin/config.inc.php**

```php
$cfg['VersionCheck'] = false;
```

#### Create user for web authentication:

```sh-session
# htpasswd -mc /etc/nginx/.users username
```

Change the owner and permissions so only “nginx” user can read the file:

```
# chown nginx /etc/nginx/.users && chmod 600 /etc/nginx/.users
```

#### Start nginx and php-fpm daemons:

```sh-session
# chkconfig nginx on && service nginx start
# chkconfig php-fpm on && service php-fpm start
```

Done!
