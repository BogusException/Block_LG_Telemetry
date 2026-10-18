# Block Smart TV Telemetry with pfSense, UniFi and pfBlockerNG

Isolate a smart TV (tested with LG webOS) on its own VLAN, keep streaming working, block its tracking domains, and stop it from bypassing your DNS.

Tested on: Netgate appliance running pfSense Plus 26.03, UniFi switch and access point managed by a UniFi Network controller, pfBlockerNG-devel.

## Example values used in this guide

Replace these with your own. Pick a VLAN ID and ranges that do not collide with anything you already run.

| Item | Example |
|---|---|
| VLAN ID | `40` |
| VLAN / interface name | `IoT_TV` |
| IoT subnet | `10.40.40.0/24` (the whole IoT scheme lives in `10.40.0.0/16`) |
| pfSense address on that VLAN | `10.40.40.1` |
| DHCP range | `10.40.40.100` to `10.40.40.200` |
| pfBlockerNG DNSBL virtual IP | `10.40.41.1/32` |
| Upstream DNS | `9.9.9.9` (Quad9) and `1.1.1.1` (Cloudflare) |

## Overview

1. A dedicated VLAN and SSID keep the TV away from your internal systems.
2. Firewall rules let the TV reach the internet and pfSense DNS only.
3. pfBlockerNG DNSBL blocks tracking domains at the DNS level.
4. A firewall rule blocks direct queries to outside DNS servers, so the TV cannot skip the blocking.

---

## Part 1: pfSense VLAN and interface

### Step 1: Create the VLAN

Interfaces > Assignments > VLANs > Add

- Parent Interface: your LAN-side port (the one the UniFi switch uplinks to). Pick it by the `- lan` label in the dropdown.
- VLAN Tag: `40`
- VLAN Priority: leave blank
- Description: `IoT_TV`

Save.

### Step 2: Assign and enable the interface

Interfaces > Assignments

1. Pick `<parent>.40` (IoT_TV) from "Available network ports" and click Add.
2. Click the new interface name (for example OPT3).
3. Set:
   - **Enable interface: checked.** Do not skip this. If it is not checked, the interface will not appear in the DHCP Server list.
   - Description: `IoT_TV`
   - IPv4 Configuration Type: Static IPv4
   - IPv4 Address: `10.40.40.1` / `24`
4. Save, then Apply Changes.

### Step 3: DHCP

Services > DHCP Server > IoT_TV tab

- Enable: checked
- Range From: `10.40.40.100`
- Range To: `10.40.40.200`
- DNS Servers: `10.40.40.1` (the TV will use pfSense for DNS, which is where pfBlockerNG filters)

Save.

---

## Part 2: Firewall rules (first version)

### Step 4: Base rules

Firewall > Rules > IoT_TV > Add. Create these three rules in this order:

1. **Pass**, TCP/UDP, Source: IoT_TV subnets, Destination: IoT_TV address, Port: 53 (DNS). Description: `IoT to DNS on 10.40.40.1`
2. **Block**, Any, Source: IoT_TV subnets, Destination: LAN subnets. Check Log. Description: `Block IoT to LAN`
3. **Pass**, Any, Source: IoT_TV subnets, Destination: Any. Description: `so TV can stream`

Save each, then Apply Changes.

### Step 5: Block all private networks, not just LAN

Rule 3 would otherwise let the TV reach other internal networks (VPN tunnels, other VLANs).

