---
date: '2026-07-26T21:08:52+05:30'
draft: false
title: 'Wombat Reverse Tunnel Tutorial'
---
![wombat](/images/wombat-reverse-tunnel-tutorial/wombat-fast.avif)
In this guide we'll expose a web service running on a machine behind a NAT to the public internet using Wombat.

By the end of this tutorial you'll have:

- A `wombat-server` running on a VPS
- A `wombat-agent` running on your local machine
- A TLS-encrypted tunnel connecting the two
- A local web service accessible from anywhere on the internet

If you're interested in how Wombat works internally, check out my previous post, [Building Wombat: A Reverse TCP Tunnel]({{% relref "blog/building-wombat-a-reverse-tcp-tunnel.md" %}}). This guide focuses purely on getting started.

# About Wombat

Wombat is a lightweight reverse TCP tunnel that securely exposes local TCP services behind a NAT to the public internet.

It consists of two components.

## wombat-server

The `wombat-server` runs on a publicly accessible machine, such as a VPS. It accepts incoming client connections and forwards them through a secure tunnel.

## wombat-agent

The `wombat-agent` runs on the machine hosting your local services. It establishes a persistent TLS-encrypted tunnel to the server and forwards traffic between your local service and connected clients.


# Installing Wombat

Download the latest release from GitHub, or use the guided installer on Unix-like systems.

Depending upon your requirement, the wombat-server & wombat-agent can be easily installed by getting the right binary from the project release section. For unix like systems, we can easily install it using the guided installer by running 

```.sh
curl -fsSL https://raw.githubusercontent.com/ajsqr/wombat/main/install.sh | bash
```

I'm going to run this on my mac to install wombat-agent and also on my ec2 instance for installing the wombat-server.

```.sh
Which component would you like to install?
  1) Agent
  2) Server
Choice [1-2]: 1
Downloading wombat-agent...
Extracting...
Installing to /usr/local/bin (you may be prompted for your password)...
Password:

✅ Successfully installed wombat-agent

Binary:
  /usr/local/bin/wombat-agent

Config directory:
  /Users/username/Library/Application Support/wombat

Run:
  wombat-agent version
```

I can then verify that installer worked by running 

```.sh
wombat-agent version
```

```.
wombat-agent 0.2.0 build f273555
```

Depending on the latest version during the time, you may get a different version.

## Setting up `wombat-server`

The `wombat-server` and `wombat-agent` communicate over a persistent TCP tunnel. Since this tunnel carries all traffic between your local service and the internet, Wombat requires it to be secured using TLS.

In this section we'll create a small private Certificate Authority (CA), use it to issue a certificate for the server, and configure `wombat-server` to use it.

> **Note**
>
> For production deployments you may already have your own PKI or certificates signed by a trusted CA. For this tutorial we'll generate everything ourselves so that you can get started quickly.

### Create a working directory

On the machine running `wombat-server`, create a directory to store the certificates.

```bash
mkdir certs
cd certs
```

---

## Generate a Certificate Authority

First, generate a private key for the Certificate Authority.

```bash
openssl genrsa -out ca.key 4096
```

Next, create a self-signed CA certificate.

```bash
openssl req \
    -x509 \
    -new \
    -nodes \
    -key ca.key \
    -sha256 \
    -days 3650 \
    -out ca.crt \
    -subj "/CN=Wombat CA"
```

This CA will be used to sign the server certificate. Later, we'll copy `ca.crt` to the machine running `wombat-agent` so it can verify the server during the TLS handshake.

---

## Generate the server certificate

Generate the server's private key.

```bash
openssl genrsa -out server.key 4096
```

Create a file named `server.cnf`.

```ini
[req]
default_bits = 4096
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = localhost

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```

The Common Name (`CN`) and Subject Alternative Name (`SAN`) specify the hostname that the certificate is valid for. In this tutorial we'll use `localhost`, so the agent must later be configured with the same server name.

Generate a Certificate Signing Request.

```bash
openssl req \
    -new \
    -key server.key \
    -out server.csr \
    -config server.cnf
```

Finally, sign the server certificate using the CA we created earlier.

```bash
openssl x509 \
    -req \
    -in server.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out server.crt \
    -days 365 \
    -sha256 \
    -extensions req_ext \
    -extfile server.cnf
```

Your directory should now contain the following files.

```text
certs/
├── ca.crt
├── ca.key
├── ca.srl
├── server.cnf
├── server.crt
├── server.csr
└── server.key
```

Only two of these files are required by `wombat-server`:

