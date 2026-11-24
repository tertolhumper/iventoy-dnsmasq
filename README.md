# PXE Boot Using dnsmasq and iVentoy

## Network Setup

A laptop connected through its built-in Wi-Fi serves as the PXE server. It provides iVentoy boot services to devices on the local LAN through the `wlp1s0` interface, using IP address `192.168.0.120`.

The router handles client IP assignment, dnsmasq provides PXE boot information, and iVentoy serves the selected ISO images.

| Parameter | Value |
|---|---|
| PXE server IP | `192.168.0.120` |
| Network interface | `wlp1s0` |
| Network | `192.168.0.0/24` |
| Gateway | `192.168.0.1` |
| DHCP mode | External |
| iVentoy directory | `~/Downloads/iventoy` |
| ISO directory | `~/Downloads/iventoy/iso/` |
| WebUI | `http://192.168.0.120:26000` |

## 1. iVentoy Installation

Download iVentoy from its official source and extract the archive.

Installation directory:

```bash
~/Downloads/iventoy
```

Startup script:

```bash
~/Downloads/iventoy/iventoy.sh
```

## 2. External DHCP Configuration

Select **External DHCP** in the iVentoy management interface.

Under Boot Information, configure:

| Parameter | Value |
|---|---|
| DHCP Mode | External |
| Server IP | `192.168.0.120` |
| NIC Name | `wlp1s0` |

The existing router provides IP addresses to clients. dnsmasq handles Proxy DHCP and supplies PXE boot information.

Internal DHCP is a separate operating mode in which iVentoy handles DHCP itself.

## 3. dnsmasq Configuration

Create the configuration file:

```bash
sudo nano /etc/dnsmasq.d/iventoy.conf
```

Configuration:

```ini
port=0
log-dhcp
interface=wlp1s0
bind-interfaces
dhcp-range=192.168.0.0,proxy

pxe-service=x86PC,"iVentoy BIOS",iventoy_loader_16000,192.168.0.120
pxe-service=x86-64_EFI,"iVentoy UEFI",iventoy_loader_16000,192.168.0.120
pxe-service=BC_EFI,"iVentoy UEFI BC",iventoy_loader_16000,192.168.0.120
```

### Directive explanation

| Directive | Purpose |
|---|---|
| `port=0` | Disables the DNS service. dnsmasq is used for DHCP and PXE only. |
| `log-dhcp` | Enables DHCP logging, including PXE-related requests. |
| `interface=wlp1s0` | Restricts dnsmasq to the specified network interface. |
| `bind-interfaces` | Binds dnsmasq directly to the specified interface. |
| `dhcp-range=192.168.0.0,proxy` | Enables Proxy DHCP without assigning client IP addresses. The router continues handling normal DHCP. |

### PXE service declarations

The `pxe-service` directive advertises boot options according to the architecture reported by the PXE client.

General syntax:

```text
pxe-service=<architecture>,"<description>",<boot-file>,<server-address>
```

**Legacy BIOS**

```ini
pxe-service=x86PC,"iVentoy BIOS",iventoy_loader_16000,192.168.0.120
```

- `x86PC`: Legacy x86 BIOS architecture.
- `iVentoy BIOS`: PXE menu description.
- `iventoy_loader_16000`: iVentoy bootloader identifier.
- `192.168.0.120`: iVentoy server address.

**UEFI x86-64**

```ini
pxe-service=x86-64_EFI,"iVentoy UEFI",iventoy_loader_16000,192.168.0.120
```

- `x86-64_EFI`: UEFI x86-64 architecture.
- `iVentoy UEFI`: PXE menu description.
- `iventoy_loader_16000`: iVentoy bootloader identifier.
- `192.168.0.120`: iVentoy server address.

**UEFI Byte Code**

```ini
pxe-service=BC_EFI,"iVentoy UEFI BC",iventoy_loader_16000,192.168.0.120
```

