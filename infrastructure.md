---
layout: page
title: Infrastructure
---

The core security stack (Wazuh, Suricata, and Zeek) is hosted on a single Ubuntu Server virtual machine running on VMware Workstation Pro (Windows 11 host).
I went with a virtualized environment for the initial build to allow for rapid snapshotting, easy access to host-machine documentation, and streamlined troubleshooting. In the future, I will definitely migrate to a dedicated bare-metal machine to eliminate hypervisor overhead and dedicate raw hardware resources to packet processing.
To minimize resource use and simplify initial integration, all three security services run concurrently on the same Ubuntu operating system. While convenient, this introduced specific setup and networking conflicts, which are detailed in the sections below.

