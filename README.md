# Block Smart TV Telemetry with pfSense, UniFi and pfBlockerNG

## The problem

Smart TVs (LG webOS in this case, but Samsung, Roku, Vizio and others behave the same way) report back to their makers constantly. That traffic includes ad and analytics beacons, device identifiers, and automatic content recognition (ACR), which fingerprints what is on the screen, including content from your own HDMI inputs, and uploads it to build a viewing profile.

### NOTE: If you are securing a non-LG Smart TV, or device like Roku, I have an appendix for you at the end. The place this data is useful is just prior to this document's Part 7.
Up until Part 7, the solution is not TV/device-dependent.

---
On a flat home or small-office network this causes three problems:

1. **Privacy.** The TV sends viewing data to advertisers and data brokers, and the vendor settings that claim to turn it off are hard to find and often incomplete.
2. **Exposure.** The TV sits on the same network as your computers, servers and file shares. A TV is a poorly patched, closed-source device that you cannot audit, and it can see everything else on your LAN.
3. **No visibility.** You cannot tell what the TV is talking to, so you cannot tell whether anything you tried actually worked.

## What the solution needs to accomplish

- Keep the TV fully usable for streaming.
- Block telemetry, advertising and tracking traffic from the TV.
- Prevent the TV from reaching any internal system, now or later.
- Prevent the TV from getting around the blocking by using its own DNS servers.
- Show you exactly which domains the TV asks for and which ones were blocked.
- Affect only the TV. Everything else on your network stays exactly as it is.
- Keep working with no ongoing manual effort.

## Features, functions and abilities

| Capability | How it is done |
|---|---|
| **Network isolation** | The TV gets its own VLAN and its own Wi-Fi SSID, with its own subnet and DHCP scope on the firewall. |
| **No access to internal systems** | A firewall rule blocks the TV from every private address range (10.x, 172.16.x, 192.168.x), including VPN tunnels and other VLANs. Only the internet and the firewall's DNS are reachable. |
| **Streaming keeps working** | After the block rules, a final pass rule sends everything else to the internet, so Netflix, YouTube and similar apps work normally. |
| **DNS-level tracker blocking** | pfBlockerNG DNSBL answers lookups for known ad, tracker and telemetry domains with nothing, so the TV never connects to them. |
| **Smart TV specific blocklist** | A community-maintained smart TV list (Perflyst SmartTV) is added on top of the general ad list and refreshes daily. |
| **No DNS bypass** | A firewall rule drops any DNS request (ports 53 and 853) from the TV that is not going to the firewall. Hard-coded DNS servers such as 8.8.8.8 time out. |
| **Private upstream DNS** | The firewall forwards lookups to Quad9 and Cloudflare instead of Google, and encrypts them with DNS over TLS so your ISP cannot read them. |
| **Full visibility** | Unbound Python mode logs every lookup and every block, per device, in the pfBlockerNG alerts page. |
| **Easy recovery** | If an app breaks, one click on the alert page whitelists the domain. |
| **Scoped to the TV only** | pfBlockerNG is applied to the IoT interface only at first. Your main LAN is untouched until you choose to extend it. |

## What equipment you need

### Firewall or router (required)

Any pfSense or pfSense Plus system. This guide was built on a Netgate 6100 running pfSense Plus 26.03. It also needs enough RAM for pfBlockerNG (a few GB is plenty for the lists used here).

### Switch (VLAN capable, only if you have one between the firewall and the access point)

**Yes, the switch matters, but the brand does not.** You need a managed switch that supports 802.1Q VLAN tagging and can carry tagged VLANs on the port that connects to the firewall and the port that connects to the access point (a "trunk"). A basic unmanaged switch is not a safe choice, because you cannot control how it handles tagged traffic.

Used here: a **UniFi Switch 8 PoE 150W (US 8 PoE 150W)**. UniFi switches carry all VLANs on every port by default, so no per-port change was needed. Any managed switch from another vendor works the same way, but you must set the trunk ports yourself.

If the access point plugs straight into a firewall port, you do not need a switch for this, but the access point must still support multiple SSIDs mapped to VLANs.

### Access point (required)

An access point that can broadcast more than one SSID and map each one to a VLAN ID. Used here: a **UniFi nano HD**, managed by a UniFi Network controller. Any business-class or prosumer access point with per-SSID VLAN support (UniFi, Omada, Aruba Instant On, Ruckus and others) works.

Consumer routers in access point mode usually cannot do this. Frankly, if you are tackling this, you already have this better class of equipment, as security and privacy are important to you.

### Controller

UniFi equipment needs a UniFi Network controller (Cloud Key, UniFi console, or self-hosted) to create the network and the SSID. Other vendors have their own management tool.

