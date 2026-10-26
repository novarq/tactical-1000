# FRR Installation and Configuration Guide

## Overview

This guide sets up IPv4 and IPv6 BGP routing with FRRouting (FRR) on a Tactical 1000 running Debian or Ubuntu. FRR's `bgpd` daemon handles BGP, its `zebra` daemon installs the selected routes into Linux, and the LAN969x/Sparx5 driver then programs eligible routes into the switch ASIC.

The example routes traffic between two VLANs on different IP networks. The ASIC performs the Layer 3 lookup, decrements the TTL or hop limit, and rewrites the Ethernet addresses, so no SAI, FPM connection, or separate hardware forwarding agent is needed.

## Prerequisites

- Debian or Ubuntu (arm64) installed on the Tactical 1000, with working management connectivity
- A LAN969x/Sparx5 kernel with Layer 3 routing support
- Root access through the management port or the [serial console](serial-console-access.md)
- Two unused switch ports and two external BGP peers
- No other network manager or routing daemon configuring the ports and routing table used here

Back up your existing routing and network configuration before you start. Keep the management port out of the switching bridge, and leave its addresses and default route unchanged. Run all commands as root, preferably on a system dedicated to this example.

## Example Network

Each switch port carries untagged traffic in its own VLAN, and VLAN interfaces on the bridge hold the router addresses.

| Link | Switch port / VLAN | Tactical 1000 address | Peer address |
| --- | --- | --- | --- |
| Left IPv4 | `sfp1` / 110 | `192.0.2.1/24` | `192.0.2.2/24` |
| Left IPv6 | `sfp1` / 110 | `2001:db8:110::1/64` | `2001:db8:110::2/64` |
| Right IPv4 | `sfp2` / 120 | `198.51.100.1/24` | `198.51.100.2/24` |
| Right IPv6 | `sfp2` / 120 | `2001:db8:120::1/64` | `2001:db8:120::2/64` |

The Tactical 1000 uses AS 65000, and the left and right peers use AS 65001 and AS 65002. The left peer advertises `203.0.113.1/32` and `2001:db8:1::1/128`, and the right peer advertises `203.0.113.2/32` and `2001:db8:2::1/128`. Configure these loopbacks on the peers and set AS 65000 as their remote AS.

All addresses, AS numbers, VLAN IDs, and port names are examples; replace them to suit your network. Do not announce these documentation prefixes to the Internet. Keep the two peer networks separate so that traffic between them has to cross the Tactical 1000.

## Installation Steps

### 1. Install FRR and Network Tools

Install FRR along with some network diagnostic tools, then stop it until the configuration is ready:

```sh
apt-get update
apt-get install frr frr-pythontools iproute2 iputils-ping ethtool tcpdump
systemctl stop frr.service
```

Use the FRR packages that your distribution supports.

### 2. Enable the BGP Daemon

Back up `/etc/frr/`, then set these options in `/etc/frr/daemons`:

```ini
bgpd=yes
vtysh_enable=yes
```

Leave the other packaged settings as they are, including the VTY bindings to localhost, and keep unused protocol daemons disabled. Zebra always starts, and other core daemons such as `watchfrr` and `staticd` may run alongside it; newer releases also start `mgmtd`.

Use the integrated configuration file, `/etc/frr/frr.conf`, and do not keep conflicting per-daemon configuration files alongside it.

### 3. Check the Kernel Configuration

