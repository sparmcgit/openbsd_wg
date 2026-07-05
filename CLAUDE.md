# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A single SSH-orchestrated shell script, `openbsd_wg`, that sets up a
point-to-point WireGuard tunnel between two OpenBSD servers. There is
no build/test/lint tooling — it's a standalone bash script, "run" by
executing it against two live (or sandboxed) hosts.

## Commands

```
./openbsd_wg [user@]<server1> [user@]<server2>
```

Requires root ssh access to both servers (or pass `user@host` per
argument). Safe to re-run: key generation is idempotent (an existing
`/etc/wireguard/wg0.key` is reused, never regenerated), and any existing
`/etc/hostname.wg0` is backed up (`.bak.<timestamp>`) before being
overwritten.

Syntax-check only, since the target commands (`wg`, `pkg_add`,
`/etc/netstart`) don't exist on a non-OpenBSD dev machine:

```
bash -n openbsd_wg
```

To exercise the orchestration logic without touching real servers, stub
`ssh`/`wg`/`pkg_add`/`ping` and a fake `/etc/netstart` under a sandboxed
`PATH` and `/etc` tree per fake host — see the conversation history for the
harness used to validate this script; there's no checked-in test suite.

## Architecture

### Fixed topology, not a general tool

The tunnel addressing (`192.168.5.1`/`192.168.5.2`, `/24`, port `51820`,
interface `wg0`) is hardcoded at the top of the script, not passed as
arguments — only the two server addresses/hostnames are parameters. This is
a two-node point-to-point setup, not a generic multi-peer WireGuard
provisioner.

### OpenBSD's wg(4) is native; wireguard-tools is only for keys

OpenBSD's WireGuard interface is a kernel driver configured entirely
through `ifconfig`/`hostname.if(5)` syntax (`wgkey`, `wgport`, `wgpeer`,
`wgendpoint`, `wgaip`) — there is no `wg-quick` step. The `wireguard-tools`
package is installed on each server *only* to get the `wg genkey`/`wg
pubkey` commands for key generation; the tunnel itself never depends on
that package being present afterward.

`wgkey` takes the literal base64 private key value, not a file path --
passing a path fails with `ifconfig: wgkey (key): invalid length` (ifconfig
tries to base64-decode the path string itself). The script keeps the
private key at `/etc/wireguard/wg0.key` on each host as the idempotency
source of truth (so re-runs don't regenerate it), but reads its content
back out to embed directly in `/etc/hostname.wg0`, which is why
`write_hostname_if` takes the private key as an argument rather than just
a path. `/etc/hostname.wg0` is chmod 600 after writing since it now holds
a secret.

### Every remote command forces /bin/bash explicitly

OpenBSD's root account defaults to `ksh`, not bash. Every `ssh` invocation
in this script explicitly runs `/bin/bash -c '...'` or `/bin/bash -s`
rather than relying on the login shell — this was a real bug found and
fixed in the sibling `openbsd_httpserver/relayd/deploy` script, and the
same pattern is applied here from the start. Keep doing this for any new
remote command added to this script.

### Key exchange is orchestrated locally, not peer-to-peer

The script never has the two servers talk to each other during setup.
Flow: ssh into server1 → get its pubkey; ssh into server2 → get its pubkey;
then write each server's `/etc/hostname.wg0` locally-assembled with the
*other* server's pubkey and endpoint, one `ssh` write per host. All
branching/error-handling lives in the local orchestrator; each remote
script is a short, mostly linear sequence (the one exception being the
"generate a key only if it doesn't exist yet" check, which must run
remotely since only the remote host knows if the file exists there).

### Quoting discipline

Remote commands are sent either as a single pre-built literal string via
`/bin/bash -c '...'`, or as a script body over stdin via `/bin/bash -s`
with a heredoc — never as multiple separate argv elements. `ssh` joins
trailing argv elements with plain spaces without re-quoting, so splitting
a command across multiple argv elements can silently corrupt it if any
value contains spaces. Follow this same pattern for any new remote call.