## What this does not do

- It does not stop a TV that connects to a hard-coded IP address with no DNS lookup.
- It does not stop DNS over HTTPS on port 443. The guide ends with a note on how to cover that.
- It only works as well as the blocklists. Check the alerts page after a day of normal use and add anything that slips through.
- It is a good idea to also turn off ACR and personalized ads in the TV's own settings. This guide is a second layer, not a replacement.

## How to use this guide

Work through the parts in order and finish each step before starting the next. The example addresses (VLAN 40, 10.40.40.0/24 and so on) can be replaced with your own, as long as you use them consistently.

---
# How to block Smart TV Telemetry with pfSense, UniFi and pfBlockerNG

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

**AI Usage: I could *never* have made this procedure so organized without Claude Code.**

## Appendix A: Using this guide with a non-LG TV

Parts 1 to 6 of this guide have nothing LG-specific in them. The VLAN, SSID, firewall rules, pfBlockerNG setup, and DNS bypass block work the same for any smart TV, streaming stick, console, or smart appliance. The only part that differs by brand is **what you block** (Part 7) and **what the device does when you block it**.

**Testing note.** The full procedure was built and tested on an LG webOS TV only. The brand notes below come from published blocklists and community reports, which are cited at the end, and have not been tested on those devices. Treat them as a starting point, and check the pfBlockerNG alerts page after a day of normal use.

### A.1 What stays the same

| Guide section | Applies to other brands? |
|---|---|
| Part 1: VLAN and interface | Yes, unchanged |
| Part 2: Firewall rules | Yes, unchanged |
| Part 3: UniFi network and SSID | Yes, unchanged |
| Part 4: Gateways and DNS | Yes, unchanged |
| Part 5: pfBlockerNG install and DNSBL | Yes, unchanged |
| Part 6: Close the DNS bypass | Yes, and more important on some brands (see A.4) |
| Part 7: Smart TV blocklist | Partly. Read A.2 and A.3 |

If you rename the DNSBL group in Part 7, call it `SmartTV` instead of `LG_SmartTV`.

### A.2 What the Perflyst SmartTV list covers

When this guide was written, the Perflyst list used in Part 7 had active sections for these vendors and platforms:

- Samsung, LG, Sony Bravia, Panasonic Viera, Philips, Toshiba, Hisense, Xiaomi
- Roku
- Android TV and Google TV (including Nvidia Shield), limited to a few Android TV specific Google hosts
- Netflix and Hulu telemetry hosts
- HbbTV (European broadcaster interactive TV) hosts
- A group of generic smart TV advertising hosts

It had **no active section for Vizio** (that section was present but commented out) and **no Amazon Fire TV section that I could find**. The list is community maintained, so open the raw file and check the section headers before relying on it for your brand.

### A.3 Per-brand notes

Block what is listed in the "Add this" column in a small custom DNSBL group, described in A.5. Where a brand is already covered by the Perflyst list, you may need nothing extra.

| Brand | Covered by Perflyst list? | Add this (DNSBL custom list) | What to watch for |
|---|---|---|---|
| **Samsung** (Tizen) | Yes. Includes `samsungacr.com`, `samsungads.com`, `samsungcloudsolution.com` hosts | Usually nothing | Uses hard-coded DNS (see A.4). Samsung's ACR and ad hosts are reported on `samsungacr.com` and related names |
| **LG** (webOS) | Yes | Nothing | Tested in this guide |
| **Vizio** (SmartCast) | No (inactive in the list) | `dis.inscape.tv` and `api.vizio.com` | `api.vizio.com` may serve more than ACR, so test SmartCast apps and whitelist if something breaks |
| **Roku** (including TCL and Hisense Roku TVs) | Yes. Includes `scribe.logs.roku.com` and other `*.logs.roku.com` hosts | Optionally `cooper.roku.com` | One community report says blocking Roku log hosts left the live TV program guide blank, with no confirmed fix. Roku appears to need to reach its own servers, so start with fewer blocks |
| **Amazon Fire TV** | No | A community Fire TV list exists (see sources), but it is unmaintained, and some of its entries are Alexa related | Blocking Alexa hosts breaks voice features. Community reports also show ad and tracking hosts being needed by some regional apps, such as German broadcaster media libraries, so whitelist as needed |
| **Google TV / Android TV** (Sony Bravia, Nvidia Shield, Chromecast with Google TV) | Partly (a few Android TV Google hosts) | Nothing safe to add blind | Google telemetry shares hosts with Google services, so broad blocking risks breaking the home screen and the Play Store. Use the TV's privacy settings for most of this |
| **Sony Bravia** | Yes (older Sony hosts, plus Google TV hosts on newer models) | Nothing | Newer Sony TVs run Google TV, so use the Google TV row too |
| **Hisense, Panasonic, Philips, Toshiba, Xiaomi** | Yes, each has a section | Nothing | None known |
| **Cross-platform** | Partly | `samba.tv` (TV viewing data broker used by several brands) | Samba TV appears in the Sony and Toshiba sections of the Perflyst list |