You need a Tactical 1000 kernel that includes the LAN969x/Sparx5 Layer 3 routing driver; Novarq maintains the sources in [novarq/linux](https://github.com/novarq/linux). The kernel must also be built with `CONFIG_IP_ROUTE_MULTIPATH=y`, because the driver's nexthop-weight checks can depend on it even for single-path IPv4 routes. Check the running kernel's configuration:

```sh
uname -r
if [ -r /proc/config.gz ]; then
  zcat /proc/config.gz | grep '^CONFIG_IP_ROUTE_MULTIPATH='
else
  grep '^CONFIG_IP_ROUTE_MULTIPATH=' "/boot/config-$(uname -r)"
fi
```

The output should be:

```text
CONFIG_IP_ROUTE_MULTIPATH=y
```

This is a kernel build option, not an FRR setting or a sysctl. If the running kernel's configuration is not available on the device, get it from your image provider. Your kernel's driver documentation lists the routing features it supports.

The kernel revision and board configuration come from the [Novarq Buildroot external tree](https://github.com/novarq/buildroot-external-novarq). The [Novarq Debian/Ubuntu build workflow](https://github.com/novarq/debootstrap) builds the root filesystem with debootstrap and pairs it with a Buildroot-built kernel.

If the option is disabled, rebuild the kernel. Set up a build by following the external tree's instructions. Then, in the configured Buildroot directory on your build host, run `make linux-menuconfig`, search for `IP_ROUTE_MULTIPATH`, enable it, save, and exit. Run `make` to rebuild. Make sure the change ends up in your build inputs so that a clean rebuild keeps it; if the build uses a custom kernel configuration file, `make linux-update-defconfig` writes the change back to that file.

Deploy `output/images/uImage-lan969x.itb` and any matching kernel modules by following your image's deployment instructions, boot the new kernel, and run the check again.

### 4. Configure the VLAN-Aware Routing Interfaces

This setup places the router addresses on VLAN interfaces above a VLAN-aware bridge. Addresses assigned directly to physical ports follow a different hardware routing model, which this guide does not cover.

Check your interface names and current configuration:

```sh
ip -br link
ip -br address
bridge link show
```

The examples use `sfp1` and `sfp2`; substitute your image's port names throughout. Both ports must be unconfigured and not part of any existing bridge, and no `br-frr` bridge should exist yet. Do not add the management port to the bridge or change its addresses or default route.

Create the bridge and attach the two data ports:

```sh
ip link add br-frr type bridge vlan_filtering 1 vlan_default_pvid 0 \
  stp_state 0 mcast_snooping 1
ip link set br-frr up

ip link set sfp1 master br-frr
bridge vlan add dev sfp1 vid 110 pvid untagged
bridge vlan add dev br-frr vid 110 self
ip link set sfp1 up

ip link set sfp2 master br-frr
bridge vlan add dev sfp2 vid 120 pvid untagged
bridge vlan add dev br-frr vid 120 self
ip link set sfp2 up

# Set the router MAC after attaching the switch ports.
ip link set br-frr address 02:00:00:00:00:01
```

Set the bridge MAC address only after the switch ports are attached, so that the driver updates the router MAC in the ASIC. Keep this order in any persistent configuration as well. Choose a locally administered MAC address that is unique in your network; the one above is only an example.

Create the routed VLAN interfaces:

```sh
ip link add link br-frr name v110 type vlan id 110
ip address add 192.0.2.1/24 dev v110
ip -6 address add 2001:db8:110::1/64 dev v110
ip link set v110 up

ip link add link br-frr name v120 type vlan id 120
ip address add 198.51.100.1/24 dev v120
ip -6 address add 2001:db8:120::1/64 dev v120
ip link set v120 up
```

To enable IPv4 and IPv6 forwarding, create `/etc/sysctl.d/90-frr-forwarding.conf` with:

```ini
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply it:

```sh
sysctl -p /etc/sysctl.d/90-frr-forwarding.conf
```

Once IPv6 forwarding is enabled, Linux stops accepting Router Advertisements on interfaces that use the default `accept_ra=1`. If the management port gets its IPv6 address or default route from Router Advertisements, configure it to keep accepting them, for example by adding `net.ipv6.conf.<mgmt>.accept_ra=2` to the same file, where `<mgmt>` is the management interface name. If a network manager configures the management port, use its equivalent setting instead.

### 5. Configure IPv6 Neighbor Discovery

With the current Tactical 1000 driver, IPv6 neighbor discovery needs one extra setting. When an IPv6 address is added to a VLAN interface, the kernel announces the address's solicited-node multicast group in an MLD report, and the bridge records it as a host multicast membership. The driver copies multicast to the CPU only for groups with such a membership. The memberships expire after the bridge's multicast membership interval, 260 seconds by default, unless an MLD querier renews them, and point-to-point links usually have no querier. After that, neighbor solicitations for the Tactical 1000's addresses no longer reach Linux. Neighbors that are already talking to the switch keep working, but a neighbor that has to resolve it again, for example after a reboot, fails, and BGP sessions over IPv6 can drop. IPv4 is not affected, because ARP uses broadcast, which always reaches the CPU.

Keep the memberships in place with one of the two methods below. The per-VLAN querier needs no list of groups; the refresh timer does not make the switch send queries.

#### Option 1: Per-VLAN MLD Querier

Make the bridge the MLD querier in each routed VLAN:

```sh
ip link set dev br-frr type bridge mcast_vlan_snooping 1
bridge vlan global set dev br-frr vid 110 mcast_querier 1
bridge vlan global set dev br-frr vid 120 mcast_querier 1
```

The bridge then sends an MLD query into each VLAN every 125 seconds. The VLAN interfaces answer it like any other host, and their reports renew the memberships for every group they have joined, so nothing needs updating when the addresses change. Enable the querier per VLAN, after the VLANs are on the bridge: a bridge-wide `mcast_querier 1` sends untagged queries, which never reach the VLAN interfaces. The bridge sends the queries from its own IPv6 link-local address, which it gets automatically once it is up.

Hosts in these VLANs also receive the queries and answer them, which adds a few small packets every two minutes. How the bridge interacts with another MLD querier in the same VLAN has not been tested.

Check the result:

```sh
bridge vlan global show dev br-frr
bridge -s mdb show dev br-frr
```

VLANs 110 and 120 should show `mcast_querier 1`, and the host entries (`port br-frr`) should stay listed instead of expiring after 260 seconds.

#### Option 2: Refresh Timer

Alternatively, add the memberships yourself and refresh them before they expire. Create `/usr/local/sbin/frr-nd-refresh` with:

```sh
#!/bin/sh
set -eu
group=ff02::1:ff00:1
for vid in 110 120; do
  if ! bridge mdb replace dev br-frr port br-frr grp "$group" vid "$vid" 2>/dev/null; then
    bridge mdb del dev br-frr port br-frr grp "$group" vid "$vid" 2>/dev/null || true
    bridge mdb add dev br-frr port br-frr grp "$group" vid "$vid"
  fi
done
```

The script refreshes each entry with `bridge mdb replace`. Older iproute2 releases, including 6.1 in Debian 12 and Ubuntu 24.04, do not have that command, so the script deletes and re-adds the entry instead.

Both example IPv6 addresses map to the solicited-node group `ff02::1:ff00:1`, and so do the EUI-64 link-local addresses derived from the example MAC. A solicited-node group has the form `ff02::1:ffXX:XXXX`, where the last 24 bits come from the unicast address. Check the actual addresses with `ip -6 address show dev v110` and `ip -6 address show dev v120`. If you change the IPv6 addresses, the MAC address, or the address generation mode, recalculate the groups and add an entry for each distinct group.

Make the script executable, run it, and check the result:

```sh
chmod 0755 /usr/local/sbin/frr-nd-refresh
/usr/local/sbin/frr-nd-refresh
bridge mdb show dev br-frr
```

Use a systemd timer to run the script every 60 seconds, well within the 260-second expiry.

Create `/etc/systemd/system/frr-nd-refresh.service` with:

```ini
[Unit]
Description=Refresh FRR example IPv6 neighbor multicast memberships
After=network-online.target
ConditionPathExists=/sys/class/net/br-frr

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/frr-nd-refresh
```

Create `/etc/systemd/system/frr-nd-refresh.timer` with:

```ini
[Unit]
Description=Renew FRR example IPv6 neighbor multicast memberships

[Timer]
OnBootSec=30s
OnUnitActiveSec=60s
AccuracySec=1s

[Install]
WantedBy=timers.target
```

Start the timer:

```sh
systemctl daemon-reload
systemctl enable --now frr-nd-refresh.timer
```

The timer maintains only the groups listed in the script. The kernel joins other groups as well, such as the all-routers group `ff02::2`, and those still expire. If you adapt the configuration, make sure that neighbor discovery, Router Advertisements, and any other IPv6 control traffic you rely on still reach the CPU.

### 6. Configure BGP and Hardware-Compatible Nexthops

The example installs routes with per-route nexthops and global IPv6 nexthops. This avoids driver restrictions on nexthop objects and link-local neighbors. Check your kernel and FRR documentation to see what your versions support.

Create `/etc/frr/frr.conf` with:

```conf
frr defaults traditional
hostname tactical-frr
service integrated-vtysh-config
log syslog informational
no zebra nexthop kernel enable
!
ip prefix-list DEMO4 seq 10 permit 203.0.113.0/24 ge 32 le 32
ipv6 prefix-list DEMO6 seq 10 permit 2001:db8::/32 ge 128 le 128
!
route-map PREFER-GLOBAL permit 10
 set ipv6 next-hop prefer-global
!
router bgp 65000
 bgp router-id 192.0.2.1
 no bgp default ipv4-unicast
 bgp graceful-restart-disable
 neighbor 192.0.2.2 remote-as 65001
 neighbor 192.0.2.2 timers 3 9
 neighbor 192.0.2.2 timers connect 2
 neighbor 198.51.100.2 remote-as 65002
 neighbor 198.51.100.2 timers 3 9
 neighbor 198.51.100.2 timers connect 2
 neighbor 2001:db8:110::2 remote-as 65001
 neighbor 2001:db8:110::2 timers 3 9
 neighbor 2001:db8:110::2 timers connect 2
 neighbor 2001:db8:120::2 remote-as 65002
 neighbor 2001:db8:120::2 timers 3 9
 neighbor 2001:db8:120::2 timers connect 2
 address-family ipv4 unicast
  neighbor 192.0.2.2 activate
  neighbor 192.0.2.2 prefix-list DEMO4 in
  neighbor 192.0.2.2 prefix-list DEMO4 out
  neighbor 198.51.100.2 activate
  neighbor 198.51.100.2 prefix-list DEMO4 in
  neighbor 198.51.100.2 prefix-list DEMO4 out
 exit-address-family
 address-family ipv6 unicast
  neighbor 2001:db8:110::2 activate
  neighbor 2001:db8:110::2 prefix-list DEMO6 in
  neighbor 2001:db8:110::2 prefix-list DEMO6 out
  neighbor 2001:db8:110::2 route-map PREFER-GLOBAL in
  neighbor 2001:db8:120::2 activate
  neighbor 2001:db8:120::2 prefix-list DEMO6 in
  neighbor 2001:db8:120::2 prefix-list DEMO6 out
  neighbor 2001:db8:120::2 route-map PREFER-GLOBAL in
 exit-address-family
!
```

The configuration works as follows:

- `no zebra nexthop kernel enable` makes Zebra install each route with its own nexthops instead of using Linux nexthop objects. Use it when the driver cannot offload nexthop objects. Some FRR versions hide the command from normal command listings, so confirm that yours accepts it. It does not disable routing, or ECMP as a whole.
- The inbound `PREFER-GLOBAL` route-map makes FRR use each peer's global or ULA IPv6 nexthop instead of its link-local one. The Tactical 1000 driver ignores link-local neighbor entries, which can leave transit packets on the CPU path even when the route shows an offload flag. Apply the route-map to both IPv6 neighbors, and make sure each peer supplies a usable global or ULA nexthop.
- The prefix lists restrict imports and exports to the example host prefixes, so default routes are rejected and the management network is not advertised. The route-map does not replace these filters.
- Zebra installs the routes in the Linux main table (254). The driver offloads routes only from the main table; routes in other tables are forwarded by the CPU.
- Keepalive and hold times of 3 and 9 seconds, a 2-second connect retry, and disabled graceful restart make route withdrawal and recovery easy to observe. Choose timers and a restart policy that suit your deployment.

### 7. Validate and Start FRR

Set the file ownership and permissions, then check the syntax:

```sh
chown frr:frr /etc/frr/frr.conf
chmod 0640 /etc/frr/frr.conf
vtysh -C -f /etc/frr/frr.conf
```

Start FRR only after validation succeeds, the VLAN interfaces are up, and the neighbor discovery setup from step 5 is in place:

```sh
systemctl restart frr.service
systemctl status frr.service
vtysh -c 'show version'
vtysh -c 'show bgp summary'
```

Once the peers are configured, the two IPv4 and two IPv6 BGP sessions should reach `Established`, and each session should receive one example prefix.

If you later change the nexthop-object mode, restart FRR so that it reinstalls the routes in the new format. The restart interrupts the BGP sessions, so schedule it as maintenance.

### 8. Verify Routes and the Forwarding Path

Check BGP route selection and Zebra's routing tables:

```sh
vtysh -c 'show bgp ipv4 unicast'
vtysh -c 'show bgp ipv6 unicast'
vtysh -c 'show ip route'
vtysh -c 'show ipv6 route'
```

Then inspect the Linux side:

```sh
ip -j -d route show exact 203.0.113.2/32
ip -6 -j -d route show exact 2001:db8:2::1/128
ip -s neigh show
bridge vlan show
bridge mdb show dev br-frr
```

The routes should have these properties:

- They use protocol `bgp`, point through `v110` or `v120`, and carry the `rt_offload` flag.
- They have no `nhid` reference, because kernel nexthop objects are disabled.
- Their IPv6 nexthops are global or ULA addresses, such as the example `2001:db8:120::2`, not link-local `fe80::` addresses.

To test forwarding between the peers, use their loopbacks as source and destination. Run these commands on the left peer, not on the Tactical 1000, whose own packets originate on the CPU and do not test the same path:

```sh
ping -I 203.0.113.1 -c 5 203.0.113.2
ping -6 -I 2001:db8:1::1 -c 5 2001:db8:2::1
```

An established BGP session, a successful ping, or an offload flag does not by itself prove that the ASIC is forwarding the traffic. To confirm it, capture the same flow at the receiving peer and on the Tactical 1000 while the ping runs:

```sh
# On the right peer (replace eth1 with the interface facing the Tactical 1000)
tcpdump -eni eth1 -v 'host 203.0.113.1 or host 2001:db8:1::1'

# On the Tactical 1000
tcpdump -ni v110 'host 203.0.113.1 or host 2001:db8:1::1'
```

At the receiving peer, the packets should carry the Tactical 1000's router MAC as their source address and a TTL or hop limit one lower than the sender used. On the Tactical 1000, the capture should stay empty, because packets forwarded in hardware never reach the CPU.

An empty capture only counts if the capture itself works. Check it by pinging a Tactical 1000 address from the same source, for example with `ping -I 203.0.113.1 -c 5 192.0.2.1` on the left peer; these packets must appear. When you stop tcpdump, also make sure it reports no packets dropped by the kernel.

### 9. Persist Startup Ordering

Configure your network setup to restore the bridge, VLAN interfaces, addresses, and MAC address update order at boot, along with the neighbor discovery setup from step 5: the querier settings, applied after the VLANs are added, or the MDB memberships and their refresh timer. The forwarding sysctls already persist through the file in `/etc/sysctl.d`.

FRR must start only after that network setup completes. If a custom service sets up the network, add `Requires=` and `After=` dependencies on it in a `frr.service` drop-in, created with `systemctl edit frr.service`. Do the same for `frr-nd-refresh.service` if you use it.

Check the packaged service ordering first with `systemctl cat frr.service`. The Debian and Ubuntu units start FRR before `network.target`, so adding `After=network-online.target` creates an ordering cycle, and so does depending on a setup service that is itself ordered after `network.target`. Depend directly on the network setup service, and order that service before `network.target`.

Enable FRR once the dependencies are in place:

```sh
systemctl daemon-reload
systemctl enable frr.service
```

After a planned reboot, check the sessions, route properties, and peer-to-peer forwarding again. Also check the neighbor discovery setup: `bridge vlan global show dev br-frr` for the querier, or the refresh timer:

```sh
systemctl status frr-nd-refresh.timer
journalctl -u frr.service -u frr-nd-refresh.service -b
```

## Performance

With hardware offload, routed TCP goodput over 10 GbE links can reach about 9.4 Gbit/s for IPv4 and 9.3 Gbit/s for IPv6. CPU load during these transfers stays around 7–10%, measured across the whole system and including monitoring overhead, not just for the FRR processes.

These are reference figures, not guaranteed capacity. Actual throughput and CPU load depend on packet sizes, traffic patterns, the endpoints, and kernel and driver support.

## CPU Monitoring

To measure CPU use per process and across the whole system, install `sysstat` and take interval samples:

```sh
apt-get install sysstat
pidstat -u -C 'bgpd|zebra|mgmtd|staticd|watchfrr' 1 10
mpstat 1 10
```

Prefer interval measurements to the `%CPU` column of `ps`, which averages over the whole lifetime of a process. Keep in mind that monitoring itself uses some CPU.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| BGP shows `(Policy)` | Inbound and outbound prefix lists or route-maps on the relevant address family |
| Routes contain `nhid` and are not offloaded | `no zebra nexthop kernel enable`, followed by a controlled FRR restart |
| IPv6 route is marked offloaded but traffic hits the CPU | Whether the active nexthop is global or ULA, the inbound `PREFER-GLOBAL` route-map on both neighbors, and the nexthop's neighbor state |
| IPv4 is not offloaded | The running kernel's multipath option, driver support, the VLAN interface model, and use of the main routing table |
| IPv6 works at first, then fails | Whether the per-VLAN querier is enabled or the refresh timer is running, and the solicited-node groups |
| Routes look correct but packets fail | Bridge MAC update order, VLAN membership, and neighbor MAC addresses |
| Startup fails or reports an ordering cycle | `frr.service` dependencies and network setup ordering |

Route filters control which routes are exchanged, not which packets are allowed through. Separating traffic into VLANs at Layer 2 does not stop routing between them. Do not rely on Linux firewall rules or software forwarding settings to stop traffic that the ASIC forwards; verify hardware enforcement separately.

## Documentation

- [Official FRR Documentation](https://docs.frrouting.org/)
- [Initial Network Configuration Guide](initial-network-configuration-guide.md)
- [Linux-Powered VLAN Segmentation](linux-powered-vlan-segmentation.md)
