# Hybrid Cloud Site-to-Site IPsec VPN (Ubuntu to Azure)

## Objective
This project demonstrates the design and implementation of a hybrid cloud architecture by establishing a secure Site-to-Site IPsec VPN tunnel between an on-premises Ubuntu server and a Microsoft Azure Virtual Network. It highlights the configuration of IKEv2 encryption, Azure virtual network gateway deployment, Linux packet forwarding, and troubleshooting of IPsec negotiation failures.

## Skills & Technologies Showcased
- Cloud Infrastructure: Microsoft Azure (Virtual Networks, Virtual Network Gateways, Azure VMs)
- Linux/OS Configuration: Ubuntu Server, package management, IPv4 packet forwarding
- Network Security: IPsec/IKEv2, strongSwan, Pre-Shared Key (PSK), AES256/SHA256 encryption
- Problem Determination: Resolving IKEv2 proposal mismatches and parsing configuration errors via system logs

## Network Topology
- On-Premises Environment: Ubuntu Linux VM
  - Local Subnet: 192.168.1.0/24
- Cloud Environment: Azure Virtual Network
  - Cloud Subnet: 10.0.0.0/24
- Gateway IP: Azure Public IP
  - 20.237.100.190

## Deployment Phases

### Phase 1: Azure Infrastructure Setup
- Provisioned an Azure Virtual Network (VNet) for the cloud environment.
- Created a Gateway Subnet and deployed a Route-based Virtual Network Gateway to act as the cloud endpoint for the IPsec tunnel.
- Deployed a test Ubuntu VM on the default Azure subnet (10.0.0.4) to validate end-to-end ICMP connectivity.

### Phase 2: On-Premises Server Configuration (Ubuntu)
Enabled IP forwarding on the Linux server by editing `/etc/sysctl.conf`:

```bash
net.ipv4.ip_forward=1
```

Reloaded the configuration and installed strongSwan:

```bash
sudo sysctl -p
sudo apt update
sudo apt install strongswan strongswan-starter
```

### Phase 3: IPsec Configuration & Problem Determination
Configured the strongSwan initiator for Azure. During the initial deployment, the connection failed with a `NO_PROPOSAL_CHOSEN` error because Azure’s strict security requirements rejected the default strongSwan encryption algorithms.

To resolve this, the `ike` and `esp` parameters were manually defined to enforce AES256 and SHA256, successfully negotiating the Phase 1 and Phase 2 security associations.

#### `/etc/ipsec.conf`

```conf
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn azure-vpn
    type=tunnel
    keyexchange=ikev2
    authby=secret
    left=%defaultroute
    leftsubnet=192.168.1.0/24
    right=20.237.100.190
    rightsubnet=10.0.0.0/24
    auto=start
    ike=aes256-sha256-modp1024,aes256-sha1-modp1024!
    esp=aes256-sha256,aes256-sha1!
```

#### `/etc/ipsec.secrets`

```conf
# Source IP      Destination IP    Auth Type    Pre-Shared Key
192.168.1.151    20.237.100.190  : PSK          "YOUR_SECRET_KEY"
```

### Phase 4: Tunnel Initiation and Verification
Restarted strongSwan and brought up the VPN connection:

```bash
sudo ipsec restart
sudo ipsec up azure-vpn
```

Validation confirmed that the IKE_SA and CHILD_SA were established successfully. To prove routing and payload encryption were working, a ping test was executed from the local Ubuntu machine to the private IP of the Azure test VM.

```bash
$ ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=59.8 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=34.6 ms
...
--- 10.0.0.4 ping statistics ---
26 packets transmitted, 26 received, 0% packet loss, time 25053ms
```

## Result
This configuration successfully established an encrypted, bidirectional IPsec tunnel between the on-premises Ubuntu environment and Azure, enabling secure communication between 192.168.1.0/24 and 10.0.0.0/24. The 0% packet loss confirms that traffic was actively flowing across the hybrid cloud boundary.

## Notes
- This project demonstrates real-world networking troubleshooting and secure connectivity design.
- The Azure gateway and pre-shared key values shown here are examples and should be replaced with environment-specific values.
- StrongSwan and Azure both enforce strict crypto policy, so matching Phase 1 and Phase 2 proposal parameters is critical.

## Repository Purpose
This repository serves as a hands-on example of configuring a Microsoft Azure Site-to-Site VPN to a Linux-based on-premises environment using strongSwan and IKEv2.
