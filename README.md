# Linux Router Lab: DHCP, DNS, NAT and Firewall on Ubuntu Server

## Project Overview

This lab turns an **Ubuntu Server 26.04 LTS** virtual machine into a small enterprise-style router that connects two isolated networks (**Admin** and **Users**) and provides **DHCP**, **DNS**, **NAT** and **firewall filtering**. Two client VMs are used to validate each service and the traffic rules.

## Technologies Used

- Ubuntu Server 26.04 LTS
- VMware Workstation (Host-only and NAT virtual networks)
- Kea DHCP 4
- bind9 (DNS)
- nftables (NAT and firewall)
- Netplan (static addressing)
- SSH (PowerShell)

## Objectives

- Build a Linux router with three network interfaces (one WAN, two LANs).
- Assign addresses automatically with a DHCP server for each LAN.
- Resolve internal names (`lab.lan`) and external names with a DNS server.
- Give both LANs Internet access through NAT.
- Block traffic from **Users** to **Admin** while allowing **Admin** to reach **Users**, and log every blocked packet.

## Network Architecture

```mermaid
flowchart TB
    I["Internet (VMware NAT)"] --> R
    R["router-srv<br/>Router + DHCP + DNS + Firewall"]
    R -->|"ens34 - 192.168.10.254"| A["Admin network<br/>192.168.10.0/24"]
    R -->|"ens35 - 192.168.20.254"| U["Users network<br/>192.168.20.0/24"]
    A --> CA["client-admin"]
    U --> CU["client-users"]
```

| Element | Value |
|---|---|
| WAN interface | `ens33` (VMware NAT, DHCP) |
| Admin network | `ens34`, VMnet2 (Host-only), `192.168.10.0/24`, gateway `192.168.10.254` |
| Users network | `ens35`, VMnet3 (Host-only), `192.168.20.0/24`, gateway `192.168.20.254` |
| DHCP pools | `.100` to `.200` in each network |
| Domain | `lab.lan` |
| VMware DHCP | Disabled on VMnet2 and VMnet3 (Kea is the only DHCP server) |

## Implementation

### 1. Virtual networks and router VM

Two Host-only networks were created with the VMware DHCP service disabled. The router VM has three network cards: NAT, VMnet2 and VMnet3.

![Virtual Network Editor](01_virtual_network_editor.png)

![Router hardware](02_router_hardware.png)

During the installation, `ens33` receives an address from VMware while `ens34` and `ens35` stay unconfigured, as expected.

![Installer network screen](03_installer_network.png)

![ip a on the router](04_ip_a_router.png)

![SSH to the router](05_ssh_router.png)

### 2. Static addressing with Netplan

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens34:
      dhcp4: false
      addresses:
        - 192.168.10.254/24
    ens35:
      dhcp4: false
      addresses:
        - 192.168.20.254/24
```

![Netplan configuration](06_netplan.png)

![Static addresses](07_ip_statiques.png)

Ping tests from Windows confirmed which interface belongs to which virtual network.

![Ping from Windows](08_ping_windows.png)

### 3. Routing and NAT

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl --system
```

![IP forwarding](09_ip_forward.png)

NAT rule (masquerade on the WAN interface):

```
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "ens33" masquerade
    }
}
```

![NAT rule](10_nftables_nat.png)

### 4. DHCP with Kea

Kea listens only on the two LAN interfaces and serves one pool per network, with the router as gateway and DNS server.

```json
"subnet4": [
  {
    "id": 1,
    "subnet": "192.168.10.0/24",
    "pools": [ { "pool": "192.168.10.100 - 192.168.10.200" } ],
    "option-data": [
      { "name": "routers", "data": "192.168.10.254" },
      { "name": "domain-name-servers", "data": "192.168.10.254" },
      { "name": "domain-name", "data": "lab.lan" }
    ]
  }
]
```

(The second subnet, `192.168.20.0/24`, follows the same pattern.)

The service is active, the logs report `added IPv4 subnets: 2`, and Kea listens on port 67 on both LAN interfaces.

![Kea running and listening on port 67](12_kea_actif.png)

### 5. DNS with bind9

bind9 answers only the internal networks, hosts the `lab.lan` zone with forward and reverse records, and forwards other queries to public resolvers.