- `BC_EFI`: EFI Byte Code architecture.
- `iVentoy UEFI BC`: PXE menu description.
- `iventoy_loader_16000`: iVentoy bootloader identifier.
- `192.168.0.120`: iVentoy server address.

These entries allow dnsmasq to advertise the corresponding iVentoy boot option to BIOS and UEFI clients.

### Apply configuration

Restart dnsmasq:

```bash
sudo systemctl restart dnsmasq
```

Check service status:

```bash
systemctl status dnsmasq
```

Monitor DHCP and PXE activity:

```bash
journalctl -u dnsmasq -f
```

## 4. Firewall Rules

Check the active firewall configuration before testing PXE boot.

For UFW:

```bash
sudo ufw status verbose
```

For nftables:

```bash
sudo nft list ruleset
```

Review the active firewall rules and ensure PXE traffic is permitted on the `wlp1s0` network interface. Check the ports used by dnsmasq and iVentoy before making any firewall changes.

## 5. UEFI Bootloader

The selected UEFI boot file is:

```text
snponly.efi
```

`snponly.efi` relies on the motherboard's UEFI Simple Network Protocol (SNP) for network connectivity instead of using its own native network driver stack.

This leaves USB controller and keyboard handling under UEFI firmware control.

The tested `ipxe.efi` build included native USB network drivers. In this setup, initializing those drivers could take control of the USB controller from the motherboard firmware and cause the keyboard to stop responding.

Switching to `snponly.efi` resolved the keyboard input issue.

## 6. systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/iventoy.service
```

Service configuration:

```ini
[Unit]
Description=iVentoy PXE Server
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
WorkingDirectory=/home/tertol/Downloads/iventoy
ExecStart=/home/tertol/Downloads/iventoy/iventoy.sh start
ExecStop=/home/tertol/Downloads/iventoy/iventoy.sh stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable iventoy
sudo systemctl start iventoy
```

Check status:

```bash
systemctl status iventoy
```

## 7. ISO Management

ISO images are stored in the iVentoy installation directory:

```bash
~/Downloads/iventoy/iso/
```

Current ISO inventory:

| ISO image | Purpose |
|---|---|
| `debian-13.7.0-amd64-netinst.iso` | Debian network installation |
| `memtest.iso` | Memory testing |
| `systemrescue-13.02-amd64.iso` | System recovery and maintenance |

List available images:

```bash
ls -lh ~/Downloads/iventoy/iso/
```

Add another ISO:

```bash
cp ~/Downloads/image.iso ~/Downloads/iventoy/iso/
```

Remove an ISO:

```bash
rm ~/Downloads/iventoy/iso/image.iso
```

iVentoy uses this directory to populate the PXE boot menu.

## 8. Web Management

Access the iVentoy WebUI:

```text
http://192.168.0.120:26000
```

Use the interface to manage the iVentoy server and available boot images.

## 9. PXE Boot Sequence

1. Client starts BIOS or UEFI network boot.
2. The router assigns the client an IP address.
3. dnsmasq receives the PXE request.
4. dnsmasq advertises the appropriate boot service based on the client architecture.
5. The client contacts the iVentoy server at `192.168.0.120`.
6. iVentoy loads its boot environment.
7. Available ISO images appear in the boot menu.
8. The client selects the required installation or recovery image.

## 10. Configuration Summary

| Component | Configuration |
|---|---|
| PXE server | iVentoy |
| PXE proxy | dnsmasq |
| Server IP | `192.168.0.120` |
| Network interface | `wlp1s0` |
| DHCP mode | External |
| Normal DHCP | Existing router |
| Proxy DHCP | dnsmasq |
| BIOS architecture | `x86PC` |
| UEFI architecture | `x86-64_EFI` |
| EFI BC architecture | `BC_EFI` |
| UEFI bootloader | `snponly.efi` |
| iVentoy bootloader identifier | `iventoy_loader_16000` |
| Service management | systemd |
| WebUI | TCP 26000 |
| ISO directory | `~/Downloads/iventoy/iso/` |
