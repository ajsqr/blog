---
date: '2025-01-11T16:47:03+05:30'
draft: false
title: 'Implemeting Websocket Handshake'
tags: [websocket, RFC, RFC 6455, Golang]
---


## Introduction

  

Welcome to the second post of the **reinventing the wheel - websocket** series. In this article, I intent to cover websocket handshake in detail and implement it in golang. Before continuing, it is recommended to go through the introductory post [here]({{<ref  "blog/introduction-to-websocket-protocol.md"  >}})

## Client Handshake

In the previous article, we saw that the client handshake request looked like this.

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```
It is a standard HTTP upgrade request with a few additional headers.

The request URI defines the endpoint that the client wishes to connect to. `Host` header is a standard HTTP/1.1 header, and it is an agreement between the client and the server about which host to use. Almost all of them are standard HTTP headers with a few exceptions.

The header `Sec-WebSocket-Protocol` sent by the client lists the extensions that the client supports. A server can accept any or none of the extensions listed there.

The `Origin` header is for intended to be used to protect again cross origin attacks. Here, based on the origin header, the server decides if it should accept the connection from this particular origin.

The `Sec-WebSocket-Key` is a special header, and the server uses this header to create the accept token. If this sounds confusing, refer the previous post, where I have written about how the accept token is generated. We will however write the necessary code to generate this token in this article.

The websocket client handshake request is a standard HTTP/1.1 upgrade request. This is a smart idea because a single server port can be used to listen for both websocket requests and HTTP requests. Any other intermediary can also understand this request. If the server has the capability to serve a websocket connection, it can decide to do so or ask the client to fall back to use HTTP protocol.

## Server Handshake 
 
 This section will demonstrate on how to perform a server side handshake for a successful websocket connection. I'm going to use golang to implement it.

To start with, we need to create a TCP listener to listen for incoming requests. 

```go
package main

import (
    "net"
)

const (
	// websocketGUID is the GUID specified in RFC 6455
	websocketGUID =  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
)

func main() {
	listener, err := net.Listen("tcp", ":8123")
	if err != nil{
		panic(err.Error())
	}

    for {
        conn, err := listener.Accept()
        if err != nil{
		    panic(err.Error())
	    }

        fmt.Println("connection", conn)
    }
}
```

The Conn object returned by the Accept method is really cool. It implements both [io.Reader](https://pkg.go.dev/io#Reader) and [io.Writer](https://pkg.go.dev/io#Writer) interfaces, so we can easily read and write through the connection. Since the incoming request is HTTP, we can use the reader and parse the request manually. Go has a powerful http library and we can use the inbuilt function to do this.

Now that we are ready to accept a connection, lets perform the important task. Generating the accept token.

The following snippet will generate the accept token based on the algorithm I covered in the previous part.

```go
// skipping imports

const (
	// websocketGUID is the GUID specified in RFC 6455
	websocketGUID =  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
)

func generateWebsocketAcceptToken(secWebsocketKey string) string {
    // concatenate the token from the request with the standard GUID
	combinedKey := []byte(fmt.Sprintf("%s%s", secWebsocketKey, websocketGUID))

    // generate the SHA1 hash of the combined key
	hash := sha1.Sum(combinedKey)

    // base64 encode the resulting hash
	encodedKey := base64.StdEncoding.EncodeToString(hash[:])
	return encodedKey
}
```

Finally, we just have to generate a valid response with all this metadata and send it to the client.

```go
func newAcceptResponse(r *http.Request) *http.Response {
	websocketKey := r.Header.Get("Sec-WebSocket-Key")
	acceptToken := generateWebsocketAcceptToken(websocketKey)
	resp := http.Response{
        // HTTP protocol versio has to be >= HTTP/1.1
		Proto : "HTTP/1.1",
		ProtoMajor: 1, 
		ProtoMinor: 1,
		StatusCode: 101,
		Header: http.Header{},
	}

	resp.Header.Set("Upgrade", "websocket")
	resp.Header.Set("Connection", "Upgrade")

    // set the accept header
	resp.Header.Set("Sec-WebSocket-Accept", acceptToken)

	return &resp
}
```


And we are done! The complete code for this example can be found [here](https://github.com/ajsqr/websocket-implementation-examples/blob/master/tcp-example/main.go). You can build and run the example by running the following commands. 

```sh
go build tcp-example/main.go
./main
```

I'm using [postman](https://www.postman.com/) client as the websocket client. After running the example above, we can easily validate that the websocket connection was successful. 

## HTTP with Websockets

Previously, we have talked about how HTTP and websocket handlers could listen to the same port, and that was one of the really cool things about the websocket handshake. However, in the above example I have a dedicated listener which only handles websocket requests. Lets change that and re-write the code to handle both websocket and HTTP requests.

The `net/http` standard library has this really useful feature called [hijacker](https://pkg.go.dev/net/http#Hijacker). I think this is very cool, but some people are really [bummed](https://groups.google.com/g/golang-nuts/c/sN6BFoli5GE/) by the name 😁. 

If the response writer implements the hijacker interface, we can very easily get a hold of the underlying TCP connection and work from there. This lets us use the same port for serving both websocket and HTTP traffic.

```go
func main() {
	ws := wsh{}
	http.ListenAndServe(":8123", &ws)
}

type wsh struct {}

func (ws *wsh) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	hijacker, ok := w.(http.Hijacker)
	if !ok{
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	_, rw, err := hijacker.Hijack()
	if err != nil{
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	resp := newAcceptResponse(r)
	resp.Write(rw)
	rw.Flush()
}
```

The complete code for the above example can be found [here](https://github.com/ajsqr/websocket-implementation-examples/blob/master/hijacker-example/main.go).

In the next post, we will see how websocket messages are fragmented and sent across the TCP connection. Stay tuned!

## Sources

1. [Upgrade](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Upgrade)

2. [GoDoc](https://go.dev/blog/godoc)