```
options {
    listen-on { 127.0.0.1; 192.168.10.254; 192.168.20.254; };
    allow-query { 127.0.0.1; 192.168.10.0/24; 192.168.20.0/24; };
    recursion yes;
    allow-recursion { 127.0.0.1; 192.168.10.0/24; 192.168.20.0/24; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
};
```

![bind9 configuration](13_bind9_conf.png)

![Zone lab.lan](14_zone_lab_lan.png)

![Syntax checks](15_named_checks.png)

![DNS tests with dig](16_tests_dns.png)

### 6. Client VMs

Each client was installed on its own Host-only network and received its address from Kea.

![client-admin hardware](17_client_admin_hardware.png)

![client-admin tests](19_client_admin_tests.png)

![Kea leases](20_kea_leases.png)

![client-users hardware](21_client_users_hardware.png)

![client-users tests](22_client_users_tests.png)

### 7. Routing between the two networks (before the firewall)

Without filtering, a client in Users reaches Admin through the router. `traceroute` shows the router as the first hop.

![Routing before the firewall](24_routage_avant_pare_feu.png)

## Firewall Policy

| Source | Destination | Action |
|---|---|---|
| Admin | Users | Allow |
| Admin and Users | Internet | Allow |
| **Users** | **Admin** | **Drop and log** |
| Admin and Users | Router DNS (53) and DHCP (67) | Allow |
| Windows host and Admin | Router SSH (22) | Allow |
| Users | Router SSH (22) | Drop (default) |

Both chains use a **default drop** policy, so only explicitly allowed traffic passes.

```
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iif lo accept
        ct state established,related accept
        ct state invalid drop
        ip protocol icmp accept
        iifname { "ens34", "ens35" } udp dport { 53, 67 } accept
        iifname { "ens34", "ens35" } tcp dport 53 accept
        iifname "ens33" tcp dport 22 accept
        iifname "ens34" tcp dport 22 accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        ct state invalid drop
        iifname "ens34" oifname "ens35" accept
        iifname { "ens34", "ens35" } oifname "ens33" accept
        iifname "ens35" oifname "ens34" counter log prefix "BLOCK Users->Admin " drop
    }
}
```

![Firewall rules](25_nftables_firewall%20%282%29.png)

## Results

- From **Users** to **Admin**: 100% packet loss (blocked), while Internet access and DNS still work.
- From **Admin** to **Users**: replies received (allowed).
- The rule counter reached 3 packets (252 bytes), and each blocked packet was written to the kernel log with its source and destination.

![Users to Admin blocked](26_users_vers_admin_bloque.png)

![Admin to Users allowed](28_admin_vers_users_ok%202.png)

![Firewall log and counter](29_log_blocage.png)

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| Kea service `active` but nothing listening on port 67 | The config file was still the package sample (subnet `192.0.2.0/24`), because the configuration paste was incomplete | Rewrote `/etc/kea/kea-dhcp4.conf` and checked `added IPv4 subnets: 2` in the logs |
| `Permission denied` when reading the Netplan file | The file is `chmod 600` by design | Read it with `sudo cat` |
| Interfaces `ens34` and `ens35` without address in the installer | VMware DHCP disabled on VMnet2 and VMnet3 | Expected behavior, static addresses set with Netplan |
| Login refused on the first boot | The login uses the username, not the full name typed in the installer | Logged in with the username |
| Windows already uses `.1` on each Host-only network | VMware assigns `.1` to the host adapter | Router set to `.254` |
| `Destination Host Unreachable` when testing Admin to Users | The client address had changed (DHCP lease went from `.100` to `.102`) | Used the current address; firewall rules rely on interfaces, not addresses |
| Slow Wi-Fi caused `HTTP 400` while uploading screenshots | Long upload time on the web page | Uploaded a few files at a time |

## Skills Demonstrated

- IP addressing and subnetting
- Linux routing and NAT
- DHCP (Kea) and DNS (bind9) administration
- Firewall design with a default-deny policy (nftables) and logging
- Network segmentation and traffic validation (ping, traceroute, dig)
- Virtual network design in VMware
- Troubleshooting and technical documentation

## Next Steps

- DHCP reservations bound to MAC addresses.
- Capture and analyze the DHCP exchange (Discover, Offer, Request, Ack) with Wireshark.
- Add a VPN (WireGuard) on the router.

## Author

**Ismail Bahia**

IT Infrastructure | Systems & Networks

GitHub: https://github.com/bahiaismail9-ai
