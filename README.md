# Agency Corporate Network — Cisco Packet Tracer Lab

A simulated corporate LAN/WLAN/WAN built in Cisco Packet Tracer. The design
covers ten department/service VLANs routed by a Layer 3 core switch, IP
telephony, a wireless BYOD segment, and a branch site reached over a
serial WAN link, with connectivity verified end-to-end via `ping` and
`tracert`.

## Topology

- **Core / distribution:** `Multilayer Switch0` (Cisco Catalyst 3650-24PS,
  hostname `SwitchL3`) — routes between all VLANs via SVIs and trunks down
  to every access switch. Also directly hosts the server farm and the
  wireless access point on access ports.
- **WAN edge:** `TB-Router` (Cisco 1941, originally `Router0`) connects the
  main site to a remote/VPN branch over a serial link.
- **Access layer:** four 2960-24TT access switches, each serving one or
  more department VLANs:
  - **Exec-Sales** — Executive + Sales (+ IP phones, printers)
  - **HR-Acc** — HR + Accounting
  - **Ops-Mktg** — Operations + Marketing
  - **IT** — IT department
- **Branch / VPN site:** `Switch4` (2960-24TT) + `VPN-Router`, serving
  `PC1-VPN`, `PC2-VPN`, and `Prn-VPN` on VLAN 110, connected back to
  `TB-Router` over a point-to-point serial link.
- **Wireless segment:** an access point (SSID `Agency`, WPA2-PSK/AES)
  serving a laptop, tablet, smartphone, smart TV, and webcam on VLAN 60.
- **IP telephony:** Cisco 7960 IP phones in the Executive and Sales zones,
  on a dedicated Telephony VLAN (50).

See `docs/screenshots/01`, `06`, `07` for the topology at earlier build
stages, and `docs/screenshots/30-full-topology-with-vpn.png` for the
current state including the VPN branch and IP phones.

## VLANs / subnets

Per the official addressing plan (`docs/screenshots/35-vlan-addressing-plan-full.png`):

| VLAN | Name / department | Network address     | Usable range                    | Gateway            |
|------|--------------------|-----------------------|------------------------------------|----------------------|
| 20   | Executive          | 192.168.20.0/24       | 192.168.20.1 – 192.168.20.253     | 192.168.20.254       |
| 21   | Sales              | 192.168.21.0/24       | 192.168.21.1 – 192.168.21.253     | 192.168.21.254       |
| 22   | HR                 | 192.168.22.0/24       | 192.168.22.1 – 192.168.22.253     | 192.168.22.254       |
| 23   | Accounting         | 192.168.23.0/24       | 192.168.23.1 – 192.168.23.253     | 192.168.23.254       |
| 24   | Operations         | 192.168.24.0/24       | 192.168.24.1 – 192.168.24.253     | 192.168.24.254       |
| 25   | Marketing          | 192.168.25.0/24       | 192.168.25.1 – 192.168.25.253     | 192.168.25.254       |
| 27   | IT                 | 192.168.27.0/24       | 192.168.27.1 – 192.168.27.253     | 192.168.27.254       |
| 30   | Servers            | 192.168.30.0/24       | 192.168.30.1 – 192.168.30.253     | 192.168.30.254       |
| 40   | Printers / Printing | 192.168.40.0/24      | 192.168.40.1 – 192.168.40.253     | 192.168.40.254       |
| 50   | Telephones / Telephony | 192.168.50.0/24    | 192.168.50.1 – 192.168.50.253     | 192.168.50.254       |
| 60   | WIFI               | 192.168.60.0/24       | 192.168.60.1 – 192.168.60.253     | 192.168.60.254       |
| 100  | Administration (switch mgmt) | 192.168.100.0/24 | 192.168.100.1 – 192.168.100.253 | 192.168.100.254   |
| 110  | VPN branch (off VPN-Router, not the core) | 192.168.110.0/24 | — | 192.168.110.254 |

VLAN 26 doesn't appear anywhere in the captured config — it's skipped
between Marketing (25) and IT (27). VLAN 100 is a routed SVI now
(`Administration`) as well as being the per-switch management addressing
scheme documented below.

