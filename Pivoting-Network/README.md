# Network Pivoting and Infrastructure Isolation Bypassing

Welcome to my hands-on repository dedicated to **Pivoting** techniques and manual traffic management in isolated networks.

This repository compiles concepts, operational logic, and network restriction bypass architectures across different layers of the OSI model — spanning from classic SSH tunneling to purpose-built pivoting tools like Ligolo-ng and Chisel.
All materials are based on real-world experience from deploying a home CTF lab.

**As this repository develops, the infrastructure will be expanded!**

---

## Lab Architecture (Network Topology)

Unless noted otherwise, all labs in this repository share the same simulated topology: three nodes split across two network segments, with a dual-homed pivot machine bridging them.

| Machine | Role | External IP | Internal IP |
|---|---|---|---|
| **Kali Linux** | Attacker machine | `192.168.122.59` | — |
| **Ubuntu-1** | Pivot host / DMZ (dual-homed) | `192.168.122.153` | `192.168.0.139` |
| **Internal target** (Ubuntu-2 / Windows, depending on the lab) | Hidden target | — | `192.168.0.53` |

```text
External network: 192.168.122.0/24

Kali Linux                         Ubuntu-1
192.168.122.59  <-------------->  192.168.122.153
 Attacker                            Pivot host
                                         |
                                    192.168.0.139
                                         |
                              Internal network: 192.168.0.0/24
                                         |
                                  Internal target
                                    192.168.0.53
```

*   **Kali Linux (Attacker):** Positioned in the outer perimeter. It has no direct access to critical internal resources.
*   **Ubuntu-1 (Pivot Machine / DMZ):** A machine at the intersection of two worlds. It features two network interfaces: one facing outward towards Kali, and the second leading into the closed internal segment.
*   **Internal target (hidden):** Located inside a completely isolated subnet. It has no internet access and is physically invisible to the attacking Kali Linux machine. Depending on the lab, this is either a second Ubuntu VM or a Windows host.

Each lab below uses this topology (or states explicitly where it diverges) and walks through a different pivoting technique.

---

## Labs

1. [SSH Tunneling](ssh-tunneling/README.md) — dynamic (SOCKS5), local, and reverse port forwarding using native SSH.
2. [Ligolo-ng](ligolo-ng/README.md) — transparent tunnel pivoting with a virtual network interface.
3. [Chisel + SOCKS5 + RDP](chisel-rdp/README.md) — reverse port forwarding, reverse SOCKS5 proxying, file transfer, and RDP access via ProxyChains.

---

**Disclaimer:** This repository is for educational purposes only, built on an isolated home lab. Never run these techniques against systems or networks you do not own or do not have explicit authorization to test.