- `server.crt` — The TLS certificate presented to connecting agents.
- `server.key` — The private key corresponding to the certificate.

The `ca.crt` file will be copied to the machine running `wombat-agent` in the next section.

---

## Configure `wombat-server`

Wombat automatically looks for its configuration file in the platform's user configuration directory.

| Platform | Configuration Directory |
|----------|--------------------------|
| Linux | `~/.config/wombat/` |
| macOS | `~/Library/Application Support/wombat/` |
| Windows | `%AppData%\wombat\` |

Create a file named `server-config.json`.

```json
{
  "certPath": "/home/ec2-user/certs/server.crt",
  "keyPath": "/home/ec2-user/certs/server.key",
  "tunnels": [
    {
      "name": "echo",
      "tunnel": "0.0.0.0:4001",
      "public": "0.0.0.0:8001",
      "tokenName": "echo"
    }
  ]
}
```

Let's briefly understand what each field means.

| Field | Description |
|------|-------------|
| `certPath` | Path to the server TLS certificate. |
| `keyPath` | Path to the server private key. |
| `name` | A unique name identifying the tunnel. |
| `tunnel` | Address where the agent connects to establish the tunnel. |
| `public` | Address where internet clients connect. |
| `tokenName` | Name of the environment variable containing the shared secret. |

In this configuration:

- Agents establish a TLS tunnel on **port 4001**.
- Internet clients connect to **port 8001**.
- Requests received on port `8001` will eventually be forwarded through the tunnel to a local service running on the machine hosting `wombat-agent`.

We're almost ready to start the server. Before we do that, we'll configure the agent and copy the CA certificate so it can verify the server's identity.

## Setting up `wombat-agent`

With the server configured, let's set up the agent.

The `wombat-agent` runs alongside the service you want to expose. It establishes a persistent TLS connection to `wombat-server` and forwards traffic between your local service and connected internet clients.

Unlike the server, the agent doesn't require its own certificate. Instead, it verifies the server's identity using the Certificate Authority (CA) certificate we generated earlier.

---

## Copy the CA certificate

Copy the `ca.crt` file from the server to the machine running `wombat-agent`.

How you transfer the file is entirely up to you. For example, using `scp`:

```bash
scp ec2-user@<your-server-ip>:~/certs/ca.crt ~/certs/
```

After copying the file, your local certificate directory should look something like this.

```text
certs/
└── ca.crt
```

This certificate will be used to verify the identity of `wombat-server` whenever the agent establishes a TLS connection.

---

## Configure `wombat-agent`

Like the server, the agent automatically looks for its configuration file inside the Wombat configuration directory.

| Platform | Configuration Directory |
|----------|--------------------------|
| Linux | `~/.config/wombat/` |
| macOS | `~/Library/Application Support/wombat/` |
| Windows | `%AppData%\wombat\` |

Create a file named `agent-config.json`.

```json
{
  "caCertPath": "/Users/username/certs/ca.crt",
  "serverName": "localhost",
  "tunnels": [
    {
      "name": "echo",
      "tunnel": "<your-server-ip>:4001",
      "local": "127.0.0.1:8080",
      "tokenName": "echo"
    }
  ]
}
```

Let's walk through each field.

| Field | Description |
|------|-------------|
| `caCertPath` | Path to the CA certificate used to verify the server. |
| `serverName` | Expected hostname in the server's TLS certificate. |
| `name` | Name of the tunnel. This must match the server configuration. |
| `tunnel` | Address of the tunnel listener on `wombat-server`. |
| `local` | Local service that Wombat should expose. |
| `tokenName` | Name of the environment variable containing the shared secret. |

There are a few important things to note.

### `serverName`

Earlier, when generating the server certificate, we configured the certificate with:

```ini
CN = localhost

