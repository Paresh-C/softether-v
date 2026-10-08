# SoftEther VPN Client on Ubuntu (CLI Setup, Boot Service & Control Script)

A practical guide to running the **SoftEther VPN Client** on Ubuntu without a GUI. The official Connection Manager GUI is Windows-only, so on Linux the client is managed with `vpncmd`. This guide covers one-time setup, DHCP on the virtual adapter, a systemd service for boot, a small `vpn` control script, and monitoring.

> **Tested on:** Ubuntu 26.04, SoftEther VPN Client v4.44 build 9807
> **Placeholders used below:** replace `<SERVER_IP>`, `<PORT>`, `<HUB>`, `<USERNAME>` with the values from your VPN administrator.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install the client](#2-install-the-client)
3. [One-time configuration](#3-one-time-configuration)
4. [Obtain an IP address (DHCP)](#4-obtain-an-ip-address-dhcp)
5. [Routing](#5-routing)
6. [Run at boot with systemd](#6-run-at-boot-with-systemd)
7. [Control script: `vpn up | down | status`](#7-control-script-vpn-up--down--status)
8. [Monitoring and logs](#8-monitoring-and-logs)
9. [Troubleshooting](#9-troubleshooting)
10. [Security notes](#10-security-notes)

---

## 1. Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential isc-dhcp-client
```

- `isc-dhcp-client` provides `dhclient`, which is no longer installed by default on recent Ubuntu releases.
- `build-essential` is only needed if you compile the client from source.

## 2. Install the client

Download the **Linux / SoftEther VPN Client** package from the [official SoftEther download page](https://www.softether-download.com/en.aspx?product=softether), then build it:

```bash
tar xzf softether-vpnclient-*-linux-x64-64bit.tar.gz
cd vpnclient
make            # accept the license terms
cd ..
sudo mv vpnclient /usr/local/
sudo chmod 600 /usr/local/vpnclient/*
sudo chmod 700 /usr/local/vpnclient/vpnclient /usr/local/vpnclient/vpncmd
```

Verify the installation:

```bash
ls -lah /usr/local/vpnclient
# expected: vpnclient  vpncmd  hamcore.se2  ...
```

## 3. One-time configuration

Start the daemon and open the management console:

```bash
sudo /usr/local/vpnclient/vpnclient start
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT
```

Inside `vpncmd`:

```text
NicCreate VPN
AccountCreate MyVPN /SERVER:<SERVER_IP>:<PORT> /HUB:<HUB> /USERNAME:<USERNAME> /NICNAME:VPN
AccountPasswordSet MyVPN /TYPE:standard
AccountConnect MyVPN
AccountStatusGet MyVPN
```

Notes:

- `AccountPasswordSet` **without** `/PASSWORD:` makes `vpncmd` prompt for the password, so it is not stored in your shell history or visible on screen.
- `AccountStatusGet` should report `Connection Completed (Session Established)`.
- SoftEther exposes the virtual adapter as `vpn_<nicname>` in lowercase. For `NicCreate VPN` the interface is **`vpn_vpn`**.

## 4. Obtain an IP address (DHCP)

An established session only gives you a layer-2 link. The adapter still needs an address:

```bash
sudo dhclient -v vpn_vpn
ip -br a show vpn_vpn
```

If no lease is received within roughly 30 seconds, the hub does not provide DHCP. Ask your administrator for a static address and configure it manually:

```bash
sudo ip addr add <STATIC_IP>/<PREFIX> dev vpn_vpn
```

## 5. Routing

By default `dhclient` may install a default route through the VPN, sending **all** traffic over it. For split tunnelling (only company traffic through the VPN), remove it:

```bash
sudo ip route del default dev vpn_vpn
```

Verify that traffic to your internal host uses the VPN interface:

```bash
ip route get <INTERNAL_HOST_IP>
ping -c3 <INTERNAL_HOST_IP>
```

If the output shows a different interface (for example your Wi-Fi), your local network probably uses the same subnet as the remote one. Add an explicit host route:

```bash
sudo ip route add <INTERNAL_HOST_IP>/32 dev vpn_vpn
```

## 6. Run at boot with systemd

Create `/etc/systemd/system/vpnclient.service`:

```ini
[Unit]
Description=SoftEther VPN Client
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStart=/usr/local/vpnclient/vpnclient start
ExecStop=/usr/local/vpnclient/vpnclient stop

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vpnclient
systemctl status vpnclient
```

### Choose a connection mode

| Mode | When to use | Setup |
|------|-------------|-------|
| **Automatic** | Always-on VPN | `AccountStartupSet MyVPN` |
| **Manual** (recommended) | Connect only when needed | `AccountStartupRemove MyVPN` and use the script below |

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountStartupSet MyVPN
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountStartupRemove MyVPN
```

> With automatic mode the session reconnects by itself, but DHCP on `vpn_vpn` still has to be triggered after each connection (script or a networkd/NetworkManager rule).

## 7. Control script: `vpn up | down | status`

Save as `~/bin/vpn` (make sure `~/bin` is in your `PATH`) and run `chmod +x ~/bin/vpn`:

```bash
#!/usr/bin/env bash
# vpn - manage a SoftEther VPN Client connection
set -euo pipefail

ACCOUNT="MyVPN"
IFACE="vpn_vpn"
VPNCMD=(sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD)

wait_for_iface() {
  for _ in {1..15}; do
    ip link show "$IFACE" &>/dev/null && return 0
    sleep 1
  done
  echo "Interface $IFACE did not appear" >&2
  return 1
}

case "${1:-}" in
  up)
    sudo systemctl start vpnclient
    sleep 2
    "${VPNCMD[@]}" AccountConnect "$ACCOUNT" >/dev/null
    wait_for_iface
    sleep 3
    sudo dhclient -v "$IFACE"
    sudo ip route del default dev "$IFACE" 2>/dev/null || true
    echo "VPN is up."
    ;;
  down)
    sudo dhclient -r "$IFACE" 2>/dev/null || true
    "${VPNCMD[@]}" AccountDisconnect "$ACCOUNT" >/dev/null
    echo "VPN is down."
    ;;
  status)
    "${VPNCMD[@]}" AccountStatusGet "$ACCOUNT" \
      | grep -E "Session Status|Established since|Data Size"
    ip -br a show "$IFACE"
    ;;
  *)
    echo "usage: vpn {up|down|status}" >&2
    exit 1
    ;;
esac
```

Usage:

```bash
vpn up
vpn status
vpn down
```

## 8. Monitoring and logs

| Goal | Command |
|------|---------|
| Connection state, uptime, traffic | `vpn status` |
| Live link up/down events | `ip monitor link` |
| SoftEther connect / disconnect / retry log | `sudo tail -f /usr/local/vpnclient/client_log/*.log` |
| Daemon logs (systemd) | `journalctl -u vpnclient -f` |
| Interface address and routes | `ip -br a show vpn_vpn` / `ip route` |

## 9. Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `dhclient: command not found` | Install it: `sudo apt install isc-dhcp-client` |
| `Error code: 34` on `AccountCreate` | The connection setting already exists. Skip creation, or `AccountDelete MyVPN` first |
| Session established but no IP | Hub has no DHCP. Request a static IP from the administrator |
| Internal host unreachable | Check `ip route get <HOST>`. Add a `/32` route via `vpn_vpn` if another interface wins |
| All internet traffic slow after connecting | Default route points at the VPN. Run `sudo ip route del default dev vpn_vpn` |
| Interface `vpn_vpn` missing | Daemon not running (`systemctl status vpnclient`) or adapter not created (`NicList` in `vpncmd`) |

## 10. Security notes

- **Never pass passwords on the command line** (`/PASSWORD:...`). They end up in shell history and scrollback. Let `vpncmd` prompt for them.
- If a password was exposed (pasted in a chat, ticket or terminal log), ask the administrator to rotate it.
- Keep `/usr/local/vpnclient` readable by root only (`chmod 600` on config, `700` on binaries), as shown in the install step. The client config contains credentials.
- Do not publish real server addresses, usernames or passwords in public gists or repositories.

---

## References

- [SoftEther VPN Project](https://www.softether.org/)
- [`vpncmd` command reference](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual)
