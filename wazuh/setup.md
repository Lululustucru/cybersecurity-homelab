# Wazuh Setup

## Objective

Deploy a centralized SIEM/XDR platform to monitor the hosts inside the Home Lab.

The Wazuh server currently receives two main types of security data:

- endpoint events from the Ubuntu Docker host through the Wazuh Agent;
- Suricata IDS events generated on pfSense and transported as EVE JSON.

## Architecture

```text
Ubuntu Docker Host
10.10.10.10
      |
      | Wazuh Agent
      | TCP 1514
      v
Wazuh Server
10.10.10.20
      |
      +-- Wazuh Manager
      +-- Wazuh Indexer
      +-- Wazuh Dashboard

pfSense / Suricata
10.10.10.1
      |
      | EVE JSON
      | syslog-ng
      | TCP 5514
      v
Wazuh Server
      |
      | rsyslog
      v
/var/log/suricata/eve.json
      |
      | Wazuh logcollector
      v
Threat Hunting
```

## Wazuh Server

The Wazuh server runs on a dedicated Ubuntu Server 24.04 LTS virtual machine.

Resources:

- 4 vCPU
- 8 GB RAM
- 60 GB virtual disk
- Network: LAB-LAN
- Static DHCP mapping: `10.10.10.20`

The installation uses the Wazuh all-in-one deployment.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh

sudo bash ./wazuh-install.sh -a
```

The `-a` option installs:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

## First Agent

The first monitored endpoint is the Ubuntu Docker server.

```text
Hostname: webubuntu
Agent name: Ubuntu-docker
IP: 10.10.10.10
Wazuh version: 4.14.8
```

The Wazuh agent communicates with the manager located at:

```text
10.10.10.20
```

The agent service is enabled at boot:

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Service verification:

```bash
sudo systemctl status wazuh-agent
```

## Verification

The agent appears as `Active` in the Wazuh dashboard.

The following information is successfully collected:

- System inventory
- Authentication events
- Sudo activity
- Vulnerability information
- MITRE ATT&CK mappings
- Host information

Authentication events can be inspected through the Threat Hunting interface.

## File Integrity Monitoring

Wazuh File Integrity Monitoring is enabled on the Ubuntu Docker host.

The existing `syscheck` configuration monitors standard system directories such as:

```xml
<directories>/etc,/usr/bin,/usr/sbin</directories>
<directories>/bin,/sbin,/boot</directories>
```

The periodic scan interval is:

```xml
<frequency>43200</frequency>
```

This corresponds to 12 hours.

The agent is also configured to scan when it starts:

```xml
<scan_on_start>yes</scan_on_start>
```

### Realtime FIM validation

A dedicated test directory was created:

```bash
sudo mkdir -p /opt/wazuh-fim-test
```

The following line was added inside the `<syscheck>` block of:

```text
/var/ossec/etc/ossec.conf
```

Configuration:

```xml
<directories realtime="yes" report_changes="yes">/opt/wazuh-fim-test</directories>
```

The agent was then restarted:

```bash
sudo systemctl restart wazuh-agent
```

Test sequence:

```bash
echo "version 1" | sudo tee /opt/wazuh-fim-test/test.txt

echo "version 2 - modified" | sudo tee -a /opt/wazuh-fim-test/test.txt

sudo rm /opt/wazuh-fim-test/test.txt
```

Wazuh detected all three operations:

| Operation | Wazuh rule | Description |
|---|---:|---|
| File creation | `554` | File added to the system |
| File modification | `550` | Integrity checksum changed |
| File deletion | `553` | File deleted |

The events were visible in:

```text
File Integrity Monitoring > Events
```

This validates realtime file monitoring on the Ubuntu endpoint.

## Suricata Integration

Suricata runs on pfSense and generates EVE JSON events.

The active EVE file on pfSense is:

```text
/var/log/suricata/suricata_hn139062/eve.json
```

The final transport path is:

```text
Suricata eve.json
      |
      v
syslog-ng on pfSense
      |
      | TCP 5514
      v
rsyslog on Wazuh Server
      |
      v
/var/log/suricata/eve.json
      |
      v
Wazuh logcollector
      |
      v
Threat Hunting
```

### Receiving Suricata events with rsyslog

The Wazuh server listens on TCP port `5514`.

The receiver configuration is stored in:

```text
/etc/rsyslog.d/30-suricata-remote.conf
```

Configuration:

```text
module(load="imtcp")

template(name="SuricataRaw" type="string" string="%rawmsg%\n")

ruleset(name="suricata_remote") {
    action(
        type="omfile"
        file="/var/log/suricata/eve.json"
        template="SuricataRaw"
        fileOwner="syslog"
        fileGroup="wazuh"
        fileCreateMode="0640"
    )
    stop
}