### A.4 Hard-coded DNS matters more on some brands

Some TVs ignore the DNS server you hand out and query public resolvers directly. One published lab test (not an independent audit) reports hard-coded resolvers on Samsung, Google TV (Sony Bravia), Fire TV, LG, Vizio and Roku, using ports 53, 443 and 853, with Roku the only one that port 53 redirection alone might handle.

Part 6 of this guide blocks ports 53 and 853 to everything except your firewall. If your logs show a TV still reaching public DNS over HTTPS on port 443, add a second alias and rule:

1. Firewall > Aliases > IP > Add. Name: `PublicDNS_Resolvers`. Add the public resolver addresses you want to deny (for example 8.8.8.8, 8.8.4.4, 1.1.1.1, 1.0.0.1, 9.9.9.9, 208.67.222.222, 208.67.220.220).
2. Firewall > Rules > IoT_TV > Add: Block, TCP, Source IoT_TV subnets, Destination `PublicDNS_Resolvers`, Port 443, Log checked. Place it with the other block rules, above the final pass rule.

The rule sits on the IoT_TV interface and matches only the TV's traffic, so the firewall's own lookups to its upstream resolvers are unaffected.

Also, DNS blocking does not stop telemetry that a TV sends to a hard-coded IP address or over a connection that is not DNS based. Turn off viewing data and personalized ads in the TV's own settings as well.

### A.5 Adding brand-specific domains

In pfBlockerNG: Firewall > pfBlockerNG > DNSBL > DNSBL Groups > Add

- Name: `Custom_TV`
- Description: `Brand specific TV telemetry`
- Action: Unbound
- Custom list box (label may vary slightly by version): paste the domains from A.3, one per line
- Save, then Update > Reload > DNSBL > Run

Check Reports > Alerts > DNSBL Python after a day. A domain that keeps getting blocked while an app misbehaves can be whitelisted with the `+` icon on that page.

### A.6 TV settings to change as a second layer

Menu paths vary by model and firmware, so treat these as starting points.

| Brand | Settings to turn off |
|---|---|
| Samsung | Settings > General & Privacy > Privacy Choices: turn off Viewing Information Services and Interest-Based Advertising |
| LG | Settings > General > Additional Settings: turn off Live Plus, and enable Limit Ad Tracking under Advertisement |
| Vizio | System > Reset & Admin: turn off Viewing Data, and enable Limit Ad Tracking under Advertising |
| Roku | Settings > Privacy > Smart TV Experience: turn off Use Info from TV Inputs, and enable Limit Ad Tracking under Advertising |
| Sony (Google TV) | Settings > Privacy: turn off Usage & Diagnostics, and opt out of Ads Personalization or delete the advertising ID |

### A.7 Wired TVs and other devices

- **Wired TV.** Skip Step 8 (the SSID). Put the switch port the TV is plugged into on VLAN 40 as an untagged (access) port. Everything else is the same.
- **Streaming sticks, consoles, smart appliances.** Join the same IoT_TV network. Add a device-specific blocklist only if the device is chatty, and expect to whitelist more on devices that depend on cloud services to work.

### Sources

- [Perflyst SmartTV blocklist (raw)](https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV.txt): vendor sections and domains
- [hkamran80 smart TV blocklist gist](https://gist.github.com/hkamran80/779019103fcd306979411d44c8d38459): Fire TV, Roku and Samsung hosts (marked unmaintained by its author)
- [How to stop smart TV spying (ACR tracking) 2026, privacysmarthome.com](https://www.privacysmarthome.com/guides/how-to-stop-smart-tv-spying-acr-tracking-2026/): Samsung, LG, Vizio, Roku domains and TV menu paths
- [How to block hardcoded DNS and DoH on smart TVs 2026, privacysmarthome.com](https://www.privacysmarthome.com/guides/how-to-block-hardcoded-dns-doh-on-smart-tvs-2026/): hard-coded resolver findings and firewall recommendations (its own lab testing)
- [Pi-hole forum: Roku TV schedule blocked](https://discourse.pi-hole.net/t/what-domain-to-whitelist-to-unblock-roku-tv-schedule/86284): Roku program guide breakage (unresolved)
- [Pi-hole forum: Fire TV Mediathek blocked](https://discourse.pi-hole.net/t/amazon-fire-tv-ard-pro7-mediathek-blockiert/3043): Fire TV app breakage and whitelisted domains
