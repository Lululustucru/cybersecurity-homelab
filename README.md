# Cybersecurity Home Lab

A personal cybersecurity homelab designed to build practical skills in networking, virtualization, Linux administration, containerization, SIEM monitoring and network intrusion detection.

The environment is built progressively and documented step by step. The goal is not only to deploy security tools, but also to understand how they communicate, troubleshoot integration issues and validate detections with real tests.

## Current Architecture

```text
                         Internet
                            |
                       Home Router
                            |
                       WAN-EXTERNAL
                            |
                         pfSense
                  WAN: 192.168.1.x
                  LAN: 10.10.10.1/24
                    Suricata IDS
                            |
                         LAB-LAN
                       10.10.10.0/24
                            |
              +-------------+-------------+
              |                           |
              |                           |
      Ubuntu Server                  Wazuh Server
      10.10.10.10                    10.10.10.20
              |                           |
          Wazuh Agent                Wazuh Manager
              |                      Wazuh Indexer
           Docker                    Wazuh Dashboard
          /      \
   Nginx :8080  Portainer :9443
```

The environment currently contains two main monitored systems:

- an Ubuntu Docker host running Nginx and Portainer;
- a dedicated Wazuh server used for centralized security monitoring.

Suricata runs directly on pfSense and monitors traffic on the LAB-LAN network.

---

## Technologies

- Hyper-V
- pfSense CE
- Ubuntu Server 24.04 LTS
- Docker
- Docker Compose
- Nginx
- Portainer
- SSH
- Wazuh 4.14
- Suricata IDS
- ET Open rules
- syslog-ng
- rsyslog
- Nmap
- Git
- GitHub

Future components may include:

- Windows Server
- Active Directory
- Sysmon
- Kali Linux
- Vulnerable machines
- Network segmentation
- Custom detection rules
- Active Response

---

## Network Plan

| Device | Role | Address |
|---|---|---|
| pfSense | Firewall / LAN Gateway / IDS | `10.10.10.1` |
| Ubuntu Server | Docker Host / Wazuh Agent | `10.10.10.10` |
| Wazuh Server | SIEM / XDR | `10.10.10.20` |
| DHCP Pool | Dynamic Clients | `10.10.10.100-200` |

Main lab network:

```text
10.10.10.0/24
```

Gateway:

```text
10.10.10.1
```

---

## Docker Services

The Ubuntu server hosts several Docker services.

### Nginx

Nginx is exposed on:

```text
http://10.10.10.10:8080
```

The container is deployed using Docker Compose.

Configuration:

```text
docker/nginx/compose.yaml
```

### Portainer

Portainer is used for container management.

It is exposed on:

```text
https://10.10.10.10:9443
```

Configuration:

```text
docker/portainer/compose.yaml
```

---

## Wazuh SIEM

A dedicated Ubuntu Server VM hosts the Wazuh all-in-one deployment.

```text
Wazuh Server: 10.10.10.20
```

The installation includes:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

The Ubuntu Docker host is monitored by a Wazuh Agent.

```text
Hostname: webubuntu
Agent name: Ubuntu-docker
IP: 10.10.10.10
```

The agent successfully reports security and system events to the Wazuh Manager.

Detailed setup documentation:

```text
wazuh/setup.md
```

---

## Suricata IDS

Suricata runs directly on pfSense and monitors the LAB-LAN interface.

ET Open rules are used to detect suspicious network activity such as:

- network scans;
- suspicious protocol activity;
- web-related attacks;
- reconnaissance traffic.

Suricata generates structured events using EVE JSON.

Example EVE alert:

```text
ET SCAN Possible Nmap User-Agent Observed
```

Signature ID:

```text
2024364
```

Detailed Suricata documentation:

```text
suricata/setup.md
```

---

## Suricata to Wazuh Integration

Suricata alerts are centralized in Wazuh using the following pipeline:

```text
Network Traffic
      |
      v
   pfSense
      |
      v
   Suricata
      |
      | EVE JSON
      v
   eve.json
      |
      v
  syslog-ng
      |
      | TCP 5514
      v
 Wazuh Server
      |
      v
   rsyslog
      |
      v
/var/log/suricata/eve.json
      |
      v
Wazuh Logcollector
      |
      v
Wazuh JSON Decoder
      |
      v
Threat Hunting
```

The main challenge was preserving the original Suricata JSON structure during transport.

The final implementation uses raw JSON transport so Wazuh can process Suricata events using its native decoder and rules.

---

## IDS Validation

A real network scan was generated from the Ubuntu server:

```bash
sudo nmap -sS -sV -O -T4 10.10.10.1
```

Suricata detected the scan:

```text
ET SCAN Possible Nmap User-Agent Observed
```

Observed event information:

