# SSH Pivoting Techniques: Dynamic, Local, and Reverse Port Forwarding

This guide demonstrates how to perform network pivoting using native SSH functionality. We will cover three fundamental tunneling methods used to bypass firewalls, route traffic through pivot hosts, and access isolated internal networks.

---

## Initial Network Setup

Before jumping into the tunnels, we need to configure our network interface to establish connectivity with our initial pivot target.

```bash
# Identify your active network interface name
ip a 

# Assign an IP address to the target interface
sudo ip addr add 10.10.10.x/24 dev enp0s8

# Bring the network interface UP
sudo ip link set up enp0s8
```

> **The Scenario:** An attacker has compromised a boundary machine (the pivot/foothold) that has dual network interfaces. This pivot host can talk to both our external attacking machine and an isolated internal network that contains our target infrastructure.

---

## Method 1: Dynamic Port Forwarding (SOCKS5 Proxy)

Dynamic port forwarding allows you to use your pivot host as a **SOCKS5 proxy server**. Any traffic routed through this proxy will originate from the pivot host, granting you full routing access to the internal network it resides in.

### 1. Configure Proxychains
We will use `proxychains4` to force third-party tools (like `nmap`) through our SSH tunnel. First, we need to append our proxy settings to the configuration file.

*Note: Make sure to comment out the default Tor proxy entry (`# socks4 127.0.0.1 9050`) at the bottom of `/etc/proxychains4.conf` first.*

```bash
echo "socks5 127.0.0.1 1080" | sudo tee -a /etc/proxychains4.conf
```

### 2. Establish the Dynamic Tunnel
From your attacking machine (e.g., Kali/Arch), execute the following command to spin up the SOCKS proxy on local port `1080`:

```bash
ssh -D 1080 host@<pivot_ip>
```

### 3. Network Reconnaissance via Proxychains
The tunnel is now fully established. You can now prefix any standard CLI network tool with `proxychains` to scan and interact with the hidden internal network:

```bash
proxychains nmap -Pn -F --min-rate 4000 <internal_target_ip>
```

---

## Method 2: Local Port Forwarding

Local port forwarding is used when you want to redirect a specific port from an internal machine back to a port on your local attacking machine. 

### The Scenario
An internal target machine (`Ubuntu` at `192.168.1.9`) is running a custom Python script that hosts a web server on port `80 (HTTP)`. We cannot access this web server directly from our attacking machine, but our pivot host can. 

Instead of dealing with the latency and overhead of running a full desktop browser through proxychains, we will map that remote port directly to our `localhost`.

### 1. Establish the Local Tunnel
Execute this command on your attacking machine:

```bash
ssh -L 8080:192.168.1.9:80 host@<pivot_ip>
```

> **How it works:** This instructs your attacking machine to open local port `8080`. Any traffic sent to `localhost:8080` will be encrypted, encapsulated through the SSH tunnel to the pivot host, and seamlessly forwarded directly to port `80` on the target machine (`192.168.1.9`).

### 2. Verify the Tunnel Socket
Open a new terminal window on your machine and verify that port `8080` is successfully listening for connections:

```bash
ss -tulnp | grep 8080
```

### 3. Accessing the Resource
Open any browser on your attacking machine and navigate to:
**`http://127.0.0.1:8080`**

You now have direct, fast access to the isolated Python web server.

---

## Method 3: Reverse Port Forwarding

Reverse port forwarding (or remote port forwarding) is utilized when you need an internal target machine to connect back to your attacking machine (e.g., catching a Reverse Shell), but the target cannot see your machine due to firewalls or NAT configurations.

### The Scenario
We want the hidden internal host (`10.10.10.1`) to send a payload back to us. While the internal host cannot route out to our attacking box, it *can* route to the pivot host. We will force the pivot host to open a port and relay all incoming connections backward to our machine.

### 1. Establish the Remote Tunnel
Run this command from your attacking machine. To make it highly practical for operations, we will add the `-f` (run in background) and `-N` (do not execute remote commands) flags so it acts as a silent background daemon:

```bash
ssh -f -N -R 4444:localhost:4444 host@<pivot_ip>
```

> **How it works:** The pivot host opens port `4444` on its own interface. Whenever an internal machine talks to the pivot host on port `4444`, the pivot host forwards that traffic back through the pre-established SSH connection directly to port `4444` on our local machine.

### 2. Simulating and Verifying the Attack

#### Step A: Fire up a local listener
On your attacking machine, spawn a `netcat` listener on port `4444` to catch incoming connections:

```bash
nc -lvnp 4444
```

#### Step B: Check listener status (Optional)
To verify your local socket is ready to accept incoming traffic relayed by the tunnel:

```bash
ss -tulnp | grep 4444
```

#### Step C: Trigger connection from the Target
From the perspective of the target machine (`10.10.10.1`) inside the hidden network, send a test payload directly to the **pivot host's IP** on port `4444`:

```bash
echo "Pivoting works!" | nc <pivot_ip> 4444
```

### Verification
Check your attacking machine's `netcat` terminal. If the text `"Pivoting works!"` immediately prints to your screen, your packets successfully traversed the network topology via the encrypted reverse pipeline. You are now ready to upgrade this connection to a fully interactive reverse shell.