[alt_names]
DNS.1 = localhost
```

Because of this, the agent must use:

```json
"serverName": "localhost"
```

During the TLS handshake, the agent verifies that the certificate presented by the server is valid for this hostname. If they don't match, the connection will be rejected.

### `name`

The tunnel name uniquely identifies a tunnel.

Both the server and the agent must use the same tunnel name.

```json
"name": "echo"
```

### `tunnel`

This is the address of the tunnel listener running on `wombat-server`.

```json
"tunnel": "<your-server-ip>:4001"
```

Replace `<your-server-ip>` with the public IP address (or domain name) of your VPS.

### `local`

The `local` field specifies the service you want to expose.

```json
"local": "127.0.0.1:8080"
```

In this tutorial we'll expose a simple HTTP server running on port `8080`, but this can point to any TCP service, including:

- Web applications
- REST APIs
- SSH
- PostgreSQL
- Redis
- MQTT
- Game servers

Wombat is completely protocol agnostic—it simply forwards TCP streams.

---

## Configure the shared secret

In addition to TLS, every tunnel is protected using a shared secret.

Notice that both configuration files contain:

```json
"tokenName": "echo"
```

This is **not** the secret itself.

Instead, it tells Wombat which environment variable to read when authenticating the tunnel.

On both the server **and** the agent, define the same environment variable.

### Linux / macOS

```bash
export echo="correct-horse-battery-staple"
```

### Windows PowerShell

```powershell
$env:echo="correct-horse-battery-staple"
```

When the agent connects, both sides independently read the value of the `echo` environment variable and verify that they match before allowing the tunnel to become active.

With both configuration files complete, we're finally ready to start the server, connect the agent, and expose our first local service to the internet.

# Running Wombat

With both components configured, we're finally ready to establish the tunnel.

## Configure your firewall

Our server configuration exposes two ports.

| Port | Purpose |
|------|---------|
| **4001** | Tunnel connection from `wombat-agent` |
| **8001** | Public endpoint exposed to internet clients |

The **public port** (`8001`) must be reachable from the internet.

The **tunnel port** (`4001`) should only accept connections from machines running `wombat-agent`. If you're deploying to AWS, Azure, GCP, or another cloud provider, it's recommended to restrict this port in your security group or firewall to trusted IP addresses instead of exposing it publicly.

For this tutorial, make sure both ports are reachable so the agent can establish the tunnel.

---

## Start `wombat-server`

On your VPS, simply run:

```bash
wombat-server run
```

Once the agent connects, you should see output similar to the following.

```json
{"time":"2026-07-27T06:40:19.323017348Z","level":"INFO","msg":"creating tunnel connection","channel":"echo"}
{"time":"2026-07-27T06:40:21.647700101Z","level":"INFO","msg":"authenticating tunnel connection","channel":"echo"}
{"time":"2026-07-27T06:40:21.696547556Z","level":"INFO","msg":"successfully authenticated tunnel","channel":"echo"}
{"time":"2026-07-27T06:40:21.696575415Z","level":"INFO","msg":"successfully established a tunnel","channel":"echo"}
```

At this point the server is ready to proxy traffic through the secure tunnel.

---

## Start a local service

Before starting the agent, let's create a simple web server to expose.

Python ships with a lightweight HTTP server that's perfect for testing.

```bash
python3 -m http.server 8080
```

You can verify it's running by visiting:

```text
http://localhost:8080
```

---

## Start `wombat-agent`

On the machine hosting your local service, run:

```bash
wombat-agent run
```

If everything is configured correctly, you'll see:

```json
{"time":"2026-07-27T12:10:21.530942+05:30","level":"INFO","msg":"attempting to connect to the tunnel","channel":"echo"}
{"time":"2026-07-27T12:10:21.630683+05:30","level":"INFO","msg":"authenticating tunnel","channel":"echo"}
{"time":"2026-07-27T12:10:21.630767+05:30","level":"INFO","msg":"successfully authenticated tunnel connection","channel":"echo"}
{"time":"2026-07-27T12:10:21.630778+05:30","level":"INFO","msg":"successfully established tunnel connection","channel":"echo"}
```

Congratulations! 🎉

Your VPS and local machine are now connected by a secure TLS tunnel.

---

## Common authentication error

If you instead see an error like:

```json
{"time":"2026-07-27T12:09:42.4269+05:30","level":"ERROR","msg":"tunnel connection failed","channel":"echo","error":"error during handshake : token not set"}
```

it means the shared secret could not be found.

Double-check that you've exported the environment variable specified by `tokenName` in both your `server-config.json` and `agent-config.json`.

For example, if your configuration contains:

```json
"tokenName": "echo"
```

then both the server and the agent must have an environment variable named `echo`.

Linux/macOS:

```bash
export echo="correct-horse-battery-staple"
```

Windows PowerShell:

```powershell
$env:echo="correct-horse-battery-staple"
```

After setting the environment variable, restart the component and try again.

---

## Test the tunnel

With both components running, open a browser and visit:

```text
http://<your-server-ip>:8001
```

Or use `curl`:

```bash
curl http://<your-server-ip>:8001
```

Although the request is sent to your VPS, Wombat transparently forwards it through the encrypted tunnel to the web server running on your local machine.

If everything has been configured correctly, you should see the directory listing served by Python's HTTP server.

Congratulations! You've successfully exposed a local service behind a NAT to the public internet using Wombat.