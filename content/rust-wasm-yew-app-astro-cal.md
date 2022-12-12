+++
title = "Rust WASM App for Moonrise Times"
date = 2022-12-07
image = "assets/flower_in_rocks.jpg"
image_alt = "Picture of flowers blooming amount rocks"
summary = """
Building a Rust WASM app with Yew to generate calendar events for astrological events such as Moonrise/Moonset.
"""

[taxonomies]
tags = ["rust", "terraform", "bulma", "elasticsearch"]
+++

[source code](github.com/whynotcats/astro-events) [view site](astro-events.whynotcats.com)

# Externalizing memory

One my on-going projects is to externalize as much of my memory as I can. I've been doing this with services like [Pinboard](pinboard.in) for recipes and blog posts, and [1password](1password.com) for passwords, software licenses, etc. One more thing I would like to be able to quickly know _and_ get notified on is when moonrise is. I've been fortunate enough to by chance have seen very cool moonrises such as a full-moon rising behind Mt Baker and a harvest moon over the Vancouver skyline. Both times I didn't have my camera on me, so I'd love to be able to better plan for it.

There are a number of apps that have information about ideal photo conditions and astrological events such as [Photo Pills](https://www.photopills.com/) and [Night Sky](https://apps.apple.com/us/app/night-sky/id475772902). Each of these have an element of what I'm looking for, though none have everything, and all are not incentivized to develop and support the features I want.
Having to switch between different apps and relying on app notifications is not something I want either, and for something that happens at predictable intervals/times calendar events should be more than enough. Calendars are an easy thing to check in the morning to see what obligations I have and at work it's a great way for coworkers to book time with me without having to go back a forth a bunch (in work cultures that emphasize using and respecting each others time).

## Spec

For an MVP generating an `.ics` file which can be imported into Google Calendar or iCalendar should be good enough. Everything should be in Rust as much as possible. Fundementally I think that Rust is a major step forward in how we code, like going from punch cards to assembler code.

## Frontend Framework

Being experienced with React something with similar concepts would be easiest to start with. Yew seems to fit the bill, with Struct / Functional Components when you want to use internal state or not, hooks, props, etc

## Geolocation Data

To calculate the times for moonrise/set we need a geo location that we'll be viewing from. A lot of applications will query an API such as Google Maps to convert something like your address or town into a lat-long pair. Often these will have a free tier of a number of queries and will return probably more information than you need. I first looked at [GeoNames](geonames.org) since they had an API but realized they offered [data dumps](http://download.geonames.org/export/dump/) of geo data that I could just use locally. THe data is simple enough, a file of geolocation data and a few meta file to enrich the data. For quick access to a relatively static dataset I considered trying redis server's [full text search](https://redis.com/redis-best-practices/indexing-patterns/full-text-search/) indexing but decided to go with a more familiar elasticsearch, since I can just dump documents and get full indexing. I'll also likely need a quick search index for future projects, so it makes sense for me to set it up.

I setup a small CLI app to load the data into elasticsearch from the files ([source code](github.com/whynotcats/admin)). This data isn't likely to change very often, so developing a process to automatically and/or incrementally update the data shouldn't be necessary. Most people would only want the precision of their city, and cities aren't created/moved that often.

# Infra with Terraform

![System Aritechture](assets/architecture_no_bg.svg)

To setup the servers I am using Digital ocean and terraform to manage the resources. Infra as code is another way of externalizing data about my pet servers. In addition to setting up the necessary `.tf` files I created markdown files listing all the commands I used to setup the server which I will automated at some point.

## Issues with Openssl

If possible enable features to remove openssl from your dependencies. Trying to develop on an OS that isn't the one on your server will give you so many headaches when trying to deploy. Even trying to use Docker to get a consistent build process, or [cross]() which is a nice wrapper around docker, gave me so many issues. I only got cross to work when I fully removed OpenSSL.

TODO: Add example features

## Elasticsearch

```sh
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
apt update
apt upgrade
# update /etc/elasticsearch/elasticsearch.yml to have network host be private ip of server <10.137.64.146>
# update /etc/elasticsearch/elasticsearch.yml to have discovery.type: single-node
systemctl daemon-reload
systemctl enable elasticsearch.service
systemctl start elasticsearch
open port to only the api
ufw enable
ufw allow ssh
ufw allow from 10.137.64.147 to any port 9200
ufw allow from 138.197.128.155 to any port 9200
ufw deny 9200
```

## Domain + Haproxy + Letsencrypt + Nginx

For SSL termination and reverse proxying I like to use Haproxy + Letsencrpyt.

For static sites I looked at using a static file server such as [miniserve](https://github.com/svenstaro/miniserve) or [binserver]() but for simlplicity -- and knowing that I will be have a number of static sites to host in the future -- I went with nginx.

## Nginx

Use nginx for serving multiple static sites

## Setup

```sh
apt update && apt upgrade && apt install nginx
ufw enable
ufw allow ssh
ufw allow from 10.137.64.148 to any port 80
ufw deny 80
```

## Conf

configurations in /etc/nginx/sites-available
data in /var/www/<subdomain>

#### Base conf

```ruby
# you must set worker processes based on your CPU cores, nginx does not benefit from setting more than that
worker_processes auto; #some last versions calculate it automatically

# number of file descriptors used for nginx
# the limit for the maximum FDs on the server is usually set by the OS.
# if you don't set FD's then OS settings will be used which is by default 2000
worker_rlimit_nofile 10000;

# only log critical errors
error_log /var/log/nginx/error.log crit;

# provides the configuration file context in which the directives that affect connection processing are specified.
events {
    # determines how much clients will be served per worker
    # max clients = worker_connections * worker_processes
    # max clients is also limited by the number of socket connections available on the system (~64k)
    worker_connections 1000;

    # optimized to serve many clients with each thread, essential for linux -- for testing environment
    use epoll;

    # accept as many connections as possible, may flood worker connections if set too low -- for testing environment
    multi_accept on;
}

http {
    # cache informations about FDs, frequently accessed files
    # can boost performance, but you need to test those values
    open_file_cache max=20000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # to boost I/O on HDD we can disable access logs
    access_log off;

    # copies data between one FD and other from within the kernel
    # faster than read() + write()
    sendfile on;

    # send headers in one piece, it is better than sending them one by one
    tcp_nopush on;

    # reduce the data that needs to be sent over network -- for testing environment
    gzip on;
    # gzip_static on;
    gzip_min_length 10240;
    gzip_comp_level 1;
    gzip_vary on;
    gzip_disable msie6;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types
        # text/html is always compressed by HttpGzipModule
        text/css
        text/javascript
        text/xml
        text/plain
        text/x-component
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/rss+xml
        application/atom+xml
        font/truetype
        font/opentype
        application/vnd.ms-fontobject
        image/svg+xml;

    # allow the server to close connection on non responding client, this will free up memory
    reset_timedout_connection on;

    # request timed out -- default 60
    client_body_timeout 10;

    # if client stop responding, free up memory -- default 60
    send_timeout 2;

    # server will close connection after this time -- default 75
    keepalive_timeout 30;

    # number of requests client can make over keep-alive -- for testing environment
    keepalive_requests 10000;

    server_tokens off;
}
```

#### Individual sites

```
server {
    listen 8080;
    list [::]:8080;
    sever_name subdomain.domain.com;

    root /var/www/blog;
}
```

`ln -s ../sites-available/<domain>` from `/etc/nginx/sites-enabled/`

Setup certbot: https://certbot.eff.org/instructions?ws=haproxy&os=ubuntufocal

## Use snapd

```sh
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo snap set certbot trust-plugin-with-root=ok # confirm plugin containment level
sudo snap install certbot-dns-digitalocean
```

## Setup DO credentials

credentials file:

```sh
dns_digitalocean_token = xxxxxxxxxxxxxx
```

chmod 600 to restrict

For a wildcard cert:

```sh
certbot certonly \
    --dns-digitalocean \
    --dns-digitalocean-credentials ~/.secrets/certbot/digitalocean.ini \
    -d whynotcats.com \
    -d '*.whynotcats.com'
```

Certificate is saved at: /etc/letsencrypt/live/<domain>/fullchain.pem
Key is saved at: /etc/letsencrypt/live/<domain>/privkey.pem
Generate pem file for haproxy by cating them

```sh
cat /etc/letsencrypt/live/<domain>/fullchain.pem /etc/letsencrypt/live/<domain>/privkey.pem > /etc/ssl/<domain>.pem
cat /etc/letsencrypt/live/whynotcats.com/fullchain.pem /etc/letsencrypt/live/whynotcats.com/privkey.pem > /etc/ssl/certs/whynotcats.com.pem
```

## Install haproxy

apt install haproxy -y

## Install certificate for haproxy

setup haproxy.cfg

```make
global
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners

defaults
    ...

frontend api
    bind <public_ip or *:80>
    bind *:443 ssl crt /etc/ssl/certs/<domain>.pem
    http-request redirect scheme https unless { ssl_fc }

    acl api hdr_sub(host) -i api.whynotcats.com
    acl phototimes_subdomain hdr_sub(host) -i best-photo-times.whynotcats.com

    use_backend static if phototimes_subdomain
    use_backend api if api
    default_backend static

backend static_sites
    option forwardfor
    http-send-name-header Host
    server server1 <static:server>

backend api
    server server1 <api-server>
```

## Setup dynmaic reload for ssl cert

apt install socat

Create script in /etc/letsencrypt/renewal-hooks/post/update_haproxy_cert.sh

## Update SSL Cert

```sh
#!/bin/bash

CERT_PATH="/etc/ssl/certs/whynotcats.com.pem"

# Create necessary concated cert
cat /etc/letsencrypt/live/whynotcats.com/fullchain.pem /etc/letsencrypt/live/whynotcats.com/privkey.pem > $CERT_PATH

# Start transaction to update the certificate
echo -e "set ssl cert $CERT_PATH <<\n$(cat $CERT_PATH)\n" | socat /run/haproxy/admin.sock -

# Commit the transaction
echo "commit ssl cert $CERT_PATH" | socat /run/haproxy/admin.sock -
```

## Allow for incoming connections

ufw enable
ufw allow ssh
ufw allow http
ufw allow https

# Styling with Bulma

For styling the frontend I wanted something that had some premade components I could use to quickly build, but without a requiring javascript. [Bulma](https://bulma.io/) seems to fit the bill, and it has sass source files which I can throw at trunk to build for me when I want to customize things, or just point at the cdn url for the latest compiled css file.
