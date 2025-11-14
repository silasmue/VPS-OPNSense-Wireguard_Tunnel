# VPS-OPNSense-Wireguard_Tunnel
Expose a VPS as public IP for parts of your network using OPNSense to manage the tunnel and the interface.
This is necessary for bypassing CGNAT (Homelabsetups), or if a second public available IP is required.
```
                       +---------------------------+ public IP eth1: 200.A.B.C
                       |            VPS           |
                       |        (WireGuard)       |
                       +---------------------------+ wg0: 10.100.0.1/30
                                    ||
                           (WireGuard Tunnel)
                                    ||
 public interface      +---------------------------+ wg0: 10.100.0.2/30
 ens3: 100.65.1.2      |          OPNSense        |
 IP in CGNAT           |   WAN1   +   WG_0_GW     |
 not reachable         +---------------------------+ eth0: both VLANS described below
 from outside                /                 \
                            /                   \
                           v                     v

                 VLAN1: 10.0.1.0/24      VLAN2: 10.0.2.0/24
                         |                           |
                         v                           v
                   +------------+              +------------+ NGINX is here:
                   |  Client    |              |  Client    | IP 10.0.2.2/24
                   |10.0.1.2/24 |              |10.0.2.1/24 |
                   +------------+              +------------+
```
*Image generated with ChatGPT5.1 Thinking*
The setup above describes the scenario. To clarify OPNSense has one physical interface that is connected to the ISP, here `ens3`, the second physical interface, `eth0` is split into two by VLAN using two VLANS with tag 1 and 2. The two VLANs are not relevant for this guide, but the reverse proxy that should be reached later is in one of the subnets with the IP: `10.0.2.2/24` (VLAN2). The VPS has only one physical interface, for everything, that interface `eth1` has a public IP. The clients in the two private subnets are connected via VLAN-aware switch with one port configured as TRUNK.

**Goal**: The goal is to route all traffic over the *normal* gateway in OPNSense, except for traffic that reaches the public IP of the VPS, that traffic is routed to OPNSense through the wireguard tunnel, there it is redirected by NAT and port-forwarding to the NGINX-server. Only the replys from the NGINX-server should be routed through the wiregurad gateway (tunnel interface) and not the physical interface. When finished it looks for the public like the server in the private network, NGINX-server has a public IP. This guide does not want to bypass the OPNSense firewall by tunneling directly to the NGINX-server, this behaivoir is described in another guide.

## VPS setup
First `wireguard` and `iptables` must be installed. After that `wireguard` can be configured. First  `publickey` and `presharedkey` need to be created.
```
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | tee /etc/wireguard/publickey
wg genpsk > /etc/wireguard/presharedkey
```
After that, the `/etc/wireguard/wg0.conf` can be edited as follows:
```
[Interface]
Address    = 10.99.0.1/30
ListenPort = 55107
PrivateKey = <VPS_PRIVATE_KEY>
MTU        = 1420

[Peer]
PublicKey           = <OPNSENSE_PUBLIC_KEY> # Fill in after the next step
PresharedKey        = <PRESHARED_KEY>
AllowedIPs          = 10.99.0.2/32, 10.0.20.2/32
PersistentKeepalive = 25
```
*Be aware the `privatekey` is already in the file and must be preserved*

When finished:
```
systemctl enable --now wg-quick@wg0
```
*Run `systemctl restart wg-quick@wg0` after finishing the configuration, when inserting the `<OPNSENSE_PUBLIC_KEY>`*

Now the `wireguard` configuration is finished, the routing needs to be configured:
```
# Clear existing routed
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X

iptables -P INPUT   ACCEPT
iptables -P OUTPUT  ACCEPT
iptables -P FORWARD DROP

sysctl -w net.ipv4.ip_forward=1

iptables -A FORWARD -i ens6 -o wg0 \
  -m conntrack --ctstate NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wg0 -o ens6 \
  -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# NAT: keep SSH and WG on the VPS, this important to don't lock yourself out and kill the tunnel
iptables -t nat -A PREROUTING -i ens6 -p tcp --dport 22    -j ACCEPT
iptables -t nat -A PREROUTING -i ens6 -p udp --dport 55107 -j ACCEPT

iptables -t nat -A PREROUTING -i ens6 -p tcp --dport 80 \
  -j DNAT --to-destination 10.99.0.2

iptables -t nat -A POSTROUTING -s 10.99.0.0/30 -o ens6 -j MASQUERADE
```
The config can be made persistend with:
```
iptables-save | sudo tee /etc/iptables/rules.v4
```

## OPNSense Wireguard Setup
