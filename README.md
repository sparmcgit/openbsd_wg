# openbsd_wg

Sets up a point-to-point WireGuard tunnel between two OpenBSD 7.9 servers,
one `wg0` interface on each end, persisted across reboots via
`/etc/hostname.wg0`.

```
server1 (wg0: 192.168.5.1/24) <====UDP 51820====> server2 (wg0: 192.168.5.2/24)
```

## Usage

```
./openbsd_wg [user@]<server1> [user@]<server2>
```

Requires root ssh access to both servers (pass `user@host` if you connect
as a different user). Example:

```
./openbsd_wg 203.0.113.10 203.0.113.20
./openbsd_wg admin@vpn1.example.com admin@vpn2.example.com
```

## What it does

1. On each server: installs `wireguard-tools` if missing (only needed for
   `wg genkey`/`wg pubkey` — OpenBSD's WireGuard interface itself is a
   native kernel driver, no userland daemon required), generates a private
   key at `/etc/wireguard/wg0.key` if one doesn't already exist, and reads
   back its public key.
2. Backs up any existing `/etc/hostname.wg0` (`.bak.<timestamp>`), then
   writes a new one on each server referencing the *other* server's public
   key and endpoint address.
3. Applies the config live with `sh /etc/netstart wg0` on both servers —
   no reboot needed.
4. Verifies the tunnel by pinging server2's tunnel address from server1,
   and reports success or failure.

Safe to re-run: it won't regenerate an existing key or blindly clobber an
existing `hostname.wg0` without backing it up first.

## Fixed parameters

The tunnel addressing is hardcoded at the top of the script (edit there to
change):

| Parameter | Value |
|---|---|
| Interface | `wg0` |
| server1 address | `192.168.5.1/24` |
| server2 address | `192.168.5.2/24` |
| Port | `51820` (UDP) |

## Requirements

- Root ssh access to both servers (or a user with permission to write
  `/etc/hostname.wg0`, `/etc/wireguard/`, and run `pkg_add`/`ifconfig`).
- Outbound package-mirror access on both servers if `wireguard-tools` isn't
  already installed.
- UDP port 51820 open between the two servers (firewall/pf rules on either
  end aren't managed by this script).