```text
Source IP:      10.10.10.10
Destination IP: 10.10.10.1
Destination:    TCP/80
Signature ID:   2024364
```

The same alert was successfully ingested by Wazuh.

Example Wazuh event:

```text
Rule ID: 86601
Rule level: 3
Description: Suricata: Alert - ET SCAN Possible Nmap User-Agent Observed
```

The alert is visible in:

```text
Threat Hunting > Events
```

This validates the complete detection chain:

```text
Nmap
  |
  v
Suricata
  |
  v
EVE JSON
  |
  v
syslog-ng
  |
  v
rsyslog
  |
  v
Wazuh
  |
  v
Threat Hunting
```

---

## File Integrity Monitoring

Wazuh File Integrity Monitoring is enabled on the Ubuntu endpoint.

Standard system directories are monitored periodically, including:

```text
/etc
/usr/bin
/usr/sbin
/bin
/sbin
/boot
```

The default periodic scan interval is:

```text
43200 seconds
```

which corresponds to 12 hours.

A dedicated realtime directory was also configured for validation:

```text
/opt/wazuh-fim-test
```

The following operations were tested:

```text
File creation
File modification
File deletion
```

Wazuh successfully detected all three events.

Observed rules:

| Event | Wazuh Rule |
|---|---|
| File added | `554` |
| Integrity checksum changed | `550` |
| File deleted | `553` |

This validates realtime endpoint file monitoring.

---

## V1 Status

The first version of the homelab currently includes:

- [x] Hyper-V virtualization
- [x] Dedicated LAB-LAN virtual network
- [x] Dedicated external WAN switch
- [x] pfSense firewall/router
- [x] LAN/WAN separation
- [x] DHCP
- [x] DHCP static mappings
- [x] Ubuntu Server
- [x] SSH administration
- [x] Docker
- [x] Docker Compose
- [x] Nginx
- [x] Portainer
- [x] Dedicated Wazuh Server
- [x] Wazuh Manager
- [x] Wazuh Indexer
- [x] Wazuh Dashboard
- [x] Wazuh Agent
- [x] Suricata IDS
- [x] ET Open rules
- [x] EVE JSON logging
- [x] Suricata to Wazuh integration
- [x] IDS detection test with Nmap
- [x] Suricata alerts visible in Wazuh Threat Hunting
- [x] File Integrity Monitoring
- [x] Realtime FIM validation
- [x] GitHub documentation

---

## Repository Structure

```text
.
├── README.md
├── .gitignore
│
├── architecture/
│   ├── architecture.md
│   └── ip-plan.md
│
├── docker/
│   ├── nginx/
│   │   ├── compose.yaml
│   │   └── index.html
│   │
│   └── portainer/
│       └── compose.yaml
│
├── suricata/
│   └── setup.md
│
├── troubleshooting/
│   └── known-issues.md
│
└── wazuh/
    └── setup.md
```

The repository structure will continue to evolve as new components are added.

---

## Troubleshooting

Problems encountered during deployment are documented rather than hidden.

Examples include:

- pfSense repository access failure;
- incorrect Hyper-V WAN connectivity;
- WAN and LAN initially using the same subnet;
- Ubuntu LVM not using the full virtual disk;
- incomplete Wazuh package cleanup;
- incorrect Wazuh Manager address on the first agent deployment;
- Suricata continuing to use an old output configuration;
- JSON corruption during Suricata log transport;
- rsyslog file permission issues;
- periodic FIM scans missing short-lived file changes.

Existing troubleshooting documentation:

```text
troubleshooting/known-issues.md
```

---

## What I Learned

This V1 required work across several layers of an infrastructure:

- virtual networking;
- routing;
- firewall configuration;
- Linux administration;
- Docker;
- SIEM deployment;
- endpoint monitoring;
- network intrusion detection;
- log transport;
- JSON parsing;
- permissions;
- service troubleshooting.

One of the main lessons was that deploying security software is only one part of the work.

The most important part is validating the complete detection pipeline and being able to identify exactly where an event is lost or altered.

For example, the Suricata integration was validated at every stage:

```text
Detection
→ EVE JSON generation
→ syslog-ng transport
→ TCP reception
→ rsyslog file output
→ JSON validation
→ Wazuh ingestion
→ Threat Hunting alert
```

---

## Next Steps

Possible improvements for V2:

- VLAN segmentation
- Windows Server deployment
- Active Directory
- Windows endpoint monitoring
- Sysmon
- Kali Linux attacker VM
- Vulnerable machines
- SSH brute-force detection
- Custom Wazuh rules
- Active Response
- MITRE ATT&CK detection mapping
- Docker log monitoring
- Custom Wazuh dashboards
- Vulnerability detection
- Honeypot deployment
- Infrastructure automation with Ansible
