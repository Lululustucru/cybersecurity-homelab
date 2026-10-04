# IP Plan

| Device | Role | IP |
|---|---|---|
| pfSense | LAN Gateway | 10.10.10.1 |
| Ubuntu Server | Docker Host | 10.10.10.10 |
| DHCP Pool | Dynamic Clients | 10.10.10.100-200 |
| Wazuh Server | SIEM / XDR | 10.10.10.20 |

## Networks

### Home network
192.168.1.0/24

### LAB-LAN
10.10.10.0/24

Gateway:
10.10.10.1

DHCP pool:
10.10.10.100 - 10.10.10.200