**Per-switch VLAN database (confirmed for Exec-Sales):** Exec-Sales only
needs 5 of the 12 VLANs locally — 20 (Executive), 21 (Sales), 40
(Printing), 50 (Telephony), 100 (Administration) — matching the devices it
actually serves. Confirmed live via CLI
(`docs/screenshots/36-exec-sales-vlan-name-table.png`,
`docs/screenshots/37-exec-sales-vlan-creation-cli.png`); see
[`configs/exec-sales.cfg`](configs/exec-sales.cfg). The other access
switches presumably follow the same "only create the VLANs you need"
pattern, but that wasn't captured for them.

## Switch management addressing (VLAN 100)

Dedicated Layer 3 addresses for in-band switch administration:

| Device                        | IPv4 Address     | IPv6 Address              |
|--------------------------------|------------------|----------------------------|
| Multilayer Switch (3650)      | 192.168.100.1/24 | 2001:db8:acad:100::1/64   |
| Exec-Sales                    | 192.168.100.2/24 | 2001:db8:acad:100::2/64   |
| HR-Acc                        | 192.168.100.3/24 | 2001:db8:acad:100::3/64   |
| Ops-Mktg                      | 192.168.100.4/24 | 2001:db8:acad:100::4/64   |
| IT                             | 192.168.100.5/24 | 2001:db8:acad:100::5/64   |
| Network gateway                | 192.168.100.254  | 2001:db8:acad:100::254/64 |

Source plan: `docs/screenshots/13-switch-management-addressing-plan.png`.
IPv6 is documented but not yet applied/verified in any screenshot.

## Trunking (access switches → core)

| Access switch | Uplink port | Native VLAN | Allowed VLANs        | Confirmed? |
|----------------|-------------|-------------|------------------------|------------|
| Exec-Sales     | g0/1        | 100         | 20,21,40,50,100 *      | core side only — see note |
| HR-Acc         | g0/1        | 100         | 22,23,40,100           | ✅ both sides |
| Ops-Mktg       | g0/1        | 100         | 24,25,40,100           | ✅ both sides |
| IT             | g0/1        | 100         | 27,40,100              | ✅ both sides |

\* The core's Gi1/0/2 port (facing Exec-Sales) was configured with
`switchport trunk allowed vlan 20,21,40,50` — VLAN 100 wasn't included in
that `allowed` list even though it's the native VLAN. Exec-Sales's own
trunk command wasn't captured in a screenshot, so its port config in
`configs/exec-sales.cfg` is inferred/symmetric and flagged for
verification. Worth double-checking in Packet Tracer.

Core-side trunk/access port map (`Multilayer Switch0`, confirmed via
`docs/screenshots/20-21`):

| Core port          | Connects to        | Mode   | VLAN(s)                  |
|----------------------|---------------------|--------|----------------------------|
| GigabitEthernet1/0/2 | Exec-Sales          | trunk  | native 100, allowed 20,21,40,50 |
| GigabitEthernet1/0/3 | HR-Acc              | trunk  | native 100, allowed 22,23,40,50,100 |
| GigabitEthernet1/0/4 | Ops-Mktg (inferred) | trunk  | not captured |
| GigabitEthernet1/0/5 | IT                  | trunk  | native 100, allowed 27,40,50,100 |
| GigabitEthernet1/0/6 | App/File/AD server  | access | vlan 30 |
| GigabitEthernet1/0/7 | App/File/AD server  | access | vlan 30 |
| GigabitEthernet1/0/8 | App/File/AD server  | access | vlan 30 |
| GigabitEthernet1/0/9 | Access Point0       | access | vlan 60 |

## Inter-VLAN routing

`Multilayer Switch0` routes between all department/service VLANs using
SVIs (see [`configs/multilayer-switch0.cfg`](configs/multilayer-switch0.cfg),
sourced from the live CLI session in `docs/screenshots/6-9`). Every VLAN
in the table above has an SVI with a `.254` gateway address, `no shutdown`,
and a `description` labeling its role.

## WAN / VPN branch

A branch segment (VLAN 110, `192.168.110.0/24`) sits off a dedicated
`VPN-Router`, reached from the main site through `TB-Router` over a
point-to-point serial link (`docs/screenshots/30`). Addressing is now
**confirmed** via `show ip route` on both routers
(`docs/screenshots/33-vpn-router-show-ip-route.png`,
`docs/screenshots/34-tb-router-show-ip-route.png`), captured in full in
[`configs/wan-vpn-link.cfg`](configs/wan-vpn-link.cfg):

