# Troubleshooting

This file documents the main issues encountered while building the homelab and the solutions that were validated.

## pfSense installer unable to access Netgate repositories

### Symptoms

The pfSense installer started correctly but failed when downloading repository data.

### Cause

The pfSense WAN interface was initially attached to the Hyper-V Default Switch.

### Solution

A dedicated external Hyper-V switch named `WAN-EXTERNAL` was created and attached to the physical network adapter.

Final pfSense interfaces:

- WAN -> WAN-EXTERNAL
- LAN -> LAB-LAN

---

## WAN and LAN on the same subnet

### Symptoms

pfSense networking behaved incorrectly because both interfaces were initially using the same subnet.

Initial configuration:

- WAN: `192.168.1.x/24`
- LAN: `192.168.1.1/24`

### Cause

The LAN network overlapped with the physical home network.

### Solution

The homelab LAN was moved to a dedicated subnet:

- LAN: `10.10.10.1/24`
- LAB-LAN: `10.10.10.0/24`

This cleanly separates the homelab network from the home network.

---

## Wazuh installation failed with "No space left on device"

### Symptoms

The Wazuh all-in-one installation failed because the root filesystem ran out of space.

### Cause

The Hyper-V virtual disk was configured with 60 GB, but Ubuntu LVM had only allocated approximately 28 GB to the root logical volume.

A larger virtual disk does not automatically mean the root filesystem is using the full capacity.

### Diagnosis

```bash
df -h /
lsblk
sudo vgs
sudo lvs
```

### Solution

The logical volume was extended to use all available free space:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

After the change:

```text
Root filesystem: ~56 GB
Free space: ~47 GB
```

---

## Incomplete Wazuh package cleanup after failed installation

### Symptoms

The `wazuh-manager` package remained partially installed and could not be removed normally.

### Cause

The failed installation left Debian package scripts referencing Wazuh files and directories that had already been removed.

### Diagnosis

The removal scripts were inspected using:

```bash
sudo bash -x /var/lib/dpkg/info/wazuh-manager.prerm remove
```

and:

```bash
sudo bash -x /var/lib/dpkg/info/wazuh-manager.postrm remove
```

### Solution

The package scripts were corrected so they checked that files and directories existed before attempting to access them.

The package could then be removed and Wazuh reinstalled cleanly.

---

## Wazuh agent configured with the wrong manager address

### Symptoms

The Ubuntu Wazuh Agent did not connect to the Wazuh Manager.

### Cause

The agent was initially configured to connect to:

```text
10.10.10.10
```

This is the Ubuntu Docker host itself.

### Solution

The manager address was corrected to:

```text
10.10.10.20
```

The agent service was then restarted and appeared as `Active` in Wazuh.

---

## Suricata GUI configuration did not match the running process

### Symptoms

pfSense showed the EVE output configured as a file, but Suricata continued behaving as if it were using the previous syslog output.

### Cause

The active Suricata process was still running with the older configuration.

A configuration reload was not enough to replace the active output configuration.

### Diagnosis

The active process was identified with:

```sh
ps auxww | grep '[s]uricata'
```

Open files were inspected with:

```sh
procstat -f <PID> | grep eve.json
```

Before the fix, the expected `eve.json` file was not opened by the process.

### Solution

The Suricata instance on the LAN interface was fully stopped and started again.

After the restart:

```sh
procstat -f <PID> | grep eve.json
```

showed the active EVE file:

```text
/var/log/suricata/suricata_hn139062/eve.json
```

This confirmed that the running process was now using the file-based EVE output.

---

## Suricata JSON was modified during transport

### Symptoms

Suricata alerts reached the Wazuh server, but Wazuh could not decode them correctly as JSON.

### Cause

The original EVE JSON was forwarded through normal syslog formatting.

A syslog header was added before the JSON payload, so the resulting line was no longer a pure JSON object.

### Diagnosis

The received event contained metadata before the JSON payload instead of starting directly with:

```text
{
```

The JSON was tested with:

```bash
sudo tail -n 1 /var/log/suricata/eve.json | jq .
```

Malformed lines produced a parsing error.

### Solution

The event was transported as raw JSON.

On pfSense, syslog-ng reads the EVE files with:

```text
flags(no-parse)
```

and sends only the message payload using:

```text
template("${MESSAGE}\n")
```

On the Wazuh server, rsyslog writes:

```text
%rawmsg%
```

directly to:

```text
/var/log/suricata/eve.json
```

Validation:

```bash
sudo tail -n 1 /var/log/suricata/eve.json | jq .
```

A formatted JSON object confirms that the payload is intact.

---

## rsyslog received Suricata traffic but did not write events to disk

### Symptoms

TCP traffic from pfSense reached the Wazuh server on port `5514`, but the Suricata destination file remained empty.

### Diagnosis

Network reception was confirmed with:

```bash
sudo tcpdump -ni any tcp port 5514
```

The rsyslog journal showed the output action being repeatedly suspended and resumed.

### Cause

The destination file had incompatible ownership and permissions.

The file had initially been created as:

```text
root:root
```

while rsyslog was operating as the `syslog` user.

### Solution

The rsyslog output configuration was set to create the file with:

```text
Owner: syslog
Group: wazuh
Mode: 0640
```

Relevant configuration:

```text
fileOwner="syslog"
fileGroup="wazuh"
fileCreateMode="0640"
```

This allows rsyslog to write the file while Wazuh can read it.

---

## Wazuh File Integrity Monitoring test initially showed no event

### Symptoms

A file was created, modified and deleted under a monitored path, but no FIM event appeared immediately in Wazuh.

### Cause

The monitored directories were using periodic FIM scans.

The configured frequency was:

```xml
<frequency>43200</frequency>
```

This corresponds to 12 hours.

A short-lived file can therefore be created and deleted entirely between two scans.

### Solution

A dedicated realtime test directory was created:

```text
/opt/wazuh-fim-test
```

The Wazuh Agent configuration was extended with:

```xml
<directories realtime="yes" report_changes="yes">/opt/wazuh-fim-test</directories>
```

After restarting the Wazuh Agent, creation, modification and deletion were detected immediately.

Observed Wazuh rules:

| Event | Rule |
|---|---:|
| File added | `554` |
| File modified | `550` |
| File deleted | `553` |

---

## Suricata rule parsing warnings

### Symptoms

Suricata reported rule parsing errors for several PostgreSQL-related rules during startup.

### Cause

Those specific ET Open rules used PostgreSQL detection features that were not available in the current Suricata build/configuration.

### Impact

The errors did not prevent Suricata from starting.

Thousands of other rules loaded successfully and IDS detections continued to work.

### Decision

The issue was treated as non-blocking for V1 because:

- Suricata started successfully;
- the engine monitored the LAN interface;
- Nmap activity triggered a valid ET Open alert;
- the alert was successfully ingested by Wazuh.

The unsupported PostgreSQL rules can be reviewed or disabled later if needed.

---

## Main troubleshooting lesson

For multi-stage security pipelines, each component should be validated independently.

The Suricata-to-Wazuh integration was debugged in this order:

```text
1. Confirm Suricata generated the alert
2. Confirm EVE JSON was written
3. Confirm syslog-ng read the file
4. Confirm TCP traffic reached port 5514
5. Confirm rsyslog wrote the destination file
6. Validate JSON with jq
7. Confirm Wazuh read the JSON
8. Confirm the alert appeared in Threat Hunting
```

This made it possible to identify the exact stage where data was being lost or modified instead of changing multiple configurations at the same time.
