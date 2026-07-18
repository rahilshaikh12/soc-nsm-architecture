---
layout: page
title: Wazuh Deployment
---

To deploy the SIEM, I utilized the Wazuh automated quick-start script. This approach efficiently provisions the Wazuh Indexer, Server, and Dashboard on a single node, which perfectly matched my all-in-one architecture plan:

```bash
curl -sO [https://packages.wazuh.com/4.14/wazuh-install.sh](https://packages.wazuh.com/4.14/wazuh-install.sh) && sudo bash ./wazuh-install.sh -a
```

## Troubleshooting Network Visibility & Assigning a Static IP

After the installation was completed, I encountered a routing issue where my host machine could not reach the newly installed Wazuh web interface.

Rather than troubleshooting the application layer, I started by verifying Layer 3 connectivity using ICMP pings between my host and the VM. This step revealed a subnet mismatch: VMware's default network adapter was set to NAT, which placed the VM inside an isolated virtual subnet rather than my local LAN.

To fix this, I bypassed the default NAT completely and built a custom dual-bridge setup in VMware's Virtual Network Editor. I mapped `VMnet0` directly to my Ethernet adapter (for the stealth SPAN capture) and `VMnet2` to my Wi-Fi adapter (for management access).

Once the `VMnet2` interface was successfully bridged and pulled an IP from my home router, I manually assigned it a static IP. Doing this after resolving the routing conflict was a critical final step; it ensures the Wazuh Manager maintains a consistent IP address for endpoint agents to report to and guarantees I never lose administrative access to the web dashboard.

## Agent Deployment

Upon verifying access to the Wazuh web interface, I navigated to the agent deployment module. From there, I generated the necessary installation commands and deployed the Wazuh agent across my designated endpoints. This successfully established a secure TLS connection between the devices and the Manager, allowing me to begin centralizing system logs, file integrity data, and security telemetry into the dashboard.