1. Firewall > Aliases > IP > Add
   - Name: `PrivateNets`
   - Type: Network(s)
   - Add `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - Save, Apply Changes
2. Firewall > Rules > IoT_TV > edit rule 2. Change Destination to "Address or Alias" and enter `PrivateNets`. Save, Apply Changes.

The DNS pass rule sits above it, so the TV can still reach pfSense for DNS.

---

## Part 3: UniFi

### Step 6: Create the network

UniFi Network > Settings > Networks > New Virtual Network

- Name: `IoT_TV`
- Router: Third-party Gateway (label varies by version)
- VLAN ID: `40`

You cannot enable DHCP here. UniFi tells you it must be handled upstream. That is expected: pfSense is the DHCP server.

### Step 7: Switch ports (verify only)

UniFi trunks all VLANs by default. In Devices > your switch > Ports, check that the port to the pfSense box and the port to the AP use Native VLAN: Default and Tagged VLAN Management: Allow All. Only change them if the TV later fails to get an address.

### Step 8: Create the SSID

Settings > WiFi > Create New

- Name: `IoT_TV`
- Password: 8 or more characters
- Network: `IoT_TV` (40)
- Broadcasting APs: All
- Application: **Standard** (the "IoT" application type tunes for simple 2.4 GHz gadgets, and a TV streams better on 5 GHz)
- Radio Band: 2.4 GHz and 5 GHz
- If offered under Advanced > Manual, choose WPA2 only. Some TVs have trouble with WPA3 or mixed mode.

Connect the TV and confirm in Status > DHCP Leases that it got a `10.40.40.x` address.

---

## Part 4: Clean up stale gateways and set DNS

### Step 9: Check for stale gateway entries

System > Routing > Gateways. If you see an old gateway (for example from a retired VPN) that now displays under the new IoT_TV interface, do not delete it yet. pfSense may have a DNS server pinned to it, and disabling it will fail with "cannot be disabled because it is in use by DNS Server".

System > General Setup > DNS Server Settings

- If a DNS server row has a Gateway set to the stale gateway, change it to **None** (or remove the gateway), Save, then disable the stale gateway.
- Note: the Gateway dropdown only appears when more than one selectable gateway exists. After the stale gateway is disabled, the dropdown disappears and DNS uses the normal WAN route. That is correct.

### Step 10: Use privacy-respecting upstream DNS

Same page. Replace any Google DNS entries with:

- `9.9.9.9` with DNS Hostname `dns.quad9.net`
- `1.1.1.1` with DNS Hostname `one.one.one.one`

The hostnames are used for TLS verification in Step 17. Save. Verify with Diagnostics > DNS Lookup (`google.com`).

---

## Part 5: pfBlockerNG

### Step 11: Install

System > Package Manager > Available Packages > search `pfBlockerNG` > install **pfBlockerNG-devel**. Accept the defaults.

### Step 12: Create the DNSBL virtual IP first

The wizard requires this to exist before the DNSBL page will accept it.

Firewall > Virtual IPs > Add

- Type: IP Alias
- **Interface: Localhost** (not the IoT_TV VLAN)
- Address: `10.40.41.1` / `32`
- Description: `pfBlockerNG DNSBL`

Save, Apply Changes.

Notes:
- If you get "This IP address is being used by another interface or VIP", pick a different unused address.
- If the wizard later reports "DNSBL: IPv4 VIP not on interface Loopback", the VIP was saved on the wrong interface. Edit it and set Interface to Localhost.

### Step 13: Run the wizard

Firewall > pfBlockerNG (the wizard starts on its own). If you abandoned it, relaunch from `/pfblockerng/pfblockerng_wizard.php` on your firewall.

- Inbound Interface: WAN
- Outbound Interface: IoT_TV only (add LAN later once you trust the behavior)
- DNSBL page: IPv4 DNSBL Virtual IP `10.40.41.1`, leave ports at default (8081 and 8443), leave the whitelist checked. There is no "Enable DNSBL" checkbox on this wizard page.

Finish the wizard.

### Step 14: Enable and choose the mode

Firewall > pfBlockerNG > General: Enable pfBlockerNG and Keep Settings both checked. Save.

Firewall > pfBlockerNG > DNSBL:

- Enable DNSBL: checked
- DNSBL Mode: **Unbound Python mode** (less memory, and it logs DNS replies so you can see exactly what each device asked for)
- Save.

Saving shows "A Force Reload DNSBL must be run". That is expected. Go to Update, select **Reload**, then **DNSBL**, click Run, and wait for "UPDATE PROCESS ENDED".

### Step 15: Confirm the DNS Resolver settings

Services > DNS Resolver > General Settings

- Enable DNS Resolver: checked
- Network Interfaces: All (or at least IoT_TV)
- Outgoing Network Interfaces: WAN
- Enable Python Module: checked, Python Module Order: Pre Validator, Python Module Script: `pfb_unbound`
- DNS Query Forwarding: Enable Forwarding Mode checked

### Step 16: Test the blocking

Restart the TV's Wi-Fi connection so it picks up pfSense DNS, then open a streaming app. Check Firewall > pfBlockerNG > Reports > Alerts > DNSBL Python. You should see rows with the TV's address and the blocked domains, and streaming should still play.

### Step 17: Encrypt upstream lookups (optional)

Services > DNS Resolver > General Settings: check "Use SSL/TLS for outgoing DNS Queries to Forwarding Servers". Save, Apply Changes. Re-test a DNS lookup. If it fails, uncheck it; your ISP or upstream may be blocking port 853.

If you swap upstream DNS servers later, update the DNS Hostname too, or TLS lookups will fail.

---

## Part 6: Close the DNS bypass

A TV can query an outside DNS server directly and skip DNSBL entirely. Block that.

### Step 18: Port alias

Firewall > Aliases > Ports > Add

- Name: `DNS_Ports`
- Type: Port(s)
- Ports: `53` and `853`
- Description: `DNS and DNS over TLS`

Save, Apply Changes.

### Step 19: Block rule and ordering

Firewall > Rules > IoT_TV > Add

- Action: Block
- Protocol: TCP/UDP
- Source: IoT_TV subnets
- Destination: Any
- Destination Port Range: From and To both `DNS_Ports`
- Log: checked
- Description: `Block outside DNS`

Final rule order, top to bottom:

1. pfBlockerNG's own rule (added automatically, leave it at the top)
2. Pass TCP/UDP to IoT_TV address port 53
3. **Block outside DNS** (must be below rule 2, or the TV loses DNS from pfSense too)
4. Block to PrivateNets
5. Pass Any (stream)

Save the list and Apply Changes.

### Step 20: Verify the bypass is closed

From a device on the IoT_TV Wi-Fi:

```
nslookup google.com 8.8.8.8
nslookup google.com
```

The first must time out. The second must resolve, and the server shown must be `10.40.40.1`.

---

## Part 7: Add smart TV telemetry feeds

The default StevenBlack ad list catches some trackers, but not vendor telemetry. Add a smart TV list.

Firewall > pfBlockerNG > DNSBL > DNSBL Groups > Add

- Name: `LG_SmartTV`
- Description: `Smart TV telemetry`
- DNSBL Source Definitions > Add:
  - Format: Auto
  - State: ON
  - Source: `https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV.txt`
  - Header/Label: `Perflyst_SmartTV`
- Action: Unbound
- Update Frequency: Once a day

Save. Then Update > Reload > DNSBL > Run. The log summary should list `Perflyst_SmartTV` with a fresh timestamp and the sanity check should read PASSED.

---

## After setup

- Use the TV normally for a day, then review Reports > Alerts > DNSBL Python. Domains that keep appearing as blocked are working. If an app breaks, the `+` icon next to a domain on that page whitelists it.
- If you see LG or other vendor domains that are not blocked, add them to a custom DNSBL list.
- Known gap: apps that use DNS over HTTPS on port 443 are not caught by the port 53 and 853 rule. pfBlockerNG has a public DoH server IP list you can add as an IP feed to cover this.
- Add the LAN as an additional outbound interface in pfBlockerNG only after you are comfortable with how the blocklists behave.
