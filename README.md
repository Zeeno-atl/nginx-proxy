# Disclaimer

Open-sourcing components that were developed in Rechip s.r.o. which is now dead.

# Nginx image

Extended nginx docker image with nginx modules and basic configuration for better security.

## -V
nginx version: nginx/1.22.1
built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
built with OpenSSL 1.1.1n  15 Mar 2022
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -ffile-prefix-map=/data/builder/debuild/nginx-1.22.1/debian/debuild-base/nginx-1.22.1=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'

## Features

- Brotli module for brotli compression
- Custom configuration for setting up proxy headers properly in file `/etc/nginx/include/proxy_headers.conf`
- Custom configuration for ssl in file `/etc/nginx/include/proxy_headers.conf`
- Custom parent nginx.conf in file `/etc/nginx/nginx.conf`
- Prometheus endpoint to gather metrics on the same port, `/nginx-metrics` endpoint, protected by .htpasswd

## How to use

docker-compose.yml:

```lang=yml, name=docker-compose.yml

networks:
  mynetwork:
    driver: bridge

services:

  nginx:
    image: docker.rechip.eu/rechip/nginx-proxy:latest # <-- use the image
    restart: unless-stopped
    ports:
      - '1080:80' # <-- bind your ports
      - '10443:443' # <-- bind your ports
    volumes:
      - '/.../metrics_password.htpasswd:/etc/nginx/conf.d/metrics.htpasswd:ro'
      - ./default.conf:/etc/nginx/conf.d/default.conf # <-- use your default.conf
      - '/etc/letsencrypt/.../:/etc/nginx/certs/:ro' # <-- use your certs (do not reference soft/hard links)
    networks:
      - mynetwork # <--

  myapp:
    # ... run an app on port 8080 ...
    networks:
      - mynetwork
```

default.conf:

```lang=conf, name=default.conf

include include/prometheus_init.conf; # <-- Initialize prometheus counters

server {
  listen 80;
  listen [::]:80;
  server_name server.rechip.eu; # <-- your hostname
  server_tokens off;

  location / {
    rewrite ^ https://$host$request_uri? permanent;
  }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name server.rechip.eu; # <-- your hostname

  include include/ssl.conf; # <-- use certs/https

  include include/prometheus_location.conf; # <-- Expose /nginx-metrics Prometheus endpoint
  
  location / {
    try_files $uri @app;
  }

  location @app {
    proxy_pass http://myapp:8080; # <--
    
    proxy_hide_header X-Powered-By;

    proxy_http_version  1.1;
    proxy_cache_bypass  $http_upgrade;

    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        "upgrade";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host  $host;
    proxy_set_header X-Forwarded-Port  $server_port;

    include include/security_headers.conf; # <-- use security_headers.conf
  }
}

```