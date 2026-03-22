# Secure Remote Access VPN using WireGuard
## Introduction

This project demonstrates the implementation of a secure remote access mechanism using WireGuard VPN. The objective is to enable a remote client to access an isolated internal network through an encrypted tunnel. The setup reflects a typical enterprise scenario where internal systems are protected behind a gateway and are not directly exposed to external networks.

## Components Used

The lab environment consists of three virtual machines:

Ubuntu Server named Wazuh (VPN Server and Gateway)
Ubuntu Server (Internal Machine)
Kali Linux (Remote Client)

The virtualization platform used is VMware Workstation.

## Network Configuration

The setup uses three different network segments:

External Network (Bridged)
This network is connected to the host’s physical network.

Range: 10.229.64.0/24
Assigned via DHCP
Used by:
Kali Linux
VPN Server (interface: ens33)

### Internal Network (Host-Only – VMnet1)
This network is isolated and not accessible externally.

Range: 172.16.1.0/24
Used by:
VPN Server (interface: ens37 → 172.16.1.1)
Internal Ubuntu Server (172.16.1.10)

### VPN Tunnel Network
This is the virtual network created by WireGuard.

Range: 10.10.0.0/24
Used by:
VPN Server (10.10.0.1)
Kali Client (10.10.0.2)

## Configurations

This section includes the key configuration files used in the setup, covering network interfaces and WireGuard VPN configuration.

### Netplan Configuration (VPN Server)

File: /etc/netplan/*.yaml

network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      addresses:
        - 172.16.1.1/24

Explanation:

ens33 is the bridged interface connected to the external network (DHCP)
ens37 is the host-only interface for the internal network
### Netplan Configuration (Internal Ubuntu)

File: /etc/netplan/*.yaml

network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.1.10/24
      routes:
        - to: default
          via: 172.16.1.1
      nameservers:
        addresses: [8.8.8.8]

Explanation:

Static IP assigned in internal network
Default gateway points to VPN server (172.16.1.1)
### WireGuard Configuration (Server)

File: /etc/wireguard/wg0.conf

[Interface]
Address = 10.10.0.1/24
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = 51820

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ens33 -j MASQUERADE

[Peer]
PublicKey = <KALI_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32

Explanation:

Defines VPN interface and tunnel network
Enables forwarding between VPN and internal network
NAT ensures proper return path for packets
Peer section defines the client
### WireGuard Configuration (Kali Client)

File: /etc/wireguard/wg0.conf

[Interface]
PrivateKey = <KALI_PRIVATE_KEY>
Address = 10.10.0.2/24

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = 10.229.64.x:51820
AllowedIPs = 172.16.1.0/24
PersistentKeepalive = 25

Explanation:

Assigns client VPN IP
Defines server endpoint
Routes internal network traffic through VPN
### IP Forwarding (Server)

File: /etc/sysctl.conf

net.ipv4.ip_forward=1

Apply using:

sudo sysctl -p

Explanation:

Enables packet forwarding between interfaces
Required for routing traffic from VPN to internal network
#### Key Notes
Replace all placeholder keys (<...>) with actual generated keys
Ensure correct interface names (ens33, ens37) based on your system
Configuration permissions for WireGuard should be restricted:
sudo chmod 600 /etc/wireguard/wg0.conf

### WireGuard Key Generation

Keys are required to establish a secure connection between the server and client. Each system generates its own key pair consisting of a private key and a public key.

#### On VPN Server
wg genkey | tee server_private.key | wg pubkey > server_public.key

View keys:

cat server_private.key
cat server_public.key
server_private.key is used in the server configuration (PrivateKey)
server_public.key is shared with the client
#### On Kali Client
wg genkey | tee client_private.key | wg pubkey > client_public.key

View keys:

cat client_private.key
cat client_public.key
client_private.key is used in client configuration
client_public.key is added to server under [Peer]
#### Key Usage Summary
Server uses:
its own private key
client’s public key
Client uses:
its own private key
server’s public key

This exchange establishes trust and enables encrypted communication between both systems.
## System Architecture

The VPN server is configured with two network interfaces:

ens33 (Bridged): connected to external network (10.229.64.x)
ens37 (Host-only): connected to internal network (172.16.1.0/24)

The server acts as a gateway between:

External client network.
VPN tunnel network.
Internal private network.

The internal Ubuntu machine is only connected to the host-only network and has no direct route to the external network.

### WireGuard Configuration
Server Configuration

File: /etc/wireguard/wg0.conf

Interface address: 10.10.0.1/24
Listening port: 51820
NAT enabled using iptables
Packet forwarding enabled

### Peer configuration includes:

Kali public key
Allowed IP: 10.10.0.2/32
Client Configuration (Kali)

File: /etc/wireguard/wg0.conf

Interface address: 10.10.0.2/24
Endpoint: 10.229.64.x:51820 (VPN server bridged IP)
Allowed IPs: 172.16.1.0/24
Persistent keepalive enabled
Routing and Forwarding

IP forwarding is enabled on the VPN server to allow traffic to pass between interfaces.

The server uses iptables rules to:

Allow forwarding between VPN and internal interfaces
Perform NAT (MASQUERADE) on outgoing traffic

This ensures that internal systems can respond to VPN clients even though they are in different subnets.

## Packet Flow

### When the VPN is inactive:

Kali attempts to access 172.16.1.10 by ping or SSH.
No route exists.
Communication fails

### When the VPN is active:

Kali sends traffic to 172.16.1.10
Routing table directs traffic to wg0
Packet is encrypted and sent to VPN server
Server decrypts and forwards via ens37
Internal Ubuntu receives the packet
Response is routed back through the VPN server
Encrypted response reaches Kali
Verification

## Commands used for validation:

Check VPN status:

sudo wg

Check interface:

ip a

Check routing table:

ip route

### Tests:

ping 172.16.1.10
ssh user@172.16.1.10
Observations

The VPN creates a virtual interface named wg0 on both client and server.
Routing entries are dynamically added to direct traffic through the tunnel.
The internal network remains isolated and cannot be accessed without VPN.
The server successfully performs routing and NAT between different networks.

# Conclusion

The project successfully demonstrates secure remote access to an isolated internal network using WireGuard VPN. The implementation highlights key networking concepts such as tunneling, routing, and network address translation. The VPN server functions as a gateway that enforces controlled and secure communication between external and internal systems.

# Summary

A remote client is able to access an otherwise unreachable internal machine through an encrypted VPN tunnel, with proper routing and forwarding handled by a dual-interface Linux server.
