# Ultraloq Bolt (Wi‑Fi) intermittent cloud disconnects on ASUS AiMesh

Observed October 2026. Not an API topic, but the same lock models; kept here because the symptom ("lock online, cloud disconnected, updates fail") is easy to mistake for an API or account problem.

**Summary:** An Ultraloq Bolt Wi‑Fi lock associates to the network and obtains an IP address, but the app shows *Cloud Status: Disconnected* at irregular intervals and firmware updates fail. The cause is **client roaming between AiMesh access points**. Binding the lock to a single access point resolves it.

If you have the same symptom on a mesh network, start with [the roaming check](#roaming-check).

---

## Environment

- Lock: Ultraloq **Bolt Fingerprint, Wi‑Fi version** (`BOLT‑F‑BK`), Xthings Home / U‑tec app
- Router: ASUS WiFi 7 router with one **AiMesh node**, Asuswrt 3.0.0.6 (Guest Network Pro / SDN)
- Lock placed on the **Default IoT Network**: 2.4 GHz only, WPA2‑Personal/AES, WiFi 7 mode off, SSID visible, AP isolation off, same subnet as the main LAN
- Other 2.4 GHz IoT devices on the same SSID operate normally

## Symptom

| App | Router |
|---|---|
| Wi‑Fi Signal **Good (‑40 dB)**, Cloud Status **Disconnected** | Lock authenticates and associates; DHCP lease issued; RSSI ‑53 dBm |
| Shortly after: Wi‑Fi Signal **Fail (0 dB)** | Syslog: `Disassoc … sending station is leaving (8)` and `Deauth_ind … leaving (3)` |
| Firmware update fails | — |
| On a phone hotspot: Cloud **Connected** and stable | — |

The interval between join and leave varied from roughly one to thirteen minutes.

## Root cause

The lock roamed between the main router and the AiMesh node. The node was the stronger signal at the door (‑40 dBm vs ‑53 dBm from the router), and the lock's Wi‑Fi stack re‑evaluated its association repeatedly. Each roam:

1. ends the association with one AP (logged as "station is leaving"),
2. begins a new association with the other AP (not visible in the main router's syslog, which only records its own radios),
3. resets the lock's cloud session, reported in the app as *Cloud Disconnected* until the session is re‑established.

Firmware updates on this model are delivered over Wi‑Fi, so they fail for the same reason.

## Resolution

In the ASUS router UI or app: **Network Map → the lock → AiMesh binding → select a specific node.** Choose the AP the lock associates with most often. Reboot the router, then the node.

After binding, the router syslog shows a single Auth/Assoc for the lock with no further disconnects, and Cloud Status remains Connected with all security and QoS features enabled.

## Roaming check

For any mesh client with good signal, a valid IP address and an intermittent cloud or app session:

1. In the router's client list, note which AP the device is attached to.
2. Check again after five minutes.
3. If the AP changed, bind the device to one AP.

In the router syslog, repeated `reason: … sending station is leaving (8)` / `(3)` entries at good RSSI, with intervals where the device does not appear at all, indicate the same thing: the device is alternating between access points, and the gaps are time spent on the other AP.

## Additional checks

These were verified during diagnosis and are listed for completeness. They did not affect the result in this case, but are reasonable to confirm on an ASUS router if the roaming check is negative:

- AiProtection (Malicious Sites Blocking, Two‑Way IPS)
- Firewall and DoS protection
- Parental Controls, QoS
- VPN servers, Instant Guard, VPN assignment on the IoT network (Guest Network Pro → VPN tab)
- Access Intranet and AP isolation on the IoT network
- DNS settings (DNS Filter, DNS‑over‑TLS), IPv6

Note on partial restarts: changing Guest Network Pro settings triggers `restart_wireless; restart_sdn`, which can leave an SDN interface in an inconsistent state until a full reboot. If a client begins failing at Auth or the 4‑way handshake immediately after a settings change, reboot before drawing conclusions.

## Observations about the lock's Wi‑Fi stack

Not the cause in this case, but relevant when configuring the SSID. The lock's radio is 2.4 GHz 802.11 b/g/n and does not attempt to join networks it cannot use, so the router log shows no auth attempts at all:

- WPA3 or WPA2/WPA3 mixed mode with PMF required
- WiFi 7 / MLO enabled on the SSID
- Hidden SSID
- Smart Connect / band‑steered SSID (the app cannot offer it)

U‑tec's documentation additionally recommends a password under 32 characters without `' * / \`, and users report better stability at a fixed 20 MHz channel width.

ASUS creates the *Default IoT Network* with WiFi 7 off and WPA2 for this reason.

## Checklist for battery‑powered IoT devices on AiMesh

- [ ] 2.4 GHz‑only SSID, WPA2‑Personal/AES, PMF off, WiFi 7 mode off
- [ ] Fixed 20 MHz channel (1, 6 or 11); review 2.4 GHz channel utilization
- [ ] SSID visible; password within the vendor's length and character limits
- [ ] Bind the device to the nearest AP when it is first added
- [ ] Leave Roaming Assistant for phones and laptops

## Further diagnostics

If the router runs Asuswrt‑Merlin with Entware, `tcpdump` on the router shows the device's cloud session directly — DNS queries, destination hosts and ports, and TLS setup — and makes session resets at each roam visible in real time. On stock firmware, the syslog association events and the client list are the available signals.
