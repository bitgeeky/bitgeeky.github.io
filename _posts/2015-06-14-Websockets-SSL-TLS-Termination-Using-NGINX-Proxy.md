---
layout: post
title: Websockets SSL/TLS Termination Using NGINX Proxy  
date: 2015-06-14 09:17:00 UTC
updated: 2015-06-14 09:17:00 UTC
comments: true
categories: websockets proxy nginx https server ws wss ssl tls homepage
---

There can be a situation where your application is configured to communicate via websockets protocol `ws` and you want to expose it over the internet while still keeping the client side secure by SSL/TLS or say the client will always get an `HTTPS` page.

In such a situation the `HTTPS` page won't allow insecure content to be present on the page so you need a `wss` protocol, but in my case I had no control over the application server. So a solution to such a problem is to use a SSL/TLS terminator in between the application server and the client.

There are a few options which act as a websockets proxy, I decided to use NGINX because of the variety of options it provides such as listening mutiple ports, allowing multiple upstreams, support for unix sockets and a bunch of other features without comprimising the performace parameters.

The final architecture after using NGINX as a websockets proxy is:
<br/>
<br/>
<img src="/public/images/nginx.png" alt="NGINX Proxy" style="margin-right:15px; width: 100%; height: 100%" />
<br/>
We use secure websockets communication on client side using the `wss` protocol and inbetween client and application server is a NGINX proxy server which allows to terminate the SSL/TLS connection and establishes an insecure connection using `ws` protocol.

All that is needed to convert an instance of NGINX to a proxy server is a few changes in configuration file.

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream appserver {
    server 192.168.100.10:9222; # appserver_ip:ws_port
}

server {
    listen 8888; // client_wss_port
    
    ssl on;
    ssl_certificate /path/to/crt;
    ssl_certificate_key /path/to/key;


    location / {
        proxy_pass https://appserver;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

The above configuration should be within `http{ }` block.

Replace the `appserver_ip` and `ws_port` with the ip and port of your application server, `client_wss_port` with the port on which client makes a `wss` connection and provide paths to certificate/key.

Just reload the NGINX configuration or restart the server to get everything up and running.
