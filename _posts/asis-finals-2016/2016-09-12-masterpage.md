---
layout: post
title: Masterpage
category: ASIS Finals 2016
---

### Description

We pwned ciphers and web services...
Yet we found another administration page which is now locked.. can you access this page?

### The Challenge

In this challenge, we were giving an ordinary login form.

![login](/assets/img/asis-finals-2016/login.png)

Upon submission of the form, we get two different behaviors.
The resulting webpage would contain either `Blocked!` or `Too long!` depending on the size of the username and password.

#### Obtaining the source code
Through a bit of recon, we discovered a `.git` folder at the root of the website :

![git_folder](/assets/img/asis-finals-2016/git_head_folder.png)

At this point, we can use [GitTools](https://github.com/internetwache/GitTools) to extract the source code of the website.
As this website is behind cloudflare, we have to update the source code of the tool a bit since
CloudFlare will block `curl` requests based on the user-agent.

In the `gitdumper.sh` script, we update the curl request with an updated user-agent :

![update_script](/assets/img/asis-finals-2016/update_script.png)

We then run the script :
![download_source](/assets/img/asis-finals-2016/download_source.png)


#### SQL injection

With the git repository at hand, we can understand how the login form works :

Run `git checkout .` to retrieve the `index.php` page.

```php
<?php
...

$id = $mysql->filter($_POST['user'], "auth");
$pw = $mysql->filter($_POST['pass'], "pass");
$ip = $mysql->filter($_SERVER['REMOTE_ADDR'], "hash");

/* resolve ip addr. */
$dns = dns_get_record(gethostbyaddr($ip));
$ip = ($dns) ? print_r($dns, true) : ($ip);

/* mysql filtration */
$filter = "_|information|schema|con|\_|ha|b|x|f|@|\"|`|admin|cas|txt|sleep|benchmark|procedure|\^";
foreach($_POST as $_VAR){
    if(preg_match("/{$filter}/i", $_VAR) || preg_match("/{$filter}/i", $ip))
    {
        exit("Blocked!");
    }
}
if(strlen($id.$pw.$ip) >= 1024 || substr_count("(", $id.$pw.$ip) > 2)
{
    exit("Too Long!");
}

/* admin auth */
$query = "SELECT id, pw FROM admin WHERE id='{$id}' AND pw='{$pw}' AND ip='{$ip}';";
$result = $mysql->query($query, 1);
if($result['id'] === "admin" && $result['pw'] === $_POST['pw'])
{
    echo $flag."<HR>";
}

...
?>
```

Basically, this application takes the supplied `user` and `pass` parameter as well as the hostname of your current IP address,
filters it through `$mysql->filter()` and uses it in an SQL query.

The server will spit out the flag only if the result of the query
returns an `id` of `admin` and a `pw` supplied through the `$_POST['pw']` parameter. Keep in mind that the `$_POST['pw']` is not
the `$_POST['pass']` parameter that we supplied initially.

Here is the source code for `$mysql->filter()` :

```php
<?php
...
function filter($str, $type='sql'){
  switch($type){
      case "url":
          return preg_replace("/[^a-zA-Z0-9-_&\/]/", "", $str);
      case "sql":
          if($this->conn){
              $_filter = preg_replace("/[^a-zA-Z0-9-_!@#$.%^+&*(){}]/", "", $str);
              if($this->mysqli){
                  return mysqli_real_escape_string($this->conn, $_filter);
              }else{
                  return mysql_real_escape_string($_filter, $this->conn);
              }
          }
      case "memo":
          if($this->conn){
              $_filter = htmlspecialchars(preg_replace("/[^a-zA-Z0-9-_:+!@#$.%^&*(){}:\/.\ <>]/", "", $str));
              if($this->mysqli){
                  return mysqli_real_escape_string($this->conn, $_filter);
              }else{
                  return mysql_real_escape_string($_filter, $this->conn);
              }
          }
      case "auth":
          return preg_replace("/[^a-zA-Z0-9-_!@#$.%^&*()\\]/", "", $str);
  }
  return $str;
?>
}
...
```

As we can see, the `pass` and `ip` fields aren't filtered as there are no `pass` and `hash` cases in the `filter()` function.

```
$id = $mysql->filter($_POST['user'], "auth");
$pw = $mysql->filter($_POST['pass'], "pass");
$ip = $mysql->filter($_SERVER['REMOTE_ADDR'], "hash");
```

There is also a second set of lines that filter are input :

```
$filter = "_|information|schema|con|\_|ha|b|x|f|@|\"|`|admin|cas|txt|sleep|benchmark|procedure|\^";
foreach($_POST as $_VAR){
    if(preg_match("/{$filter}/i", $_VAR) || preg_match("/{$filter}/i", $ip))
    {
        exit("Blocked!");
    }
}
```

This filter checks all parameters passed through the `POST` request and blocks the request based on the `$filter` regex.

Nevertheless, we can still bypass this by using SQL string manipulation functions and obtain the flag.
Here is the final payload :

![payload](/assets/img/asis-finals-2016/payload.png)


With this payload, we inject a `UNION` query that will return a row containing an
 `id` of `admin` and a `pw` of `0`, resulting in the flag being printed on the page :

![flag](/assets/img/asis-finals-2016/flag.png)

