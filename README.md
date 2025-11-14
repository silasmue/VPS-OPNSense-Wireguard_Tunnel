# VPS-OPNSense-Wireguard_Tunnel
Expose a VPS as public IP for parts of your network using OPNSense to manage the tunnel and the interface.
This is necessary for bypassing CGNAT (Homelabsetups), or if a second public available IP is required.
```
                       +---------------------------+ public IP eth0: 200.A.B.C
                       |            VPS           |
                       |        (WireGuard)       |
                       +---------------------------+ wg0: 10.100.0.1/30
                                    ||
                           (WireGuard Tunnel)
                                    ||
                       +---------------------------+ wg0: 10.100.0.2/30
                       |          OPNsense        |
                       |   WAN1   +   WG_0_GW     |
                       +---------------------------+ eth0: both VLANS described below
                             /                 \
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
