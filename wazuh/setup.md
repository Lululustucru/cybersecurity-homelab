# Wazuh Setup

## Objective

Deploy a centralized SIEM/XDR platform to monitor the hosts inside the Home Lab.

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

## Lessons Learned

- Always verify actual filesystem size after installing Ubuntu with LVM.
- A virtual disk size does not necessarily equal the root filesystem size.
- Check `df`, `lsblk`, `vgs` and `lvs` when troubleshooting storage.
- Failed package installations can leave inconsistent package states.
- Service status alone does not guarantee correct network configuration.
- Verify the manager address and network connectivity when deploying agents.
