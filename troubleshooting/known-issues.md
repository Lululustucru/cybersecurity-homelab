# Troubleshooting

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

## WAN and LAN on the same subnet

Initial configuration:

- WAN: 192.168.1.x/24
- LAN: 192.168.1.1/24

This was corrected by moving the LAN to:

- LAN: 10.10.10.1/24
