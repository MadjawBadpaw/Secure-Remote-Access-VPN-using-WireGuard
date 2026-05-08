# Secure Remote Access VPN using WireGuard

## Introduction

This project demonstrates a secure remote access implementation using WireGuard VPN. A remote client connects to an isolated internal network through an encrypted tunnel — reflecting a real enterprise scenario where internal systems are protected behind a gateway and never directly exposed to external networks.

---

## Lab Environment

| Machine | OS | Role |
|---|---|---|
| Wazuh Server | Ubuntu Server | VPN Server + Gateway |
| Internal Machine | Ubuntu Server | Internal Target |
| Remote Client | Kali Linux | External Client |

**Virtualization Platform:** VMware Workstation

---

## Network Architecture

```
[ Kali Linux ]────────────────────────────────────────────────────[ VPN Server ]────────[ Internal Ubuntu ]
  10.10.0.2                   WireGuard Tunnel                      10.10.0.1              172.16.1.10
  10.229.64.x                 (Encrypted)                     ens33: 10.229.64.x
                                                              ens37: 172.16.1.1
         External Network (Bridged)                               Internal Network (Host-Only VMnet1)
            10.229.64.0/24                                              172.16.1.0/24
```

### Network Segments

| Network | Range | Interfaces | Used By |
|---|---|---|---|
| External (Bridged) | 10.229.64.0/24 | ens33 | Kali, VPN Server |
| Internal (Host-Only) | 172.16.1.0/24 | ens37 | VPN Server, Internal Ubuntu |
| VPN Tunnel | 10.10.0.0/24 | wg0 | VPN Server, Kali Client |

---

## Configuration Files

### Netplan — VPN Server
`/etc/netplan/*.yaml`
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true                    # External — DHCP
    ens37:
      addresses:
        - 172.16.1.1/24              # Internal — Static
```

### Netplan — Internal Ubuntu
`/etc/netplan/*.yaml`
```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.1.10/24
      routes:
        - to: default
          via: 172.16.1.1            # Gateway = VPN Server
      nameservers:
        addresses: [8.8.8.8]
```

### WireGuard — VPN Server
`/etc/wireguard/wg0.conf`
```ini
[Interface]
Address    = 10.10.0.1/24
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = 51820

# Enable forwarding and NAT when tunnel comes up
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp   = iptables -A FORWARD -o wg0 -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE

# Cleanup when tunnel goes down
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ens33 -j MASQUERADE

[Peer]
PublicKey  = <KALI_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32
```

### WireGuard — Kali Client
`/etc/wireguard/wg0.conf`
```ini
[Interface]
Address    = 10.10.0.2/24
PrivateKey = <KALI_PRIVATE_KEY>

[Peer]
PublicKey           = <SERVER_PUBLIC_KEY>
Endpoint            = 10.229.64.x:51820
AllowedIPs          = 172.16.1.0/24        # Route internal network through VPN
PersistentKeepalive = 25
```

### IP Forwarding — VPN Server
`/etc/sysctl.conf`
```bash
net.ipv4.ip_forward=1
```
Apply with:
```bash
sudo sysctl -p
```

---

## Key Generation

WireGuard uses asymmetric key pairs. Each machine generates its own private/public key pair.

### On VPN Server
```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key

cat server_private.key    # goes into server wg0.conf → PrivateKey
cat server_public.key     # shared with Kali client → [Peer] PublicKey
```

### On Kali Client
```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key

cat client_private.key    # goes into client wg0.conf → PrivateKey
cat client_public.key     # shared with server → [Peer] PublicKey
```

### Key Usage Summary

| Machine | Uses | Shares |
|---|---|---|
| VPN Server | Own private key + Kali's public key | Its public key with Kali |
| Kali Client | Own private key + Server's public key | Its public key with Server |

---

## Packet Flow

### VPN Inactive
```
Kali → ping 172.16.1.10 → No route exists → FAIL
```

### VPN Active
```
Kali (10.10.0.2)
  │  Traffic destined for 172.16.1.10
  │  Routing table directs to wg0
  ▼
Packet encrypted → sent to VPN Server (10.229.64.x:51820)
  │
  ▼
VPN Server decrypts → forwards via ens37
  │
  ▼
Internal Ubuntu (172.16.1.10) receives packet
  │
  ▼
Response routed back through VPN Server → encrypted → back to Kali
```

---

## Verification

### Check VPN Status
```bash
sudo wg
```

### Check Interfaces
```bash
ip a
```

### Check Routing Table
```bash
ip route
```

### Connectivity Tests
```bash
ping 172.16.1.10
ssh user@172.16.1.10
```

---

## Security Notes

```bash
# Restrict WireGuard config permissions
sudo chmod 600 /etc/wireguard/wg0.conf
```

- Never share private keys
- Replace all `<...>` placeholders with actual generated keys
- Verify interface names match your system (`ens33`, `ens37` may differ)

---

## Conclusion

This project successfully demonstrates secure remote access to an isolated internal network using WireGuard VPN. The VPN server acts as a dual-interface gateway — handling tunneling, routing, and NAT between the external client and the internal network. The internal machine remains completely unreachable without an active VPN connection, validating the isolation and access control objectives of the setup.
