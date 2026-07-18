---
layout: page
title: Infrastructure
---

The core security stack (Wazuh, Suricata, and Zeek) is hosted on a single Ubuntu Server virtual machine running on VMware Workstation Pro (Windows 11 host).

I selected a virtualized environment for the initial build to allow for rapid snapshotting, easy access to host-machine documentation, and streamlined troubleshooting. In the future, I plan to migrate to a dedicated bare-metal machine to eliminate hypervisor overhead and dedicate raw hardware resources to packet processing.

To minimize resource use and simplify initial integration, all three security services run concurrently on the same Ubuntu operating system. While convenient, this introduced specific setup and networking conflicts, which are detailed in the sections below.

## Network Architecture & Edge Visibility

To achieve full visibility into raw internet traffic without introducing the cost or complexity of an enterprise edge router (e.g., pfSense/OPNsense), I utilized a highly targeted Port Mirroring (SPAN) topology.

A managed switch is placed physically inline between the ISP modem and the home router, operating strictly at Layer 2 to avoid exposing the management interface to the WAN. The switch is configured to mirror all ingress and egress traffic crossing the WAN boundary and forward it to the IDS sensor.

### Physical Switch Topology

![Home Lab Network Architecture Diagram](/assets/images/network_topology.png)

* **Port 1 (SPAN Source):** Connected to the ISP Modem (captures raw, unfiltered internet traffic).
* **Port 2 (Bridged):** Connected to the router's WAN interface.
* **Port 3 (SPAN Destination):** Connected to the dedicated capture NIC on the VMware host machine (configured as a stealth listener with no IP bindings).

To ensure the router can successfully communicate with the modem, Ports 1 and 2 must share a Layer 2 broadcast domain. However, securing this configuration requires strict logical isolation. In this deployment, the WAN-facing cluster (Ports 1, 2, and 3) was assigned to **VLAN 3**, while the internal management connections (Ports 4 and 5) were assigned to **VLAN 2**. This strict VLAN segregation is a critical security imperative; it prevents Layer 2 broadcast leakage and ensures that raw, untrusted internet traffic cannot bypass the router's firewall to enter the trusted LAN via the switch's backplane.

### Out-of-Band Switch Provisioning

To prevent network disruption and IP conflicts, I had to make configuration changes to the switch before being deployed into the live environment. Out of the box, the switch defaults to a static IP of `192.168.0.1`. Because my machine was disconnected from the main network, it defaulted to an APIPA address (`169.254.x.x`), which cannot route to the switch's subnet. To establish communication, I manually assigned a static IP within the switch's `/24` subnet (e.g., `192.168.0.10`).

Once access to the management web UI was established, I permanently changed the switch’s management IP to `192.168.0.5`. This prevents a Layer 3 IP collision, as the factory default (`192.168.0.1`) is the same as the default gateway address of the upstream home router.

## Virtual Machine Network Configuration

To support both management access and secure traffic ingestion, the Ubuntu VM is configured with a dual-vNIC (Virtual Network Interface Card) architecture:

* **vNIC 1 (Management):** Bridged to the host's Wi-Fi adapter, receiving an internal IP address from the home router. This allows for secure SSH access, Wazuh UI access, and software updates from behind the local firewall.
* **vNIC 2 (Sensor/Capture):** Bridged directly to the host's dedicated Ethernet adapter (connected to the switch's SPAN destination port).

**Security Imperative:** Because vNIC 2 sits strictly outside the router's firewall perimeter, it is exposed to raw, untrusted internet traffic. To mitigate the risk of compromise, the IP bindings (IPv4/IPv6) on this interface were completely disabled at the OS level. It operates purely as an unrouted, stealth listener, ensuring Suricata and Zeek can ingest mirrored frames without exposing the host to external attacks.
