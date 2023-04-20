> 本文是 [Tuning the Go HTTP Client Settings for Load Testing](http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/)的中文翻译版，内容有删减


While working on a load testing tool in Go, I ran into a situation where I was seeing tens of thousands of sockets in the `TIME_WAIT` state.

Here are a few ways to get into this situation and how to fix each one.

Repro #1: Create excessive TIME_WAIT connections by forgetting to read the response body
----------------------------------------------------------------------------------------

Run the following code on a linux machine:

```
package main

import (
  "fmt"
  "html"
  "log"
  "net"
  "net/http"
  "time"
)

func startWebserver() {

  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
  })

  go http.ListenAndServe(":8080", nil)

}

func startLoadTest() {
  count := 0
  for {
      resp, err := http.Get("http://localhost:8080/")
      if err != nil {
          panic(fmt.Sprintf("Got error: %v", err))
      }
      resp.Body.Close()
      log.Printf("Finished GET request #%v", count)
      count += 1
  }

}

func main() {

  // start a webserver in a goroutine
  startWebserver()

  startLoadTest()

}
```

and in a separate terminal while the program is running, run:

```
netstat -n | grep -i 8080 | grep -i time_wait | wc -l
```

and you will see this number constantly growing:

```
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
166
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
231
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
293
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
349
...
```

Fix: Read Response Body
-----------------------

Update the `startLoadTest()` method to add the following line of code (and related imports):

```
func startLoadTest() {
  for {
          ...
      if err != nil {
          panic(fmt.Sprintf("Got error: %v", err))
      }
      io.Copy(ioutil.Discard, resp.Body)  // <-- add this line
      resp.Body.Close()
                ...
  }

}
```

Now when you re-run it, calling `netstat -n | grep -i 8080 | grep -i time_wait | wc -l` while it’s running will return 0.

Repro #2: Create excessive TIME_WAIT connections by exceeding connection pool
-----------------------------------------------------------------------------

Another way to end up with excessive connections in the `TIME_WAIT` state is to consistently exceed the connnection pool and cause many short-lived connections to be opened.

Here’s some code which starts up 100 goroutines which are all trying to make requests concurrently, and each request has a 50 ms delay:

```
package main

import (
  "fmt"
  "html"
  "io"
  "io/ioutil"
  "log"
  "net/http"
  "time"
)

func startWebserver() {

  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {

      time.Sleep(time.Millisecond * 50)

      fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
  })

  go http.ListenAndServe(":8080", nil)

}

func startLoadTest() {
  count := 0
  for {
      resp, err := http.Get("http://localhost:8080/")
      if err != nil {
          panic(fmt.Sprintf("Got error: %v", err))
      }
      io.Copy(ioutil.Discard, resp.Body)
      resp.Body.Close()
      log.Printf("Finished GET request #%v", count)
      count += 1
  }

}

func main() {

  // start a webserver in a goroutine
  startWebserver()

  for i := 0; i < 100; i++ {
      go startLoadTest()
  }

  time.Sleep(time.Second * 2400)

}
```

In another shell run `netstat`, note that the number of connections in the `TIME_WAIT` state is growing again, _even though the response is being read_

```
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
166
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
231
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
293
root@14952c2356a7:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
349
...
```

To understand what’s going on, we’ll need to dig in a little deeper into the `TIME_WAIT` state.

What is the socket `TIME_WAIT` state anyway?
--------------------------------------------

So what’s going on here?

What’s happening is that we are creating lots of short lived TCP connections, and the Linux kernel networking stack is keeping tabs on the closed connections to prevent certain problems.

From [The TIME-WAIT state in TCP and Its Effect on Busy Servers](http://www.isi.edu/touch/pubs/infocomm99/infocomm99-web/):

> The purpose of TIME-WAIT is to prevent delayed packets from one connection being accepted by a later connection. Concurrent connections are isolated by other mechanisms, primarily by addresses, ports, and sequence numbers[1].

Why so many TIME_WAIT sockets? What about connection re-use?
------------------------------------------------------------

By default, the Golang HTTP client will do connection pooling. Rather than closing a socket connection after an HTTP request, it will add it to an idle connection pool, and if you try to make another HTTP request before the idle connection timeout (90 seconds by default), then it will re-use that existing connection rather than creating a new one.

This will keep the number of total socket connections low, as long as the pool doesn’t fill up. If the pool is full of established socket connections, then it will just create a new socket connection for the HTTP request and use that.

So how big is the connection pool? A quick look into [transport.go](https://golang.org/src/net/http/transport.go) tells us:

```
var DefaultTransport RoundTripper = &Transport{
        ... 
  MaxIdleConns:          100,
  IdleConnTimeout:       90 * time.Second,
        ... 
}

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2
```

*   The `MaxIdleConns: 100` setting sets the size of the connection pool to 100 connections, but _with one major caveat_: this is on a per-host basis. See the comments on the `DefaultMaxIdleConnsPerHost` below for more details on the implications of this.
*   The `IdleConnTimeout` is set to 90 seconds, meaning that after a connection stays in the pool and is unused for 90 seconds, it will be removed from the pool and closed.
*   The `DefaultMaxIdleConnsPerHost = 2` setting below it. What this means is that even though the entire connection pool is set to 100, there is a _per-host_ cap of only 2 connections!

In the above example, there are 100 goroutines trying to concurrently make requests to the same host, but the connection pool can only hold 2 sockets. So in the first “round” of the goroutines finishing their http request, 2 of the sockets will remain open in the pool, while the remaining 98 connections will be closed and end up in the `TIME_WAIT` state.

Since this is happening in a loop, you will quickly accumulate thousands or tens of thousands of connections in the `TIME_WAIT` state. Eventually, for that particular host at least, you will run out of ephemeral ports and not be able to open new client connections. For a load testing tool, this is _bad news_.

Fix: Tuning the http client to increase connection pool size
------------------------------------------------------------

Here’s how to fix this issue.

```
import (
     .. 
)

var myClient *http.Client

func startWebserver() {
      ... same code as before

}

func startLoadTest() {
        ... 
  for {
      resp, err := myClient.Get("http://localhost:8080/")  // <-- use a custom client with custom *http.Transport
                ... everything else is the same
  }

}


func main() {

  // Customize the Transport to have larger connection pool
  defaultRoundTripper := http.DefaultTransport
  defaultTransportPointer, ok := defaultRoundTripper.(*http.Transport)
  if !ok {
      panic(fmt.Sprintf("defaultRoundTripper not an *http.Transport"))
  }
  defaultTransport := *defaultTransportPointer // dereference it to get a copy of the struct that the pointer points to
  defaultTransport.MaxIdleConns = 100
  defaultTransport.MaxIdleConnsPerHost = 100

  myClient = &http.Client{Transport: &defaultTransport}

  // start a webserver in a goroutine
  startWebserver()

  for i := 0; i < 100; i++ {
      go startLoadTest()
  }

  time.Sleep(time.Second * 2400)

}
```

This bumps the total maximum idle connections (connection pool size) and the per-host connection pool size to 100.

Now when you run this and check the `netstat` output, the number of `TIME_WAIT` connections stays at 0

```
root@bbe9a95545ae:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
0
root@bbe9a95545ae:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
0
root@bbe9a95545ae:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
0
root@bbe9a95545ae:/# netstat -n | grep -i 8080 | grep -i time_wait | wc -l
0
```

The problem is now fixed!

If you have higher concurrency requirements, you may want to bump this number to something higher than 100.