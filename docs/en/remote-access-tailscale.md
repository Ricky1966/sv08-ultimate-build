# Phoenix Remote Access with Tailscale

This page documents the Tailscale configuration validated on the Sovol SV08 **Phoenix** on August 25, 2026.

The goal is remote access to **SSH** and **Mainsail/Moonraker** without exposing router ports and without turning the CB1 into a subnet router or exit node.

## Validated baseline

Phoenix system during the test:

- BTT CB1;
- Debian 11 Bullseye, arm64;
- kernel `5.16.17-sun50iw9`;
- NetworkManager active;
- IP connectivity through `wlan0`;
- `systemd-resolved` disabled;
- `/etc/resolv.conf` managed directly by NetworkManager;
- TUN support available (`CONFIG_TUN=m`, `/dev/net/tun` present).

During validation Phoenix was connected to the Internet through an Android hotspot. This did not prevent Tailscale from working.

## Chosen architecture

Phoenix is configured as a normal Tailscale node.

The following are not used:

- subnet routing;
- exit node;
- advertised routes;
- Tailscale DNS management on the CB1.

This is intentionally conservative: the already-working local network must remain independent from Tailscale.

## Installation

The official Tailscale APT repository for Debian Bullseye was used, then the arm64 `tailscale` package was installed.

Validated version:

```text
1.102.3
```

The `tailscaled` service is installed and enabled through systemd.

## Startup and DNS

Because Phoenix uses NetworkManager with a traditional `/etc/resolv.conf` and `systemd-resolved` is inactive, Tailscale is started without taking over DNS management:

```bash
sudo tailscale up --accept-dns=false
```

After authentication, the validated preferences showed:

```text
RouteAll: false
CorpDNS: false
WantRunning: true
AdvertiseRoutes: null
```

The local default route and the DNS resolvers configured by NetworkManager remained unchanged.

## Moonraker and the Tailscale network

Tailscale assigns tailnet IPv4 addresses from `100.64.0.0/10`.

In the initial Phoenix configuration, Moonraker trusted only loopback and RFC1918 networks. As a result, Mainsail itself loaded from the web server, but its WebSocket connection to Moonraker failed with `HTTP 401 Unauthorized` when the client arrived through Tailscale.

The validated fix is to add the Tailscale network to `[authorization]` in `moonraker.conf`:

```ini
[authorization]
trusted_clients:
    127.0.0.0/8
    10.0.0.0/8
    172.16.0.0/12
    192.168.0.0/16
    100.64.0.0/10
    ::1/128
    fe80::/10
```

After restarting only the Moonraker service, the loaded `Trusted Clients` block correctly included `100.64.0.0/10`.

## Validated tests

The following were completed successfully:

- Phoenix node authentication in the tailnet;
- Tailscale IPv4 reachability from the Kubuntu PC;
- SSH to Phoenix using its Tailscale address;
- Mainsail loading through the Tailscale address;
- Mainsail → Moonraker WebSocket operation after adding `100.64.0.0/10` to trusted clients;
- local default route remaining unchanged;
- local DNS remaining unchanged with `--accept-dns=false`.

During the mobile-hotspot test, after the first settling packet, ping latency from the PC to Phoenix dropped to roughly the 10–20 ms range.

## Test still to complete

The August 25 test was performed while Phoenix and the PC were using the same Internet infrastructure through a mobile hotspot, even though SSH and Mainsail used Tailscale addresses.

A true remote test with the two devices on different Internet networks still needs to be explicitly validated, for example:

- Phoenix on the home network;
- PC or phone on mobile data or another external network.

No configuration changes are expected for this test: Tailscale should keep the same node identity and automatically select the available path.

## Security notes

- Do not expose Moonraker or SSH with public port forwarding only to obtain remote access: the Phoenix Tailscale model removes that need.
- `100.64.0.0/10` is added to `trusted_clients` because access is limited to peers admitted to the tailnet. Tailnet ACL/grant policy remains the proper layer for further restricting which peers can reach Phoenix.
- Do not enable subnet routing or exit-node functionality without a concrete requirement.
- Always create and verify a backup before editing `moonraker.conf`.
