---
title: "How to reverse proxy the WebSocket protocol"
description: "The article will introduce how to reverse proxy the WebSocket protocol."
publishDate: "4 Dec 2023"
tags: ["webdev", "go", "programming", "tutorial"]
---

## Introduction

The article will introduce how to reverse proxy the WebSocket protocol.

We'll start with the basics, understanding reverse proxying and the WebSocket protocol. Additionally, we'll delve into learning how to implement and explore relevant details by reading the source code of Hertz's open-source websocket reverseproxy extension.

- **[Hertz](https://github.com/cloudwego/hertz)**: ByteDance's open-source high-performance Golang HTTP framework;
- **[reverseproxy](https://github.com/hertz-contrib/reverseproxy)**: A WebSocket reverse proxy extension for the Hertz framework, inspired by [fasthttp-reverse-proxy](https://github.com/yeqown/fasthttp-reverse-proxy) for WebSocket reverse proxying.

## Basic knowledge

### What is a reverse proxy?

A reverse proxy is a type of proxy server positioned between internal servers and an external network, used to handle requests to internal servers. When a client sends a request, it doesn't directly access the target server but does so through the reverse proxy server. This proxy server is responsible for forwarding the request to one or more target servers and returning the obtained response to the client.

The typical functions of a reverse proxy are as follows:

- **Hide Backend Servers**: Clients communicate only with the reverse proxy, not directly with the backend servers, thereby concealing information about the backend servers.
- **Load Balancing**: The reverse proxy can distribute requests among multiple backend servers based on load, balancing the server load.
- **Cache Static Resources**: It can cache static content to reduce server load and enhance response speed.
- **Security**: It can act as a firewall, filtering malicious requests to improve security.

A schematic representation of a reverse proxy is as follows: NGINX is a commonly used reverse proxy server (often referred to as a gateway), where user requests are directed to the reverse proxy server, which then forwards them to a cluster of backend servers.

![reverseproxy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gcp6rbg4b26no3fa4a7r.png)

### What is WebSocket?

WebSocket is a protocol that enables **full-duplex** communication over a single TCP connection. Unlike the traditional HTTP request-response model, WebSocket allows the establishment of persistent connections between servers and clients, enabling bidirectional communication for real-time data transmission. It permits servers to actively push data to clients without requiring client-initiated requests.

- **Full-Duplex Communication**: Allows bidirectional communication between servers and clients, enabling real-time sending and receiving of data.
- **Low Latency**: Established over a single TCP connection, reducing connection setup overhead and lowering communication latency.
- **Reduced Data Transfer Overhead**: Compared to traditional HTTP polling, WebSocket minimizes data transfer overhead, making data transmission more efficient.

## Implementation Approach

After grasping the fundamental knowledge, we can now start considering the specific implementation approach.

### Terminology Agreement

Let's first establish a few terms that will be used to facilitate our understanding of the entire process:

- **Client**: User;
- **Proxy Server**: Reverse proxy server, for instance, NGINX mentioned earlier and the Hertz server used here;
- **Backend Server**: The actual server that receives and handles client requests, not directly communicating with the client;

### Basic Concept

As we're implementing a reverse proxy service based on the Hertz HTTP framework, the reverse proxy server here is Hertz itself. To enable websocket reverse proxying, we can achieve this by providing an `http.Handler`. This handler can forward the request for establishing a websocket connection from the user to the backend server. Subsequently, a websocket connection is established between the proxy server and the backend server, and another between the client and the proxy server. The proxy server manages message transmission between these two connections.

The process of establishing a websocket reverse proxy from connection setup to completion is outlined below:

- **1. The client initiates a handshake request to establish a websocket connection with the proxy server.**
- **2. The proxy server forwards the request to the backend server.**
- **3. The proxy server establishes a websocket connection with the backend server.**
- **4. The client establishes a websocket connection with the proxy server.**

Once all connections are established, the websocket reverse proxy is set up. Below is an example of a client sending information through the websocket reverse proxy:

- **5. The client writes a message to the websocket connection established with the proxy server.**

- **6. The proxy server reads the message from the websocket connection with the client.**

- **7. The proxy server writes the read message to the websocket connection established with the backend server.**

I've also provided a sequence diagram where the numbering corresponds to the steps outlined above. Readers can refer to the diagram to better understand the entire process of reverse proxying websocket connections.

![process](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hshn4460yv1drgnc9gze.png)

### Specific Implementation

With the basic concept of implementation in place, we can now delve into the specific implementation details. Here, we will comprehend the concrete implementation process by reviewing the code in the [Hertz framework websocket reverse proxy extension](https://github.com/hertz-contrib/reverseproxy).

The code for websocket reverse proxy in this extension resides in two main files:

- **[ws_reverse_proxy.go](https://github.com/hertz-contrib/reverseproxy/blob/master/ws_reverse_proxy.go)**: Contains the primary business logic for websocket reverse proxying.
- **[ws_reverse_proxy_option.go](https://github.com/hertz-contrib/reverseproxy/blob/master/ws_reverse_proxy_option.go)**: Provides customizable configuration options for users.

Readers can either clone this repository and review the code alongside or directly examine the source code analysis provided below.

#### Main Struct (WSReverseProxy)

```go
type WSReverseProxy struct {
	target  string
	options *Options
}
```

The main structure of the websocket reverse proxy appears to be quite simple, consisting of only two fields:

- target: The target address for reverse proxying, which is the path to the backend server.
- options: Configuration options. WSReverseProxy offers three customizable configuration options to users in the form of Functional Options.

#### Constructor (NewWSReverseProxy)

```go
func NewWSReverseProxy(target string, opts ...Option) *WSReverseProxy {
	if target == "" {
		panic("target string must not be empty")
	}
	options := newOptions(opts...)
	wsrp := &WSReverseProxy{
		target:  target,
		options: options,
	}
	return wsrp
}
```

The logic of the constructor method is as follows:

1. It begins by checking whether the backend server path provided in the method parameter is empty. If it's empty, it directly triggers a panic.
2. Initialization of configuration options occurs next. If the user has configured any options, it uses those values; otherwise, it resorts to default values.
3. Creation of the main structure takes place, followed by assigning the values.
4. Finally, the populated main structure is returned to the user.

#### Core Method ServeHTTP

```go
func (w *WSReverseProxy) ServeHTTP(ctx context.Context, c *app.RequestContext)
```

The `ServeHTTP` method is the core function for implementing websocket reverse proxying. As mentioned earlier, it serves as the **Handler**. Users utilize an instance of the main structure returned by the constructor method to invoke `ServeHTTP` and register the corresponding routes to facilitate the process of websocket reverse proxying.

Given that `ServeHTTP` is quite extensive, I'll break down the method into four parts according to the process of implementing websocket reverse proxying. We'll analyze these four parts sequentially.

##### Part 1: Prepare Forwarding Header

```go
forwardHeader := prepareForwardHeader(ctx, c)
if w.options.Director != nil {
    w.options.Director(ctx, c, forwardHeader)
}
```

In this section, the Handler processes the request header of the client's handshake request using the `prepareForwardHeader` method and returns a `forwardHeader` to enable the proxy server to initiate a request to establish a websocket connection with the backend server.

Additionally, if the user has customized the `Director` option, further processing of the `forwardHeader` can be performed accordingly.

```go
type Director func(ctx context.Context, c *app.RequestContext, forwardHeader http.Header)
```

Next, let's take a look at the `prepareForwardHeader` method:

```go
func prepareForwardHeader(_ context.Context, c *app.RequestContext) http.Header {
	forwardHeader := make(http.Header, 4)
	if origin := string(c.Request.Header.Peek("Origin")); origin != "" {
		forwardHeader.Add("Origin", origin)
	}
	if proto := string(c.Request.Header.Peek("Sec-Websocket-Protocol")); proto != "" {
		forwardHeader.Add("Sec-WebSocket-Protocol", proto)
	}
	if cookie := string(c.Request.Header.Peek("Cookie")); cookie != "" {
		forwardHeader.Add("Cookie", cookie)
	}
	if host := string(c.Request.Host()); host != "" {
		forwardHeader.Set("Host", host)
	}
	clientIP := c.ClientIP()
	if prior := c.Request.Header.Peek("X-Forwarded-For"); prior != nil {
		clientIP = string(prior) + ", " + clientIP
	}
	forwardHeader.Set("X-Forwarded-For", clientIP)
	forwardHeader.Set("X-Forwarded-Proto", "http")
	if string(c.Request.URI().Scheme()) == "https" {
		forwardHeader.Set("X-Forwarded-Proto", "https")
	}
	return forwardHeader
}
```

The method logic is as follows:

- **1. First, initialize a `http.Header` with a size of 4 (`type Header map[string][]string`);**

- **2. Check the client's request header for fields such as `Origin`, `Sec-Websocket-Protocol`, `Cookie`, and `Host` (HTTP Header). If any are present, set them in the `forwardHeader`;**

  The `Sec-Websocket-Protocol` header is used to specify the subprotocol that the client and server will use when establishing a WebSocket connection. The WebSocket protocol allows defining one or more subprotocols during connection establishment, which can describe the types of data or message formats to be sent and received on that connection.

- **3. Check if the client's request header contains the `X-Forwarded-For` field. If present, append the current client's IP to the existing `X-Forwarded-For`. If not present, set the `X-Forwarded-For` field in `forwardHeader` to the current client's IP;**

  The `X-Forwarded-For` header is typically added by a proxy server when forwarding requests and is used to identify the original client's IP address.

  In network communication, when a request passes through multiple proxy servers (like load balancers, reverse proxies, etc.), the actual IP address of the original client that initiated the request might be hidden. To trace the true origin of the request, proxy servers usually add the `X-Forwarded-For` field in the HTTP request header, which contains the original client's IP address.

  An example might look like this:

   ```
   Note: client1, proxy1, proxy2 here are actual IP addresses
   X-Forwarded-For: client1, proxy1, proxy2
   ```

- **4. Set the `X-Forwarded-Proto` field in `forwardHeader` and return it;**

  The `X-Forwarded-Proto` header is typically added by a proxy server when forwarding requests and indicates the protocol (HTTP or HTTPS) used in the original request.

**Points to Note:**

- **Difference between `header.Add` and `header.Set` methods;**

The `Add` method checks if the key already exists. If it doesn't, it sets the corresponding key and value. If it does exist, it appends the current value to the existing value array. On the other hand, the `Set` method doesn't perform any checks. It sets the key and value regardless of whether it already exists or not.

Combining the explanation with the following code might make it easier for you to understand:

```go
header := make(http.Header)
header.Add("Key1", "Value1")
fmt.Println(header)

header.Add("Key1", "Value2")
fmt.Println(header)

header.Set("Key1", "Value3")
fmt.Println(header)

// output:
// map[Key1:[Value1]]
// map[Key1:[Value1 Value2]]
// map[Key1:[Value3]] 
```

- **Why aren't fields like `Connection`, `Upgrade`, `Sec-WebSocket-Key`, etc., set for `forwardHeader`?**

Readers familiar with the WebSocket protocol might know that these fields are all essential for a WebSocket handshake request. They indicate that the client initiating the handshake intends to upgrade to the WebSocket protocol. If you add the following code within the `prepareForwardHeader` method, you'll find that indeed these fields are present in the client's request.

```go
fmt.Println("Upgrade: " + string(c.Request.Header.Peek("Upgrade")))
fmt.Println("Connection: " + string(c.Request.Header.Peek("Connection")))
fmt.Println("Sec-WebSocket-Key: " + string(c.Request.Header.Peek("Sec-Websocket-Key")))
```

However, the reason these fields aren't included in `forwardHeader` is because, as mentioned earlier, the client doesn't establish the WebSocket connection directly with the backend server but with the proxy server.

Now, some readers might wonder how the WebSocket connection is established with the backend server if `forwardHeader` doesn't contain these necessary fields. Please refer to the upcoming second part for further clarification.

##### Part 2: Proxy Server Establishes WebSocket Connection with Backend Server

```go
connBackend, respBackend, err := w.options.Dialer.Dial(w.target, forwardHeader)
if err != nil {
    hlog.CtxErrorf(ctx, "can not dial to remote backend(%v): %v", w.target, err)
    if respBackend != nil {
        if err = wsCopyResponse(&c.Response, respBackend); err != nil {
            hlog.CtxErrorf(ctx, "can not copy response: %v", err)
        }
    } else {
        c.AbortWithMsg(err.Error(), consts.StatusServiceUnavailable)
    }
    return
}
```

In the first part, we've prepared the `forwardHeader`. Now, the next step involves passing the backend server's path (target) along with the `forwardHeader` to the `Dial` method. The `Dial` method initiates a handshake request to the backend server, allowing the proxy server to establish a WebSocket connection with the backend server.

Within the `Dial` method, the necessary fields mentioned in the first part are also added to the initiated request:

```go
req.Header["Upgrade"] = []string{"websocket"}
req.Header["Connection"] = []string{"Upgrade"}
req.Header["Sec-WebSocket-Key"] = []string{challengeKey}
req.Header["Sec-WebSocket-Version"] = []string{"13"}
```

**The `connBackend` returned by the `Dial` method is the instance of the WebSocket connection established between the proxy server and the backend server.**

Of course, if the `Dial` method encounters an error, indicating a failure to establish a WebSocket connection between the proxy server and the backend server, the corresponding response needs to be returned to the client. However, as this isn't the primary logic, I won't delve into further details here.

##### Part 3: Establishing WebSocket Connection between Client and Proxy Server

```go
if err := w.options.Upgrader.Upgrade(c, func(connClient *hzws.Conn) {
    defer connClient.Close()
    ...
}); err != nil {
    hlog.CtxErrorf(ctx, "can not upgrade to websocket: %v", err)
}
```

In the second part, we saw that the proxy server had successfully established a WebSocket connection with the backend server using the `Dial` method. In this third part, we'll use the `Upgrade` method to enable the client to establish a WebSocket connection with the proxy server.

**`Upgrade` will finalize the established WebSocket connection with the client's handshake and pass it into `connClient`, signifying that `connClient` is the instance of the WebSocket connection established between the client and the proxy server.**

Of course, if the `Upgrade` method encounters an error, indicating a failed connection establishment, it will log the error, and the entire `ServeHTTP` method will consequently terminate.

##### Part 4: Connection Communication

```go
var (
    errClientC  = make(chan error, 1)
    errBackendC = make(chan error, 1)
    errMsg      string
)

hlog.CtxDebugf(ctx, "upgrade handler working...")

gopool.CtxGo(ctx, func() {
    replicateWSRespConn(ctx, connClient, connBackend, errClientC)
})
gopool.CtxGo(ctx, func() {
    replicateWSReqConn(ctx, connBackend, connClient, errBackendC)
})

for {
    select {
    case err = <-errClientC:
        errMsg = "copy websocket response err: %v"
    case err = <-errBackendC:
        errMsg = "copy websocket request err: %v"
    }

    var ce *websocket.CloseError
    var hzce *hzws.CloseError
    if !errors.As(err, &ce) || !errors.As(err, &hzce) {
        hlog.CtxErrorf(ctx, errMsg, err)
    }
}
```

In the third part, the WebSocket connection between the client and the proxy server was established, and we obtained the connection instance `connClient`. Next is to enable the client to communicate with the backend server in a reverse proxy manner using the following two connections:

- `connClient`: WebSocket connection instance between the client and the proxy server
- `connBackend`: WebSocket connection instance between the proxy server and the backend server

As seen, we first prepare two channels to receive errors during communication:

- `errClientC`: For receiving errors when the backend server sends messages to the client
- `errBackendC`: For receiving errors when the client sends messages to the backend server

Then, we retrieve two goroutines from the coroutine pool to concurrently execute the `replicateWSRespConn` and `replicateWSReqConn` methods. The naming of these methods might seem peculiar because WebSocket is a full-duplex communication protocol, where requests and responses are relative.

However, here, we uniformly consider messages sent from the backend server to the client as responses and messages sent from the client to the backend server as requests, in terms of the client's perspective.

Since this can be a bit convoluted, I've created a schematic diagram to aid in understanding.

![schematicDiagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f64hjgcyd99y61hq34tu.png)

Let's continue examining these two concurrently executed communication methods. Since the logic of both methods is essentially similar, let's take a look at the logic of the `replicateWSRespConn` method:

```go
func replicateWSRespConn(ctx context.Context, dst *hzws.Conn, src *websocket.Conn, errC chan error) {
	for {
		msgType, msg, err := src.ReadMessage()
		if err != nil {
			...
			errC <- err

			if err = dst.WriteMessage(hzws.CloseMessage, msg); err != nil {
				hlog.CtxErrorf(ctx, "write message failed when replicate websocket conn: err=%v", err)
			}
			break
		}

		err = dst.WriteMessage(msgType, msg)
		if err != nil {
			hlog.CtxErrorf(ctx, "write message failed when replicating websocket conn: msgType=%v msg=%v err=%v", msgType, msg, err)
			errC <- err
			break
		}
	}
}
```

The logic of the method is quite simple. It involves reading messages sent from the backend server to the proxy server through the `ReadMessage` method of `connBackend` and then writing the read messages into the connection between the client and the proxy server using the `WriteMessage` method of `connClient`.

Any errors encountered are passed to `errClientC` for unified handling outside the method.

Returning to the outer method, errors are received from `errClientC` and `errBackendC` using a `for-select` structure. If the error is not a `CloseError`, it is logged.

**With this, we have completed the understanding of the core logic within the `ServeHTTP` method. The logic of the `ServeHTTP` method aligns entirely with the websocket reverse proxy process outlined in the basic approach. Readers can compare the basic approach with the specific implementation to achieve a better understanding.**

## Example Usage (Echo Server)

Just now, we've finished reading through the core code of the Hertz framework's WebSocket reverse proxy extension. Here, we'll demonstrate the usage of this extension by creating an example of an echo server. This example aims to assist readers in better understanding how to utilize this extension.

We'll break down this example into three parts, and the complete code will be provided at the end.

Firstly, let's globally declare the addresses and paths for the proxy server and the backend server:

```go
var (
	proxyURL    = "ws://127.0.0.1:8080/ws"
	backendURL  = "ws://127.0.0.1:9090/backend"
	proxyAddr   = "127.0.0.1:8080"
	backendAddr = "127.0.0.1:9090"
)
```

### Proxy Server

```go
// proxy server
wsrp := reverseproxy.NewWSReverseProxy(backendURL)
ps := server.Default(server.WithHostPorts(proxyAddr))
ps.GET("/ws", wsrp.ServeHTTP)
go ps.Spin()
```

Using the constructor method `NewWSReverseProxy` that we learned in the specific implementation section, create a WebSocket reverse proxy instance with the `backendURL` as the target parameter. Next, register the `ServeHTTP` method under the `/ws` route, and start the proxy server via a goroutine. It's important to note that the address of the proxy server and the registered route should correspond to the `proxyURL`.

### Backend Server

```go
go func() {
    // backend server
    bs := server.Default(server.WithHostPorts(backendAddr))
    bs.GET("/backend", func(ctx context.Context, c *app.RequestContext) {
        upgrader := &websocket.HertzUpgrader{}
        if err := upgrader.Upgrade(c, func(conn *websocket.Conn) {
            for {
                msgType, msg, err := conn.ReadMessage()
                if err != nil {
                    hlog.Errorf("backend read message err: %v", err)
                }
                err = conn.WriteMessage(msgType, msg)
                if err != nil {
                    hlog.Errorf("backend write message err: %v", err)
                }
            }
        }); err != nil {
            hlog.Errorf("upgrade error: %v", err)
            return
        }
    })
    bs.Spin()
}()
```

Start the backend server via a goroutine and register the `/backend` route. Similarly, ensure that the address of the backend server and the registered route correspond to the `backendURL`. Then, handle incoming WebSocket connections within the handler.

Begin by using the `Upgrade` method to upgrade the HTTP protocol to the WebSocket protocol. Subsequently, read messages sent by the client (in this context, the client refers to the proxy server) from the established WebSocket connection and write the messages back to the WebSocket connection. This completes the echo logic, which continuously repeats this process within a `for` loop.

### Client

```go
// client
conn, _, err := reverseproxy.DefaultOptions.Dialer.Dial(proxyURL, make(http.Header))
if err != nil {
    hlog.Errorf("client dial err: %v", err)
    return
}

time.Sleep(time.Second)

var echoInput string
for {
    fmt.Print("send: ")
    _, _ = fmt.Scanln(&echoInput)
    err = conn.WriteMessage(websocket.TextMessage, []byte(echoInput))
    if err != nil {
        hlog.Errorf("client write message err: %v", err)
    }
    _, echoOutput, err := conn.ReadMessage()
    if err != nil {
        hlog.Errorf("client read message err: %v", err)
    }
    fmt.Println("receive: " + string(echoOutput))
}
```

Using the `Dial` method, send a handshake request to the proxy server to upgrade to the WebSocket protocol. Once you obtain the returned connection instance, utilize the `Scanln` method to read user-input messages and write them to the connection. Simultaneously, receive the returned messages from the connection and print them. Continue this process within a `for` loop.

With this, we've completed an example of a WebSocket reverse proxy echo server.

### Complete Code

The complete code for the echo server example is as follows:

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/hlog"
	"github.com/hertz-contrib/reverseproxy"
	"github.com/hertz-contrib/websocket"
)

var (
	proxyURL    = "ws://127.0.0.1:8080/ws"
	backendURL  = "ws://127.0.0.1:9090/backend"
	proxyAddr   = "127.0.0.1:8080"
	backendAddr = "127.0.0.1:9090"
)

func main() {
	// proxy server
	wsrp := reverseproxy.NewWSReverseProxy(backendURL)
	ps := server.Default(server.WithHostPorts(proxyAddr))
	ps.GET("/ws", wsrp.ServeHTTP)
	go ps.Spin()

	time.Sleep(time.Second)

	go func() {
		// backend server
		bs := server.Default(server.WithHostPorts(backendAddr))
		bs.GET("/backend", func(ctx context.Context, c *app.RequestContext) {
			upgrader := &websocket.HertzUpgrader{}
			if err := upgrader.Upgrade(c, func(conn *websocket.Conn) {
				for {
					msgType, msg, err := conn.ReadMessage()
					if err != nil {
						hlog.Errorf("backend read message err: %v", err)
					}
					err = conn.WriteMessage(msgType, msg)
					if err != nil {
						hlog.Errorf("backend write message err: %v", err)
					}
				}
			}); err != nil {
				hlog.Errorf("upgrade error: %v", err)
				return
			}
		})
		bs.Spin()
	}()

	time.Sleep(time.Second)

	// client
	conn, _, err := reverseproxy.DefaultOptions.Dialer.Dial(proxyURL, make(http.Header))
	if err != nil {
		hlog.Errorf("client dial err: %v", err)
		return
	}

	time.Sleep(time.Second)
    
	var echoInput string
	for {
		fmt.Print("send: ")
		_, _ = fmt.Scanln(&echoInput)
		err = conn.WriteMessage(websocket.TextMessage, []byte(echoInput))
		if err != nil {
			hlog.Errorf("client write message err: %v", err)
		}
		_, echoOutput, err := conn.ReadMessage()
		if err != nil {
			hlog.Errorf("client read message err: %v", err)
		}
		fmt.Println("receive: " + string(echoOutput))
	}
}
```

## Summary

That concludes all the content of this article. We started from the basics of WebSocket reverse proxy, determined the basic approach to implementation, and delved deeper into understanding how to reverse proxy WebSocket by reading the code of the WebSocket reverse proxy extension in the Hertz framework and using an echo server as an example.

Hopefully, this article has been helpful for readers in understanding WebSocket reverse proxy. If there are any mistakes or questions, please feel free to comment or message me :).

## Reference List

- https://github.com/hertz-contrib/reverseproxy
- https://github.com/cloudwego/hertz-examples
- https://github.com/yeqown/fasthttp-reverse-proxy
- https://github.com/cloudwego/hertz