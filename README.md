# SoftEther VPN Client on Ubuntu (CLI Setup & Control Script)

A practical guide to running the **SoftEther VPN Client** on Ubuntu without a GUI. The official Connection Manager GUI is Windows-only, so on Linux the client is managed with `vpncmd`.

This guide covers:

* Installing the SoftEther VPN Client
* Creating and configuring a VPN connection
* Creating the virtual network adapter
* Obtaining a VPN address through DHCP
* Split-tunnel routing
* Managing the connection with a simple `vpn` CLI script
* Monitoring connection status and routes
* Troubleshooting common problems

> **Tested on:** Ubuntu 26.04, SoftEther VPN Client v4.44 build 9807
> **Placeholders:** Replace values such as `<VPN_ACCOUNT>`, `<SERVER_IP>`, `<PORT>`, `<HUB>`, and `<USERNAME>` with the values provided by your VPN administrator.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install the client](#2-install-the-client)
3. [One-time configuration](#3-one-time-configuration)
4. [Obtain an IP address (DHCP)](#4-obtain-an-ip-address-dhcp)
5. [Routing and split tunneling](#5-routing-and-split-tunneling)
6. [Starting the SoftEther client](#6-starting-the-softether-client)
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

* `isc-dhcp-client` provides `dhclient`.
* `build-essential` is only required if compiling the SoftEther client from source.

Verify:

```bash
dhclient --version
```

---

## 2. Install the client

Download the **Linux / SoftEther VPN Client** package from the [official SoftEther download page](https://www.softether-download.com/en.aspx?product=softether).

Extract and build it:

```bash
tar xzf softether-vpnclient-*-linux-x64-64bit.tar.gz
cd vpnclient

make
```

Accept the license terms when prompted.

Then install it:

```bash
cd ..
sudo mv vpnclient /usr/local/
sudo chmod 600 /usr/local/vpnclient/*
sudo chmod 700 /usr/local/vpnclient/vpnclient
sudo chmod 700 /usr/local/vpnclient/vpncmd
```

Verify:

```bash
ls -lah /usr/local/vpnclient
```

You should see files such as:

```text
vpnclient
vpncmd
hamcore.se2
```

The two important executables are:

```text
/usr/local/vpnclient/vpnclient
/usr/local/vpnclient/vpncmd
```

---

## 3. One-time configuration

Start the SoftEther VPN Client:

```bash
sudo /usr/local/vpnclient/vpnclient start
```

Then open the management console:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT
```

Inside `vpncmd`, create the virtual adapter:

```text
NicCreate VPN
```

Create the VPN connection:

```text
AccountCreate <VPN_ACCOUNT> /SERVER:<SERVER_IP>:<PORT> /HUB:<HUB> /USERNAME:<USERNAME> /NICNAME:VPN
```

Set the password:

```text
AccountPasswordSet <VPN_ACCOUNT> /TYPE:standard
```

`AccountPasswordSet` without `/PASSWORD:` makes `vpncmd` prompt for the password, so the password is not exposed in shell history or the command line.

Connect:

```text
AccountConnect <VPN_ACCOUNT>
```

Check the connection:

```text
AccountStatusGet <VPN_ACCOUNT>
```

A successful connection should report:

```text
Session Status | Connection Completed (Session Established)
```

### Verify the account

From the normal shell:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountList
```

The output should contain your configured VPN connection setting.

### Verify the virtual adapter

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD NicList
```

The SoftEther adapter in this guide is named:

```text
VPN
```

On Linux, this appears as:

```text
vpn_vpn
```

Verify it:

```bash
ip -br link
```

Expected:

```text
vpn_vpn    UNKNOWN    ...
```

---

## 4. Obtain an IP address (DHCP)

An established SoftEther session provides a Layer-2 connection, but the Linux interface still needs an IP address.

Run:

```bash
sudo dhclient -v vpn_vpn
```

Then:

```bash
ip -br a show vpn_vpn
```

You should see an address assigned to the VPN interface.

If no DHCP lease is received, the VPN hub may not provide DHCP.

In that situation, ask the VPN administrator for the appropriate static configuration.

For a static address:

```bash
sudo ip addr add <VPN_IP>/<PREFIX> dev vpn_vpn
```

---

## 5. Routing and split tunneling

This setup uses **split tunneling**.

Normal Internet traffic continues to use the physical network connection, while specific internal networks are routed through the VPN.

Check the default route:

```bash
ip route
```

For example:

```text
default via <LOCAL_GATEWAY> dev <PHYSICAL_INTERFACE>
```

This means Internet traffic is not automatically sent through the VPN.

Check where a public destination will go:

```bash
ip route get 1.1.1.1
```

It should use your normal network interface.

### VPN routes

VPN-specific routes may look like:

```text
<INTERNAL_NETWORK_1> via <VPN_GATEWAY> dev vpn_vpn
<INTERNAL_NETWORK_2> via <VPN_GATEWAY> dev vpn_vpn
<INTERNAL_HOST> via <VPN_GATEWAY> dev vpn_vpn
```

Check the complete routing table:

```bash
ip route
```

Check which interface will be used for a particular destination:

```bash
ip route get <INTERNAL_HOST_IP>
```

The result should use:

```text
dev vpn_vpn
```

### Adding an explicit route

If an internal host is not automatically routed through the VPN:

```bash
sudo ip route add <INTERNAL_HOST_IP>/32 via <VPN_GATEWAY> dev vpn_vpn
```

For an entire network:

```bash
sudo ip route add <INTERNAL_NETWORK>/<PREFIX> via <VPN_GATEWAY> dev vpn_vpn
```

### Important

Do **not** blindly delete the default route through the VPN.

If your goal is split tunneling, the normal default route should remain on the physical network interface.

---

## 6. Starting the SoftEther client

This setup can run the SoftEther daemon directly without requiring a systemd service.

Start the daemon:

```bash
sudo /usr/local/vpnclient/vpnclient start
```

Then connect the account:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountConnect <VPN_ACCOUNT>
```

Check the connection:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountStatusGet <VPN_ACCOUNT>
```

To disconnect:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountDisconnect <VPN_ACCOUNT>
```

### Optional systemd service

If you want the SoftEther daemon to start automatically at boot, create:

```text
/etc/systemd/system/vpnclient.service
```

with:

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

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vpnclient
```

Check:

```bash
systemctl status vpnclient
```

> The control script below does not require this systemd service. It starts the SoftEther daemon directly.

---

## 7. Control script: `vpn up | down | status`

A small shell script makes the VPN easier to manage.

Create:

```bash
mkdir -p ~/bin
nano ~/bin/vpn
```

Make sure `~/bin` is in your `PATH`:

```bash
echo "$PATH" | tr ':' '\n' | grep -Fx "$HOME/bin"
```

If it is not:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Make the script executable:

```bash
chmod +x ~/bin/vpn
```

### Current script

```bash
#!/usr/bin/env bash

# vpn - manage a SoftEther VPN Client connection

set -euo pipefail

ACCOUNT="<VPN_ACCOUNT>"
IFACE="vpn_vpn"

VPNCMD=(
  sudo
  /usr/local/vpnclient/vpncmd
  localhost
  /CLIENT
  /CMD
)

wait_for_iface() {
  for _ in {1..15}; do
    if ip link show "$IFACE" &>/dev/null; then
      return 0
    fi

    sleep 1
  done

  echo "Interface $IFACE did not appear" >&2
  return 1
}

case "${1:-}" in

  up)
    sudo /usr/local/vpnclient/vpnclient start
    sleep 2

    "${VPNCMD[@]}" AccountConnect "$ACCOUNT" >/dev/null

    wait_for_iface
    sleep 3

    if ! ip -4 addr show "$IFACE" | grep -q '<VPN_SUBNET_PREFIX>'; then
      sudo dhclient -v "$IFACE"
    fi

    echo "VPN is up."
    ;;

  down)
    sudo dhclient -r "$IFACE" 2>/dev/null || true

    "${VPNCMD[@]}" AccountDisconnect "$ACCOUNT" >/dev/null

    echo "VPN is down."
    ;;

  status)
    "${VPNCMD[@]}" AccountStatusGet "$ACCOUNT"

    echo
    ip -br a show "$IFACE"

    echo
    echo "Routing:"
    ip route
    ;;

  *)
    echo "usage: vpn {up|down|status}" >&2
    exit 1
    ;;

esac
```

> Replace `<VPN_SUBNET_PREFIX>` with the VPN subnet prefix appropriate for your environment, for example `192.168.x.`. Alternatively, remove the address check and always run `dhclient`.

### Usage

Connect:

```bash
vpn up
```

Check status:

```bash
vpn status
```

Disconnect:

```bash
vpn down
```

### What `vpn up` does

1. Starts the SoftEther VPN Client daemon.
2. Connects the configured VPN account.
3. Waits for the virtual adapter.
4. Checks whether the adapter already has an IPv4 address.
5. Runs DHCP if necessary.
6. Leaves the normal Internet default route untouched.

### What `vpn down` does

1. Releases the DHCP lease.
2. Disconnects the SoftEther VPN account.
3. Leaves the SoftEther daemon running.

Keeping the daemon running makes the next:

```bash
vpn up
```

simpler because only the VPN connection itself needs to be re-established.

### What `vpn status` does

Displays:

* SoftEther session status
* VPN server information
* Encryption information
* Session statistics
* Traffic statistics
* VPN interface address
* Current routing table

---

## 8. Monitoring and logs

### Connection status

```bash
vpn status
```

Or directly:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountStatusGet <VPN_ACCOUNT>
```

A successful connection contains:

```text
Session Status | Connection Completed (Session Established)
```

### Interface

```bash
ip -br a show vpn_vpn
```

### Routing

```bash
ip route
```

### Determine the route for a destination

```bash
ip route get <IP>
```

### Monitor link changes

```bash
ip monitor link
```

### SoftEther logs

```bash
sudo tail -f /usr/local/vpnclient/client_log/*.log
```

### systemd logs

Only relevant if you install the optional systemd service:

```bash
journalctl -u vpnclient -f
```

---

## 9. Troubleshooting

### `vpn: command not found`

Check:

```bash
command -v vpn
```

If `~/bin` is missing from `PATH`:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Then:

```bash
command -v vpn
```

Expected:

```text
/home/<USER>/bin/vpn
```

### `Unit vpnclient.service not found`

If you are not using the optional systemd configuration, start the client directly:

```bash
sudo /usr/local/vpnclient/vpnclient start
```

The control script already uses this method.

### `The specified VPN Connection Setting does not exist`

List configured accounts:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD AccountList
```

Make sure the account name in the script matches the configured connection:

```bash
ACCOUNT="<VPN_ACCOUNT>"
```

### `dhclient: command not found`

Install:

```bash
sudo apt install isc-dhcp-client
```

### Session established but no IP address

Check:

```bash
ip -br a show vpn_vpn
```

Then:

```bash
sudo dhclient -v vpn_vpn
```

If DHCP fails, the VPN hub may not provide DHCP.

### VPN interface is missing

Check:

```bash
ip -br link
```

Then check the SoftEther adapters:

```bash
sudo /usr/local/vpnclient/vpncmd localhost /CLIENT /CMD NicList
```

The expected SoftEther adapter is:

```text
VPN
```

which appears on Linux as:

```text
vpn_vpn
```

### Internal host is unreachable

Check:

```bash
ip route get <INTERNAL_HOST_IP>
```

The route should use:

```text
dev vpn_vpn
```

If another interface wins, add an explicit route if appropriate:

```bash
sudo ip route add <INTERNAL_HOST_IP>/32 via <VPN_GATEWAY> dev vpn_vpn
```

### Internet traffic is not using the VPN

This is expected with split tunneling.

Check:

```bash
ip route get 1.1.1.1
```

The result should use your normal physical network interface.

### `git pull` stops working after `vpn down`

If your Git server is only reachable through the VPN, disconnecting the VPN will make it unreachable.

Check:

```bash
ip route get <GIT_SERVER_IP>
```

With the VPN connected, the route should use:

```text
dev vpn_vpn
```

---

## 10. Security notes

* **Never pass passwords on the command line** using `/PASSWORD:...`.
* Use `AccountPasswordSet` interactively so credentials do not appear in shell history.
* If a password is accidentally exposed in a terminal log, chat, ticket, or repository, rotate it.
* Keep `/usr/local/vpnclient` restricted to root where practical.
* Do not commit VPN credentials to Git.
* Do not publish real VPN server addresses, usernames, session keys, internal addresses, or other connection details in public repositories.
* Avoid publishing complete `AccountStatusGet` output because it may contain connection-specific information.

---

## References

* [SoftEther VPN Project](https://www.softether.org/)
* [SoftEther Download Page](https://www.softether-download.com/en.aspx?product=softether)
* [`vpncmd` Command Reference](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual)
