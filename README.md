# Cybersecurity Home Lab

A personal cybersecurity home lab designed to build practical skills in networking, virtualization, Linux administration, containerization, monitoring and defensive security.

The environment is built progressively and documented step by step so that each component can be understood, reproduced and improved.

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
                       |
                    LAB-LAN
                       |
               Ubuntu Server
                 10.10.10.10
                       |
                    Docker
                  /          \
            Nginx :8080   Portainer :9443
```

## Technologies

- Hyper-V
- pfSense CE
- Ubuntu Server
- Docker
- Docker Compose
- Nginx
- Portainer
- SSH
- Git
- GitHub

Planned components:

- Wazuh
- Suricata
- Windows Server
- Active Directory
- Kali Linux
- Vulnerable machines
- Centralized logging
- Network segmentation

## Network Plan

| Device | Role | Address |
|---|---|---|
| pfSense | LAN Gateway | 10.10.10.1 |
| Ubuntu Server | Docker Host | 10.10.10.10 |
| DHCP Pool | Dynamic Clients | 10.10.10.100-200 |

## Implemented

- [x] Hyper-V virtualization
- [x] Dedicated LAB-LAN virtual network
- [x] Dedicated external WAN switch
- [x] pfSense firewall/router
- [x] LAN/WAN separation
- [x] DHCP
- [x] DHCP static mapping
- [x] Ubuntu Server
- [x] SSH administration
- [x] Docker
- [x] Docker Compose
- [x] Nginx
- [x] Portainer
- [x] GitHub documentation

## Project Goals

The objective of this lab is to gain practical experience with:

- Network architecture
- Routing and firewalling
- Linux system administration
- Containerization
- Infrastructure documentation
- SIEM
- IDS/IPS
- Log analysis
- Active Directory security
- Offensive security
- Detection engineering

## Repository Structure

```text
architecture/
docker/
hyperv/
pfsense/
ubuntu/
troubleshooting/
screenshots/
```

## Troubleshooting

Problems encountered during deployment are documented rather than hidden.

Examples include:

- pfSense repository access failure
- WAN connectivity through Hyper-V
- WAN and LAN initially sharing the same subnet
- DHCP and static mappings
- Ubuntu networking
