# Network Pivoting and Infrastructure Isolation Bypassing

Welcome to my hands-on repository dedicated to **Pivoting** techniques and manual traffic management in isolated networks. 

This repository compiles concepts, operational logic, and network restriction bypass architectures across different layers of the OSI model — spanning from classic SSH tunneling to... 
All materials are based on real-world experience from deploying a home CTF lab.

**As this repository develops, the infrastructure will be expanded!**

---

## Lab Architecture (Network Topology)

The testbed is simulated in an isolated virtualization environment and consists of three nodes divided into two independent network segments:

*   **Kali Linux (Attacker):** Positioned in the outer perimeter (Home Network). It has no direct access to critical internal resources.
*   **Ubuntu-1 (Pivot Machine / DMZ):** A machine at the intersection of two worlds. It features two network interfaces: one facing outward towards Kali, and the second leading into the closed internal segment.
*   **Ubuntu-2 (Hidden Target):** Located inside a completely isolated subnet. It has no internet access and is physically invisible to the attacking Kali Linux machine.

Navigation panel:
[ssh-tunneling]()
[proxychains]()
[Ligoli-ng](https://github.com/aliasdef/Project/blob/main/Pivoting%20network/2_LIGOLO-NG.md)