| Link / interface                    | TB-Router          | VPN-Router          |
|---------------------------------------|----------------------|------------------------|
| Serial WAN link (10.0.0.0/24)         | Se0/1/0 — 10.0.0.1   | Se0/1/1 — 10.0.0.2     |
| Link toward the core (192.168.10.0/24)| Gi0/0 — 192.168.10.1 | —                       |
| VLAN 110 LAN gateway                  | —                     | Gi0/0 — 192.168.110.254 |
| Loopback0 (192.168.200.0/24)          | 192.168.200.1        | 192.168.200.2           |

Routing is entirely static/default — no dynamic routing protocol was
observed:
- **TB-Router:** a static route to the branch (`192.168.110.0/24 via
  10.0.0.2`) plus a default route toward the core (`0.0.0.0/0 via
  192.168.10.2`).
- **VPN-Router:** just a default route back toward TB-Router (`0.0.0.0/0
  via 10.0.0.1`).

`PC1-VPN` successfully pings both the local gateway and traces a 4-hop
route all the way to `PC1-Exec` on the main site
(`docs/screenshots/32-pc1-vpn-ping-tracert.png`), which lines up with this
table:

```
1  192.168.110.254   (VPN-Router LAN gateway)
2  10.0.0.1           (TB-Router, WAN side)
3  192.168.10.2        (next hop toward the core — see note below)
4  192.168.20.1        (PC1-Exec)
```

**Open question:** TB-Router's default route points at `192.168.10.2` as
its next hop on the core-facing link, but no screenshot yet confirms which
device owns that address (presumably an interface or SVI on
`Multilayer Switch0`, but that's not in the current core config —
see Notes below).

## Verification performed

**Inter-VLAN routing, from `PC1-Exec` (192.168.20.1):**

| Destination        | Subnet       | 
|---------------------|---------------|
| 192.168.20.2        | Executive     
| 192.168.21.1         | Sales         
| 192.168.22.1         | HR          
| 192.168.23.1         | Accounting 
| 192.168.24.2         | Operations    
| 192.168.25.2         | Marketing     
| 192.168.27.1         | IT            
| 192.168.30.1         | Servers (AD)  
| 192.168.40.1         | Printing    
| 192.168.60.1         | Wifi          
| 192.168.110.254      | VPN branch gw 
| 10.0.0.1             | WAN hop       


**Branch connectivity, from `PC1-VPN` (192.168.110.x):**
- Ping to `192.168.10.1` —
- Ping to `192.168.10.2` — 
- `tracert 192.168.20.1` — 

**Wireless (from earlier build stage):** Laptop0 pinged three Wi-Fi-segment
hosts with 0% loss (`docs/screenshots/11`).

**Switch management plane:** `show ip interface brief` on Exec-Sales,
HR-Acc, and IT confirms VLAN 100 is `up`/`up` with the addresses from the
management plan (`docs/screenshots/14`, `15`, `16`).

## Repo layout

```
.
├── README.md
├── configs/
│   ├── multilayer-switch0.cfg   # core: all SVIs + trunk/access ports
│   ├── exec-sales.cfg
│   ├── hr-acc.cfg
│   ├── ops-mktg.cfg
│   ├── it-switch.cfg
│   └── wan-vpn-link.cfg         # inferred WAN/VPN addressing
└── docs/
    └── screenshots/              # Packet Tracer screenshots, in build order
        ├── 01–16   … original department network + wireless build
        ├── 17–32   … trunking, inter-VLAN SVIs, VPN branch, verification
        └── 33–37   … WAN routing tables, full VLAN plan, Exec-Sales VLAN DB
```

## Notes / next steps

- Confirm `GigabitEthernet1/0/4` (core → Ops-Mktg) trunk config and
  Exec-Sales's own `switchport trunk ...` commands with a screenshot —
  both are currently inferred rather than confirmed (Exec-Sales' VLAN
  *database* is now confirmed, but not its trunk port).
- Figure out which device owns `192.168.10.2` — TB-Router's default route
  points there, but it isn't confirmed against a `Multilayer Switch0`
  config screenshot. It's likely a core-side interface/SVI facing the WAN
  edge that just hasn't been captured yet.
- IPv6 addressing from the original plan (`2001:db8:acad:100::/64`) still
  hasn't been applied/verified anywhere.
- Capture the VLAN database creation (`vlan <id> / name ...`) on HR-Acc,
  Ops-Mktg, and IT for consistency with what's now documented for
  Exec-Sales.
- Consider adding the `.pkt` file itself if you want the repo to be fully
  reproducible in Packet Tracer.
