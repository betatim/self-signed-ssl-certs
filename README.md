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
