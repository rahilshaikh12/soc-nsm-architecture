---
layout: page
title: Zeek
---

To complement Suricata's signature-based detection, I deployed Zeek to capture rich, protocol-level network forensics. Because Zeek is not maintained in Ubuntu’s default repositories, I pulled the package directly from the official Zeek OpenSUSE repository.

I added the repository and the GPG signing key, then installed the application:

```bash
echo 'deb [http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/](http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/) /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL [https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key](https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key) | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install zeek
```

To ensure the system could seamlessly execute Zeek commands, I appended the installation directory to my environment path (`echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc && source ~/.bashrc`) and verified the installation by checking the version.

## Sensor Configuration & JSON Formatting

Zeek must listen on the exact same stealth interface as Suricata to inspect the mirrored SPAN traffic. I edited `/opt/zeek/etc/node.cfg` and bound the `interface=` variable to my dedicated capture vNIC.

By default, Zeek writes its output in a tab-separated `.log` format. Because I intended to forward these logs to Wazuh, I needed them structured as JSON to ensure the SIEM's decoders could parse them efficiently. I enforced this by appending `@load policy/tuning/json-logs.zeek` to the bottom of the `/opt/zeek/share/zeek/site/local.zeek` configuration file. Once configured, I initialized the sensor using the `sudo zeekctl deploy` command.

## SIEM Integration & Verification

With Zeek actively generating JSON-formatted forensic data, I configured the Wazuh Manager to ingest the specific protocol logs I cared about most for network visibility. I opened `/var/ossec/etc/ossec.conf` and added individual `<localfile>` blocks for Zeek's connection, DNS, HTTP, and SSL logs.

For example, the `conn.log` block was formatted as follows:

```xml
<localfile>
  <location>/opt/zeek/logs/current/conn.log</location>
  <log_format>json</log_format>
</localfile>
```

I repeated this structure for `dns.log`, `http.log`, and `ssl.log`. After restarting the `wazuh-manager` service to apply the configuration, I validated the ingestion pipeline. By generating baseline web traffic from a separate machine and monitoring the Wazuh dashboard, I successfully confirmed that Zeek’s protocol metadata was parsing correctly alongside my Suricata alerts, completing the centralized visibility architecture.

## Troubleshooting: JSON Decoder Field Limits

After restarting the manager to apply the new Zeek localfile entries, I noticed that the `analysisd` daemon began throwing a continuous stream of `ERROR: Too many fields for JSON decoder` alerts. 

Wazuh's analysis engine has a default limit on how many fields it will parse out of a single JSON event before dropping it. Zeek's `ssl.log`, in particular, flattens TLS certificate chain data into a massive number of nested fields when serialized to JSON. This pushed the events well past Wazuh's default field limit, causing those critical SSL logs to be silently discarded.

To resolve this, I increased the maximum decoder order size. To ensure my changes persist through future Wazuh updates, I added the override directly into `/var/ossec/etc/local_internal_options.conf` rather than editing the default configuration file:

```conf
analysisd.decoder_order_size=1024
```

After restarting the `wazuh-manager` service once more, the decoder errors ceased entirely, and the Zeek protocol metadata began ingesting and parsing correctly alongside my Suricata alerts, completing the centralized visibility architecture.
