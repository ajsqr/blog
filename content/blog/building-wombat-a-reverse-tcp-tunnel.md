---
date: '2026-07-22T18:55:32+05:30'
draft: false
title: 'Building Wombat - A Reverse TCP Tunnel'
---

![wombat](/images/building-wombat-a-reverse-tcp-tunnel/wombat.jpg "Photo by pen_ash on pixabay")

## The Backstory

For a while now, I have been tinkering with setting up my own homelab. The challenge with this was almost always one thing—accessing my private network over the internet.

Most of the systems I had in my lab weren't meant to be online 24x7. They were prototypes, demos, or tests. I could have easily run these services in the cloud and turned them off whenever I wasn't using them, but where's the fun in that? It also helped that I had more than enough hardware sitting there idle.

I then started looking into making my hardware reachable over the internet. Even though it was possible to achieve that with some port forwarding, it felt like an entire topic of its own. Besides, it was potentially very easy to misconfigure. Naturally, the next option I had was using a service like ngrok or Cloudflare Tunnel. I was hooked by how easy and reliable those services were, and I wanted to understand how they worked under the hood. As I understood more about how tunneling software worked, I couldn't help but create my own version of it, and so Wombat was born.

## Tunnel Vision

So the objective is simple: expose locally running services over the internet without network configuration or third-party services. If we could somehow let incoming requests reach the local server, and if we could send the responses back, we would have achieved our goal. As it turns out, that "somehow" is a tunnel.

Due to NAT, devices on a private network generally cannot accept unsolicited incoming connections from the internet. In the real world, the clients are on the internet, and they cannot directly reach our local network.

But what if the connection was initiated by our local system? If there was an open connection to the internet, a client could connect to it. If this connection was kept alive, we could use it to exchange data between external clients and the local service. This would mean we have two components:

1. A server running on the public internet (aka server)
2. A helper service running on the local system (aka agent)

Once the server starts listening, the helper service can connect to it. After that, it's just a matter of exchanging data between them. The server can start accepting external clients and instruct the local service to open local connections. It can also relay data for each session. To make this communication simpler, we will need a protocol.

## The Protocol

Now that we have solved the transport mechanism, we need to use it to transport streams between external clients and local services. Just sending data streams to and fro won't cut it. We need to make sure each data stream reaches the intended recipient.

The core idea was to send metadata frames and data frames between the server and the agent so they can make informed routing decisions. If the server accepts a new external connection, it can inform the agent about it. Upon receiving this information, the agent can create a proxy connection to the local service. If these connections are named, we can then use those names as identifiers in the data frames.

```go
// Wombat Frame
type Frame struct {
	// Ident defines the type of frame.
	// It can signal opening or closing a connection,
	// a data frame, a ping frame, etc.
	Ident FrameIdentifier

	// ConnectionID identifies a client connection.
	// It is used to multiplex multiple client streams
	// through a single tunnel.
	ConnectionID uint32

	// PayloadSize defines the size of the payload.
	PayloadSize uint32

	// Payload contains the frame data.
	Payload []byte
}
```

Similarly, when a connection is closed, the server or the agent can inform the other and close the corresponding local connection. In between, data is streamed through the tunnel.

![wombat](/images/building-wombat-a-reverse-tcp-tunnel/wombat-arch.png "Wombat architecture")

From the architecture diagram, it's clear that both ends of the tunnel are quite similar. Perhaps we could implement something generic for managing a tunnel endpoint. Maybe something that can deal with dispatching frames.

## The Dispatcher and the Mailman

It is helpful to think of a tunnel endpoint as a mailman. The mailman is not aware of what is being sent. It only knows that its job is to forward packages (aka frames) to the other end and deliver incoming packages (aka frames) to the correct session.

The client packages the data in an envelope (aka a frame) and dispatches it. The tunnel endpoint is prepared to receive the frame.

When the endpoint receives a frame, it has enough information to deliver it to the correct session—without ever knowing what the payload contains.

![wombat](/images/building-wombat-a-reverse-tcp-tunnel/disptcher.png "Wombat dispatcher")

## Putting It All Together

Frames give us a way to package streamed data and, with connection identifiers, distinguish which session a particular frame belongs to. With a generic tunnel dispatcher, we can transport frames from one end to the other without knowing what the data is about. The agent and the server—the services that exist at either end of the tunnel—can work without ever worrying about frame transport or multiplexing. They simply consume frames from their inboxes and dispatch frames containing client data.

Putting it all together, we have a working model of a reverse tunnel. Application developers can expose their local services over the internet without worrying about how data is transported between the server and the agent.

## State of Wombat

At the time of writing this post, I have open-sourced Wombat and published its first [release](https://github.com/ajsqr/wombat/releases/tag/v0.1.0). The reverse tunnel implementation is stable and features multiple tunnels, multiplexing, concurrent sessions, and graceful session termination—enough for a beta release. I plan to add more features, with a particular focus on security.

Wombat is still evolving, but building it taught me far more about TCP, multiplexing, framing, and Go networking than I expected. What started as a weekend project gave me a deeper appreciation for the elegant ideas behind tools like ngrok and Cloudflare Tunnel.

If you're interested, the [source code](https://github.com/ajsqr/wombat) is available on GitHub. Feedback and contributions are always welcome.

## Lessons Learned

1. Understanding how packets are [delivered](https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/) to an end device.
2. Learning that during [`io.Reader.Read`](https://pkg.go.dev/io#Reader), callers should always process the `n > 0` bytes returned before considering the error.