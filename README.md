# Figuring out self-signed certs


## Add jovyan.example.com to your /etc/hosts

Edit `/etc/hosts` and add the following line at the end `127.0.0.1 jovyan.example.com`.
This will configure things so that the domain name jovyan.example.com resolves
to 127.0.0.1.


## Start nginx

We will start a nginx server that serves a static page for `jovian.example.com`
and has a self-signed certificate for this domain.

In the top level directory of this repository run:
```
docker run --rm -v `pwd`/nginx/:/etc/nginx/ -v `pwd`/static:/usr/share/nginx/html -p 49999:443 -p 9999:80 nginx:1.19.4
```

You should be able to open http://jovyan.example.com:9999 in your browser and see the
contents of `static/index.html`. This is served without SSL. If you open
https://jovyan.example.com:49999 in your browser you should receive a warning
that the certificate is self-signed and untrusted. Do not click "make exception" or
"trust anyway".


## Build a "client" docker image

To test how BinderHub reacts to talking to a web site that uses self-signed
certificates we will build a docker image based on BinderHub and add our
certificate to it.

```
docker build -t https-client-testing:v1 -f client/Dockerfile .
```

## Find the IP of the nginx container

We need the IP address assigned to the nginx container. Run
```
docker network inspect bridge
```
and look for an entry in `Containers` that corresponds to your nginx container.
This is easy if you aren't running any other containers as there will only
be one. `docker ps` will list what is running and tell you the name which
should let you find the container if there are many. Make a note of the
`IPv4Address` (something like `127.17.0.2/16`).

## Start the client container

```
docker run --rm -it --entrypoint /bin/bash https-client-testing:v1
```

From inside the client container modify `/etc/hosts` to add a line like
```
127.17.0.2 jovyan.example.com
```
to the end of it. The IP address is the one of the nginx container.

You should now be able to `curl https://jovyan.example.com` from within the
client container. Should print the same HTML as is in `static/index.html`.

Next lets try out Python. Start `python`:
```python
import asyncio
from tornado.httpclient import AsyncHTTPClient

async def get():
  client = AsyncHTTPClient()
  return await client.fetch("https://jovyan.example.com")

asyncio.run(get())
```
This should "just work" as tornado should use the OS level certificate
store.

Now let's configure tornado to use pycurl. This should also work. In the
same `python` interpreter as before:

```python
AsyncHTTPClient.configure("tornado.curl_httpclient.CurlAsyncHTTPClient")
r = asyncio.run(get())
# Should print some HTML that looks like `index.html`
r.body
```

This should also "just work".

The widely used `requests` library will _not_ work because it uses its own
CA bundle. And we only updated the OS CA bundle.
```python
import requests
# this will raise a SSLCertVerificationError
requests.get("https://jovyan.example.com")
```


## Generate your own certificate

This repository contains certificates for `jovyan.example.com` but they have
a short lifetime and you might want to create your own.  You have two options here.  The first is to create a self-signed certificate.  The second is to create a separate certificate authority to sign certs with.

### 1. Create Self-Signed Cert
```
openssl req -newkey rsa:2048 -nodes -keyout nginx/jovyan.example.com.key -addext "subjectAltName = DNS:jovyan.example.com" -x509 -days 30 -out nginx/jovyan.example.com.crt
```
You can answer most of the questions with the default answer. The value that
matters is the domain name you provide as `subjectAltName`.

Once you have generated the certificate you can look at it with:
```
openssl x509 -in nginx/jovyan.example.com.crt -text -noout
```
There should be a section like the following in it:
```
X509v3 extensions:
    X509v3 Subject Key Identifier:
        4E:3F:5C:25:95:B9:0A:06:B8:D6:02:BF:27:ED:E2:F8:83:F0:5A:6C
    X509v3 Authority Key Identifier:
        keyid:4E:3F:5C:25:95:B9:0A:06:B8:D6:02:BF:27:ED:E2:F8:83:F0:5A:6C

    X509v3 Basic Constraints: critical
        CA:TRUE
    X509v3 Subject Alternative Name:
        DNS:jovyan.example.com
```

### Create local CA & test certificate

Instructions pulled from here: https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/

Create the CA key and cert:

```
openssl genrsa -des3 -out testCA.key 2048
openssl req -x509 -new -nodes -key testCA.key -sha256 -days 30 -out testCA.pem
```

Generate the certificate signing request:

```
openssl genrsa -out nginx/jovyan.example.com.key 2048
openssl req -new -key nginx/jovyan.example.com.key -out nginx/jovyan.example.com.csr
```

Create a certificate extensions configuration file (nginx/jovyan.example.com.ext) with the following content:

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = dev.deliciousbrains.com
```

And then use your new CA to sign your .csr:
```
openssl x509 -req -in nginx/jovyan.example.com.csr -CA testCA.pem -CAkey testCA.key -CAcreateserial -out nginx/jovyan.example.com.crt -days 825 -sha256 -extfile nginx/jovyan.example.com.ext
```

If you choose this setup, you'll need to update the file client/Dockerfile to contain the following:

```
FROM jupyterhub/k8s-binderhub:0.2.0-n361.h6f57706

RUN mkdir /usr/local/share/ca-certificates/extra
COPY testCA.pem /usr/local/share/ca-certificates/extra/
RUN update-ca-certificates
```