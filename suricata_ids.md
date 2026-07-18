---
layout: page
title: Suricata IDS
---

To deploy the intrusion detection system, I installed Suricata from the official Open Information Security Foundation (OISF) repository to ensure I was running the latest stable build:

```bash
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install suricata
```

Once installed, I edited the main configuration file (`/etc/suricata/suricata.yaml`) to bind the engine to my stealth capture interface.

## Handling Dynamic WAN IP Challenges

Because my SPAN switch is positioned directly behind the ISP modem, the sensor captures raw public IP traffic before it hits the NAT of my router. Since I am on a dynamic residential ISP connection and do not have a static public IP, I could not hardcode my WAN IP into the configuration. To ensure traffic was successfully ingested regardless of my DHCP lease, I initially configured both the `HOME_NET` and `EXTERNAL_NET` variables as `"any"`.

While this solved the immediate ingestion problem, setting both variables to `"any"` limits Suricata's directional awareness, which can prevent certain directional rules from triggering. I am currently researching long-term automated solutions for dynamic WAN IP updates on edge sensors.

## Troubleshooting the Rule Engine Syntax Error

This `"any"` configuration also introduced a critical failure when I attempted to download intrusion signatures using `suricata-update`.

By default, Suricata's server variables (like `$SMTP_SERVERS` or `$DNS_SERVERS`) inherit the `$HOME_NET` value. Many intrusion rules use negation to hunt for anomalies (e.g., `!$SMTP_SERVERS` to alert on unauthorized internal mail servers). Because my `HOME_NET` was `"any"`, the variables inherited that value. Negating `"any"` (`!any`) creates a mathematical null set (NIL). This empty IP space caused a fatal syntax error, forcing the rule parser to panic and abort the rule load entirely.

To resolve this, I manually decoupled the specific service variables from `$HOME_NET` in the `suricata.yaml` file. By hardcoding those service variables to a generic internal `/24` subnet, I provided the rule engine with a valid IP space to parse, allowing the signatures to load and compile successfully.

## Sensor Validation & Alert Testing

Before integrating the sensor with the SIEM, I needed to verify that Suricata was actively inspecting mirrored traffic and successfully generating alerts. I used the `tail -f` command on the Ubuntu VM to monitor the `/var/log/suricata/fast.log` file in real-time.

To trigger a detection, I opened a terminal on a separate host within the local network and executed a simple `curl` request to `http://testmynids.org/uid/index.html` (a standard benign payload designed specifically to safely trigger IDS rules). Almost instantly, the corresponding alert populated in my active terminal session. This successfully validated that the physical SPAN configuration, the stealth vNIC, and the Suricata rule engine were all aligned and functioning correctly.

## SIEM Integration: Ingesting Suricata Alerts into Wazuh

With Suricata successfully capturing raw traffic, the next step was to ship those local alerts into my SIEM for centralized visibility. Because the Wazuh Manager and Suricata are running on the same VM, I was able to configure the Manager to ingest Suricata's logs directly from the local disk.

I edited the Wazuh Manager's core configuration file (`/var/ossec/etc/ossec.conf`) and appended the following XML block to the `<localfile>` section:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Specifying the `json` format here is critical; Suricata outputs highly structured JSON data in its `eve.json` file, and Wazuh natively contains out-of-the-box decoders capable of parsing these fields (like source IP, destination port, and signature classification) without requiring custom regex.

After saving the changes, I restarted the `wazuh-manager` service to load the new configuration into memory. Upon logging back into the Wazuh web UI, I navigated to the Threat Hunting dashboard. By applying a filter for `rule.groups: suricata`, I successfully validated that real-time edge telemetry was flowing directly into my SIEM.
