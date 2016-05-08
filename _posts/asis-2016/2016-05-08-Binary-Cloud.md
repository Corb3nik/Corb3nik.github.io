---
layout: post
title: Binary Cloud
category: ASIS 2016
---

### Description
You can upload anything you want! (But please, do not inject php files.)
Due to the security issues, you can only upload one file per request.
Note that the server removes all uploaded/cached files every minute.

Regards,
ASIS Team.

---

#### The Challenge

This was a very fun challenge for me, as it was based off my own [research](http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/).

**1. Information Gathering**

I started things off by looking for a `robots.txt` file.

Going to `https://binarycloud.asis-ctf.ir/robots.txt` shows us three entries :

![robots](/static/img/asis-2016/robots.png)

Visiting the `/cache` and `/uploads` pages show us a `forbidden` message.
On the other hand, following the link `/debug.php` gives us a `phpinfo()` page!

![phpinfo](/static/img/asis-2016/phpinfo.png)

From this page, we know that they're running PHP 7.0.4.

As the description and `robots.txt` mentionned cache files, I checked if OPcache is enabled. Turns out it is, and its location corresponds to the `/cache` folder found in the robots.txt file :

    opcache.file_cache=/home/binarycloud/www/cache


<br/>
**2. Obtaining the Source Code**

Looking at the URLs, we can notice right away a potential LFI vulnerability, as the upload module URL looks like the following :

    https://binarycloud.asis-ctf.ir/?page=upload

We can use [PHP wrappers](http://php.net/manual/en/wrappers.php.php) to obtain the source code :

    http://binarycloud.asis-ctf.ir?page=php://filter/convert.base64-encode/resource=upload

Viewing the source of the URL above will show us the `base64` encoded source code :

![base64](/static/img/asis-2016/base64.png)

This is a section of the extracted source code for `upload.php` :

  function ew($haystack, $needle) {
      return $needle === "" || (($temp = strlen($haystack) - strlen($needle)) >= 0 && strpos($haystack, $needle, $temp) !== false);
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
    die("<script>alert('$msg');history.go(-1);</script>");
  }

  filter_directory();

  if($_SERVER['QUERY_STRING'] && $_FILES['file']['name']){
    if(!file_exists($_SERVER['QUERY_STRING'])) error("error3");
    $name = preg_replace("/[^a-zA-Z0-9\.]/", "", basename($_FILES['file']['name']));
          if(ew($name, ".php")) error("error");
    $filename = $_SERVER['QUERY_STRING'] . "/" . $name;
    if(file_exists($filename)) error("exists");
    if (move_uploaded_file($_FILES['file']['tmp_name'], $filename)){
      die("uploaded at <a href=$filename>$filename</a><hr><a href='javascript:history.go(-1);'>Back</a>");
    }else{
      error("error");
    }
  }

From this code, we know that :

  - We can't upload `.php` files.
  - Our request URI must not contain the strings `cache` or `binarycloud`.
  - Our filename is filtered through `preg_match()` and `basename()`.

Since we can't upload `.php` files, the goal will most likely be to inject `.php.bin` files in the `/cache` folder so we can run our own code.

**Bypass filter_directory()**

The upload module takes the query string of our POST request and uses it as the destination folder for our file.

Sending a POST request at `/upload.php?FOLDER` will try to upload our file in the `FOLDER/` folder.

Since we want upload in the `cache/` folder, we need to send our POST request like so : `/upload.php?cache`.
The problem here is that `filter_directory` will block us from using the `cache` keyword.

    function filter_directory(){
      $data = parse_url($_SERVER['REQUEST_URI']);
      $filter = ["cache", "binarycloud"];
      foreach($filter as $f){
        if(preg_match("/".$f."/i", $data['query'])){
          die("Attack Detected");
        }
      }
    }


