---
layout: post
title: Rock with the wired shark! (misc70)
category: internetwache-2016
---

## Description

Sniffing traffic is fun. I saw a wired shark. Isn't that strange?

---

## Challenge Files
[dump.pcapng](/assets/bin/internetwache-2016/shark-dump.pcapng)

---

## The Challenge

We are given one pcapng file containing some HTTP requests.

Exporting the HTTP objects, we obtain a flag.zip file and two HTML files.
The HTML files aren't that important in this challenge.

The flag.zip file is protected by a password.

Examining the request for the flag.zip file in wireshark, we see a basic auth header :

    GET /flag.zip HTTP/1.1
    Host: 192.168.1.41:8080
    Connection: keep-alive
    Authorization: Basic ZmxhZzphenVsY3JlbWE=
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.80 Safari/537.36
    DNT: 1
    Referer: http://192.168.1.41:8080/
    Accept-Encoding: gzip, deflate, sdch
    Accept-Language: en-US,en;q=0.8,ht;q=0.6

Decoding `ZmxhZzphenVsY3JlbWE=` gives us `flag:azulcrema`.

The password for the zip file is `azulcrema`.

Open the zip file to obtain the flag.

**Flag** : IW{HTTP_BASIC_AUTH_IS_EASY}

