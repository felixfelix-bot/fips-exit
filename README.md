# fips-exit

Turns a Linux [fips](https://github.com/jmcorgan/fips) node into an **exit
node**: mesh clients send it their internet traffic, IPv4 and IPv6, and it
forwards that traffic to the legacy internet from its own addresses.

The client side is in [fips2go](https://github.com/fr34aky/fips2go)
(Settings → Advanced → Exit node). Any Linux fips node can be a client too,
with the kernel's `ip6tnl` (see `test/netns.sh` for the exact commands).

## How it works

Transport only. A client's packet becomes the payload of a mesh packet to
the exit, with next-header 4 (IPv4) or 41 (IPv6) — plain IP-in-IPv6,
RFC 2473. There is no tunnel header and no second encryption: fips already
encrypts and authenticates every packet end to end, and the outer source
address *is* the client's identity. On the exit this is the kernel's own
`ip6tnl`, so no daemon runs here; `fips-exit` only configures the kernel:

- one `ip6tnl` device per client (`fx` + 12 hex digits of its address),
  which is also the allow-list: no device, no traffic;
- NAT (masquerade) to the uplink for both families;
- an nftables table that binds each device to its client's two inner
  addresses, lets clients reach the internet only (not this host, its other
  networks, or each other), and clamps the TCP MSS;
- a routing rule so that replies to a client's IPv6 traffic go back into its
  tunnel.

Addresses inside the tunnel need no negotiation. A client's IPv6 source is
its fips address (`fd…`). Its IPv4 source is `10.` followed by the three
bytes after the `fd` of that address; both sides derive it, and `add`
refuses a client whose derived address clashes with an existing one.

## Install

Needs: a running fips with its TUN (`fips0`), Python 3, iproute2 (`ip`,
`tc`), nftables, and a kernel with `ip6_tunnel`, `cls_u32`, `act_pedit` and
`act_csum` (every mainstream distribution).

```sh
sudo install -m 755 fips-exit /usr/local/sbin/fips-exit
sudo install -m 644 fips-exit.service /etc/systemd/system/
sudo systemctl enable --now fips-exit
```

## Use

```sh
sudo fips-exit add npub1…  alice-phone   # allow a client (npub or fd… address)
sudo fips-exit list
sudo fips-exit remove alice-phone
sudo fips-exit status                    # devices and their packet counters
sudo fips-exit down                      # remove everything this script made
```

The client list is `/etc/fips-exit/clients`; `up` rebuilds everything from
it, so it is safe to run again at any time. `/etc/fips-exit/config` takes
`WAN=<interface>` (default: the interface of the default route) and
`MESH_IF=<interface>` (default `fips0`).

On the client, enter this node's npub as the exit node. A direct fips link
between client and exit (add the exit as a peer) keeps latency down; it is
not required.

## Packet sizes

A mesh packet is at most 1280 bytes at fips's default transport MTU, so 1240
bytes are left for the inner packet.

| Traffic | What happens |
| --- | --- |
| TCP, both families | MSS clamped in both directions (1200 / 1180); never larger than 1240. |
| Client → internet, up to 1280 bytes | The client sends two fragments of the *outer* packet; the kernel here reassembles them. |
| Internet → client, IPv4, up to 1280 bytes | Fragmented as IPv4 by this host and reassembled by the client. For that, DF is cleared on IPv4/UDP arriving on the uplink (a `tc` filter). |
| Internet → client, IPv6, 1241–1280 bytes, not TCP | **Lost.** Linux does not fragment the outer packet of an `ip6tnl` tunnel, and IPv6 has no in-path fragmentation. |

The last row is mostly QUIC over IPv6: such a connection stalls or falls
back to TCP. Two ways out, neither in this version:

- carry 1320-byte mesh packets between client and exit (a fips transport
  MTU of about 1400 and a matching `tun.mtu` on both ends, over a direct
  link), after which nothing is fragmented at all;
- a small daemon here in place of the kernel tunnel, which fragments the
  outer packet as the client already does — and is also where a carrier
  over fips's native datagram API would live.

## What the operator and the client should know

- The exit sees its clients' internet traffic as any VPN endpoint does, and
  the internet sees the exit's address.
- fips2go's DNS proxy does not use the exit yet: lookups leave from the
  phone's own connection.
- `fips-exit` adds its own nftables table and opens two firewalls it knows:
  fips's own (a drop-in `fips.d/fips-exit.nft` admitting next-header 4, 41
  and 44 from the clients' addresses; `FIPS_FIREWALL=` in the config to
  leave it alone) and ufw (runtime rules in its before-chains, tagged
  `fips-exit`; `ufw reload` drops them and `fips-exit up` restores them).
  Any other host firewall that drops by default must allow that input on
  the mesh interface and forwarding for `fx*` devices itself.
- `up` turns on IPv4 and IPv6 forwarding, and sets `accept_ra=2` on the
  uplink if it was 1, so an SLAAC uplink keeps its default route. `down`
  leaves forwarding as it is.

## Test

```sh
unshare -Urnm test/netns.sh
```

Runs without root and without fips: network namespaces stand in for two
clients, the exit and the internet, and a 1280-byte link stands in for the
mesh. It checks both families, NAT, the size cases above, TCP, the binding
of a client to its addresses, isolation, removal and teardown.
