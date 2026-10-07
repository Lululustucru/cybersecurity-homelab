# Homelab Architecture

## Overview

The V1 homelab is built on Hyper-V and uses pfSense as the central firewall and router.

The environment separates the physical home network from a dedicated laboratory network.

The main security components are:

- pfSense for routing and firewalling;
- Suricata for network intrusion detection;
- Ubuntu Server as the monitored Docker host;
- Wazuh as the central SIEM/XDR platform.

## Network Topology

```text
                              Internet
                                 |
                                 v
                           Home Router
                         192.168.1.0/24
                                 |
                                 |
                         Hyper-V WAN-EXTERNAL
                                 |
                                 v
                              pfSense
                    WAN: 192.168.1.x/24
                    LAN: 10.10.10.1/24
                         Suricata IDS
                                 |
                                 |
                         Hyper-V LAB-LAN
                          10.10.10.0/24
                                 |
                    +------------+------------+
                    |                         |
                    |                         |
                    v                         v
            Ubuntu Docker Host          Wazuh Server
              10.10.10.10               10.10.10.20
                    |                         |
                    |                         |
              Wazuh Agent               Wazuh Manager
                    |                    Wazuh Indexer
                 Docker                  Wazuh Dashboard
                /      \
               /        \
        Nginx :8080   Portainer :9443
```

## Hyper-V Networks

### WAN-EXTERNAL

`WAN-EXTERNAL` is an external Hyper-V virtual switch connected to the physical network adapter.

It provides pfSense WAN access to the home network and Internet.

```text
Home network: 192.168.1.0/24
pfSense WAN:  192.168.1.x
```

### LAB-LAN

`LAB-LAN` is the dedicated virtual network used by the homelab.

```text
Network: 10.10.10.0/24
Gateway: 10.10.10.1
```

The laboratory network is routed through pfSense.

## Addressing

| Device | Address | Role |
|---|---|---|
| pfSense | `10.10.10.1` | Firewall, router, gateway, Suricata IDS |
| Ubuntu Docker Host | `10.10.10.10` | Docker host and monitored Linux endpoint |
| Wazuh Server | `10.10.10.20` | SIEM/XDR server |
| DHCP Clients | `10.10.10.100-200` | Dynamic LAB-LAN clients |

The detailed addressing plan is available in:

```text
architecture/ip-plan.md
```

## pfSense

pfSense is the central network component.

Its V1 roles are:

- routing between the laboratory and external network;
- default gateway for LAB-LAN;
- DHCP service;
- firewall policy enforcement;
- Suricata IDS host.

LAN address:

```text
10.10.10.1/24
```

## Ubuntu Docker Host

The monitored Ubuntu server uses:

```text
IP:       10.10.10.10
Hostname: webubuntu
```

Its main roles are:

- Linux administration target;
- Docker host;
- Nginx web server;
- Portainer container management;
- Wazuh endpoint.

Services currently exposed inside the lab:

| Service | Port |
|---|---:|
| SSH | `22/TCP` |
| Nginx | `8080/TCP` |
| Portainer | `9443/TCP` |

The installed Wazuh agent is registered as:

```text
Ubuntu-docker
```

## Wazuh Server

The Wazuh server uses:

```text
IP: 10.10.10.20
```

It is a dedicated Ubuntu Server VM running the Wazuh all-in-one deployment:

- Wazuh Manager;
- Wazuh Indexer;
- Wazuh Dashboard.

The server receives endpoint telemetry from the Ubuntu Docker host and IDS events from Suricata.

## Endpoint Monitoring Flow

The Ubuntu endpoint sends security events directly to the Wazuh Manager.

```text
Ubuntu Docker Host
10.10.10.10
      |
      | Wazuh Agent
      | TCP 1514
      v
Wazuh Manager
10.10.10.20
      |
      v
Wazuh Indexer
      |
      v
Wazuh Dashboard
```

This flow is used for events including:

- authentication activity;
- sudo activity;
- system inventory;
- vulnerability information;
- File Integrity Monitoring events.

## Suricata IDS Flow

Suricata is installed directly on pfSense and monitors the LAN interface.

The tested detection path is:

```text
Ubuntu Docker Host
10.10.10.10
      |
      | Network traffic
      v
pfSense / Suricata
10.10.10.1
      |
      | EVE JSON
      v
syslog-ng
      |
      | TCP 5514
      v
Wazuh Server
10.10.10.20
      |
      v
rsyslog
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

TCP port `5514` is dedicated to the Suricata EVE JSON transport in this V1.

## IDS Validation Flow

The V1 detection path was validated with an Nmap scan generated from the Ubuntu Docker host:

```bash
sudo nmap -sS -sV -O -T4 10.10.10.1
```

The validated chain is:

```text
Nmap scan
   |
   v
Suricata detection
   |
   v
EVE JSON alert
   |
   v
syslog-ng transport
   |
   v
TCP 5514 reception
   |
   v
rsyslog file output
   |
   v
Wazuh JSON ingestion
   |
   v
Native Suricata rule
   |
   v
Threat Hunting event
```

The observed alert was:

```text
ET SCAN Possible Nmap User-Agent Observed
Signature ID: 2024364
Wazuh Rule ID: 86601
```

## File Integrity Monitoring

Realtime FIM validation is performed on the Ubuntu Docker host.

Test path:

```text
/opt/wazuh-fim-test
```

Flow:

```text
File creation / modification / deletion
                |
                v
          Wazuh Agent
                |
                v
          Wazuh Manager
                |
                v
 File Integrity Monitoring
```

The V1 successfully detected:

- file creation;
- file modification;
- file deletion.

## V1 Security Boundaries

The current V1 has a simple flat LAB-LAN:

```text
10.10.10.0/24
```

No VLAN segmentation is implemented yet.

This is intentional for the first version so that the basic routing, monitoring and detection pipeline can be validated before adding network complexity.

VLAN segmentation is planned as a future improvement.

## V1 Data Flows Summary

| Source | Destination | Protocol / Port | Purpose |
|---|---|---|---|
| Ubuntu Docker Host | Wazuh Server | `TCP 1514` | Wazuh Agent events |
| pfSense / Suricata | Wazuh Server | `TCP 5514` | EVE JSON transport |
| Admin workstation | Ubuntu Docker Host | `TCP 22` | SSH administration |
| LAB-LAN clients | Ubuntu Docker Host | `TCP 8080` | Nginx |
| LAB-LAN clients | Ubuntu Docker Host | `TCP 9443` | Portainer |

## Planned Architecture Evolution

Future versions may add:

- VLAN segmentation;
- Windows Server;
- Active Directory;
- Windows endpoint monitoring;
- Sysmon;
- Kali Linux;
- vulnerable machines;
- dedicated attacker and victim networks;
- additional Wazuh agents;
- custom detections and Active Response.
