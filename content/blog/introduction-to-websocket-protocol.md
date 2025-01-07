---
date: '2025-01-07T21:28:44+05:30'
draft: false
title: 'Introduction To WebSocket Protocol'
tags: [websocket, RFC, RFC 6455]
---

## Introduction

Welcome to the first post of the **reinventing the wheel - websocket** series. In a series of posts, we will attempt to master the websocket protocol and even write our own implementation of the protocol. 

In this post we will briefly look at the websocket protocol, and the problems that it solves. 

## A Brief History Of The Internet
**January 1, 1983** is widely considered as the birthday of our internet. However, it was in 1989, while working at CERN, Tim Berners-Lee wrote a proposal to build a hypertext system over the internet. Initially called the _Mesh_, it was later renamed the _World Wide Web_ during its implementation in 1990. This was when HTTP was initially introduced. Although there were some versions before it, it was after the introduction of HTTP1/0 that the internet got more interesting. 

HTTP/1.0 was awesome. It introduced the request / response model of communication over the internet. The client would send a request to the server, and the server would send back an appropriate response and finally close the connection. This was going great, until it wasn't. 

See the problem was that, as the applications evolved, they were required to send not just one, but multiple requests and responses between them. This was problematic because each new request and response required a new HTTP (or more specifically TCP) connection. 

To fix this and to improve HTTP overall, HTTP/1.1 was introduced. Now, we were able to reuse the connection for subsequent request / response. This was great, but our problems were not over. The applications evolved yet again, and now they had a different problem. Some of the resources need to be updated constantly and for this to work, applications has to constantly poll the server for updates. This was inefficient.

## Enter Websockets

Now that HTTP/1.1 allows us to reuse the TCP connection between the client and the server, why not use the connection for bi-directional communication as well? Well, websockets did exactly that!

Websocket protocol allowed us to reuse the connection and use it for bi-directional communication. It was meant to replace long polling and the difficulties that came with it. Not just that, websockets are made to work well with existing HTTP setup. The protocol was designed to work over port 80 (HTTP) and 443 (HTTPS). There are no restrictions to setting it up on its own custom port either. When we see how the websocket handshake looks like, you will have a new found admiration for the simplicity of the protocol.

## The Websocket Protocol 

The websocket protocol has two parts, A (opening and closing) handshake and then the data transfer. 

### Opening Handshake

During the handshake, a client would initiate the process by sending a request that roughly looks like this.

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
The beauty of this opening request is that it is a valid HTTP request. It is basically an upgrade request. This means that the same port in the server can be listened for serving both HTTP requests and websocket requests. 

Depending on the servers capability to deal with websocket requests, a server can upgrade the connection to a websocket connection, or deny with a regular HTTP response.

The request header from the client carries a lot of important information. I think the most interesting header in this bunch is the `Sec-WebSocket-Key`. This is used by the server during its handshake. This is interesting because it helps the server "prove" that the server did in fact receive a handshake from the client. The server extracts the value of this header from the request and creates an accept token. Let me explain this with the help of an example.

---
#### Example

In the request above, we received `dGhlIHNhbXBsZSBub25jZQ==` as the header. 

The server will then concatenate it with a special GUID - `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`.
The resulting value will now look like `dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-
   C5AB0DC85B11` 
The server will then take the SHA-1 hash of this string and then finally base64 encode the resulting hash. 
This should give the result as `s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` - and now we have the accept token.

---
Although the other headers are just as important, this particular header was the special one. 


After a few additional validations that the server performs, it may accept the connection by performing the final part of the handshake. The handshake response from the server will look like this

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

As we can see, the server sent a HTTP response status 101. Any other response code would mean that the handshake was unsuccessful. You will also notice that the value of  `Sec-WebSocket-Accept` is the accept token that we calculated above. The client will also calculate this value from its end, and see if it matches with the one the server sent. If they are the same, we have successfully completed the handshake.


 ### Data Transfer

Data transfer between the client and server is transmitted in a sequence of frames. A frame consists of part (or in some cases, the entire) data and a whole bunch of metadata. The websocket data transfer protocol is really interesting. Lets start by looking at the structure of a frame

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+

The first bit here is the FIN bit. It is set to 1 iff its the final frame of a message. In some cases, the first frame could very well be the last frame.

The three frames that follow the FIN bit are reserved bits. The 4 bits that follow defines the opcode. Opcode is really important as it carries meaning about what the frame is really about. Using opcode, we can determine if the frame is regarding text data, binary data or some other control frame. 

As I said before, the data transfer protocol here is really interesting. The next bit, MASK, is partly responsible for this. If the masking bit is set, it means that there is a 16 bit masking key within the frame. This masking key has to be used to decode the encoded payload. The websocket protocol mandates that every frame from client to server to be masked, while every message from server to client be unmasked.

This is interesting because, if you observe the wire and capture the message sent from client to server and server to client, the captured frames will look different. 

The seven bits after the masking bit gives an idea about the payload length. The value in these can go up to 127. Values up to 125 conveys the payload length. But a value of 126 means that the payload length is represented by 2 bytes (16 bits) and a value of 127 means that the payload length is defined by an 8 byte (64 bit) unsigned integer. This means that the payload in a single frame could go up to 2^63 bytes (9,223,372,036,854,775,807 bytes ~= 9.22 exabytes 💀)

### Closing Handshake

Closing handshake is simpler compared to the opening handshake. Here, a client or server could initiate closing by sending a control frame with close opcode. Upon receiving the frame, the other party sends a close frame. After receiving this acknowledgement frame, the first party can safely close the connection. 

### Up Next
In the next post, I will talk about websocket handshake in detail and implement it using golang.

### Sources
1. [RFC 6455 The Websocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
2. [Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Evolution_of_HTTP)