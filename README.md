# Secure Remote Access VPN using WireGuard
## Introduction

This project demonstrates the implementation of a secure remote access mechanism using WireGuard VPN. The objective is to enable a remote client to access an isolated internal network through an encrypted tunnel. The setup reflects a typical enterprise scenario where internal systems are protected behind a gateway and are not directly exposed to external networks.

## Components Used

The lab environment consists of three virtual machines:

Ubuntu Server (VPN Server and Gateway)
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

Internal Network (Host-Only – VMnet1)
This network is isolated and not accessible externally.

Range: 172.16.1.0/24
Used by:
VPN Server (interface: ens37 → 172.16.1.1)
Internal Ubuntu Server (172.16.1.10)

VPN Tunnel Network
This is the virtual network created by WireGuard.

Range: 10.10.0.0/24
Used by:
VPN Server (10.10.0.1)
Kali Client (10.10.0.2)
## System Architecture

The VPN server is configured with two network interfaces:

ens33 (Bridged): connected to external network (10.229.64.x)
ens37 (Host-only): connected to internal network (172.16.1.0/24)

The server acts as a gateway between:

External client network
VPN tunnel network
Internal private network

The internal Ubuntu machine is only connected to the host-only network and has no direct route to the external network.

WireGuard Configuration
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

##Packet Flow

When the VPN is inactive:

Kali attempts to access 172.16.1.10
No route exists
Communication fails

When the VPN is active:

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

Test connectivity:

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
