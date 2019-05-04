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

On the /admin page, we have a form from which we can send a URL. This URL will
be visited by an "admin". From this feature alone, we can determine that we're dealing with
an XSS challenge.

There is also a note in the bottom of the page suggesting that we should try
to access this page from `127.0.0.1:8080` rather than from the external host
(`bypasses-everywhere.ctf.insecurity-insa.fr`) :

> You're currently seeing this page from an unknown IP address.
> I'm usually connecting to this page by clicking a link from another hidden page on http://127.0.0.1:8080, so I'm pretty sure this page is safe :)

This gives us an idea of what we have to do : Find an XSS on the website, send it to the admin,
and leak the contents of `http://127.0.0.1:8080`.

# Finding an XSS Vulnerability

There are a total of two XSS vulnerabilities in this webapp, one on `/article` and another on
`/admin`. For this solution, we won't be looking at the `/admin` page at all.

Going through the available parameters on the `/article` page, we find that two of them are
vulnerable to XSS : `time` and `unit`.


