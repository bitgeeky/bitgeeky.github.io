---
layout: post
title: Simple HTTPS Server In Python Using Self Signed Certs
date: 2015-06-07 20:54:20 UTC
updated: 2015-06-07 20:54:20 UTC
comments: true
categories: python openssl https server homepage
---

So I came across a situation where I needed to boot up an https server to host some files and guess what its really very easy to do than what I imagined.

Generate self signed certificates using OpenSSL
----
###Generate your server key

```
$ openssl genrsa -des3 -out server.key 1024
```
You will be prompted for a password for your key. Enter, confirm and continue.

###Generate your Certificate Signing Request (CSR)

```
$ openssl req -new -key server.key -out server.csr
```
You will be prompted again for your key password. Enter the one you created from step 1 above. You can then accept the defaults for all of the prompts you are presented with except the Common Name. This is key, and what makes the enhanced certificate validation happy. Since we are doing local development your common name will be `localhost`.

```
Common Name (e.g. server FQDN or YOUR name) []:localhost
```

###Generate your Certificate
Lastly we need to create our certificate. Again, use your key password and you will be all set.

```
$ openssl x509 -req -days 1024 -in server.csr -signkey server.key -out server.crt
```
###Generate a pem file
```
$ cat server.crt server.key > server.pem
```
Python implementation of server:
----

```
#!/usr/bin/python

import BaseHTTPServer, SimpleHTTPServer
import ssl

httpd = BaseHTTPServer.HTTPServer(('localhost', 4443), SimpleHTTPServer.SimpleHTTPRequestHandler)
httpd.socket = ssl.wrap_socket (httpd.socket, certfile='/path/to/server.pem', server_side=True)
httpd.serve_forever()
```
Run `$ python server.py`

Default host here is `localhost` and port is `4443`.

That's it ! You have finally deployed an `https` server.
Go to `https://localhost:4443` and click `Advanced > Proceed to localhost(unsafe)` to accept certificates and see the serverd files.

###Sources:
[Generating valid self signed certificates for localhost development](https://developer.salesforce.com/blogs/developer-relations/2011/05/generating-valid-self-signed-certificates.html)
<br/>
[Creating an HTTPS server in Python](http://www.piware.de/2011/01/creating-an-https-server-in-python/)