input(
    type="imtcp"
    port="5514"
    ruleset="suricata_remote"
)
```

The destination file is:

```text
/var/log/suricata/eve.json
```

The `wazuh` group can read the file while rsyslog retains write ownership.

### JSON validation

Before configuring Wazuh ingestion, the received Suricata data was validated with:

```bash
sudo tail -n 1 /var/log/suricata/eve.json | jq .
```

A correctly formatted JSON object confirms that the EVE event was preserved during transport.

### Wazuh logcollector configuration

The following block was added inside `<ossec_config>` in:

```text
/var/ossec/etc/ossec.conf
```

Configuration:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

After the change, the Wazuh Manager was restarted:

```bash
sudo systemctl restart wazuh-manager
```

### IDS validation

A detection test was generated from the Ubuntu Docker host:

```bash
sudo nmap -sS -sV -O -T4 10.10.10.1
```

Suricata generated the following alert:

```text
ET SCAN Possible Nmap User-Agent Observed
```

Relevant event information:

```text
Source IP:      10.10.10.10
Destination IP: 10.10.10.1
Destination:    TCP/80
Signature ID:   2024364
```

Wazuh successfully processed the event with its native Suricata rules.

Observed Wazuh event:

```text
Rule ID: 86601
Rule level: 3
Description: Suricata: Alert - ET SCAN Possible Nmap User-Agent Observed
```

The alert is visible in:

```text
Threat Hunting > Events
```

This validates the full pipeline:

```text
Nmap
  -> Suricata
  -> EVE JSON
  -> syslog-ng
  -> TCP 5514
  -> rsyslog
  -> Wazuh logcollector
  -> Wazuh rule
  -> Threat Hunting
```

## Troubleshooting

### Wazuh installation failed with "No space left on device"

The Hyper-V virtual disk had been configured with 60 GB.

However:

```bash
df -h /
```

showed that the root filesystem only had approximately 28 GB.

Inspection with:

```bash
lsblk
sudo vgs
sudo lvs
```

showed that Ubuntu LVM had approximately 28 GB of unused space inside the Volume Group.

The logical volume was extended with:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

After the operation:

```text
Root filesystem: ~56 GB
Free space: ~47 GB
```

### Incomplete Wazuh cleanup

The failed installation partially removed Wazuh components.

The `wazuh-manager` package remained installed because its Debian removal scripts referenced files that had already been deleted.

The package removal scripts were inspected using:

```bash
sudo bash -x /var/lib/dpkg/info/wazuh-manager.prerm remove
```

and:

```bash
sudo bash -x /var/lib/dpkg/info/wazuh-manager.postrm remove
```

The scripts were corrected to verify that files and directories existed before attempting to access them.

The remaining package was then removed successfully.

### Agent initially configured with wrong manager address

The first agent deployment mistakenly used:

```text
10.10.10.10
```

as the Wazuh manager address.

This was the Ubuntu Docker host itself.

The correct Wazuh manager address is:

```text
10.10.10.20
```

After updating the agent configuration and restarting the service, the agent successfully connected.

### Suricata JSON initially failed to decode correctly

The first integration attempt forwarded Suricata EVE events through standard syslog formatting.

The syslog header modified the message before it reached Wazuh, so the event was no longer a pure JSON object.

The final solution was to preserve the EVE payload as raw JSON throughout the transport chain.

On pfSense, syslog-ng reads the EVE file with `flags(no-parse)` and sends only the message payload.

On the Wazuh server, rsyslog writes `%rawmsg%` directly to the destination file.

The received data is then validated with `jq` before Wazuh ingests it.

### rsyslog file permissions

During testing, TCP traffic was reaching port `5514`, but rsyslog could not write to the destination file.

The file had initially been created with incompatible ownership.

The final rsyslog configuration creates the file with:

```text
Owner: syslog
Group: wazuh
Mode: 0640
```

This allows rsyslog to write the events and Wazuh to read them.

### FIM test initially produced no event

The default FIM configuration used periodic scans every 12 hours.

A file that was created, modified and deleted between two scans could therefore remain undetected.

A dedicated directory with:

```xml
realtime="yes"
```

was used to validate immediate file change detection.

## Lessons Learned

- Always verify actual filesystem size after installing Ubuntu with LVM.
- A virtual disk size does not necessarily equal the root filesystem size.
- Check `df`, `lsblk`, `vgs` and `lvs` when troubleshooting storage.
- Failed package installations can leave inconsistent package states.
- Service status alone does not guarantee correct network configuration.
- Verify the manager address and network connectivity when deploying agents.
- Validate log data at every step of a forwarding pipeline.
- Preserve structured JSON when the receiving SIEM expects JSON.
- Network connectivity does not guarantee that a service can write to its destination file.
- Periodic FIM and realtime FIM serve different monitoring needs.
