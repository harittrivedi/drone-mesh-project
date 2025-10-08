# Drone Telemetry + RoIP Encrypted Mesh

This repository contains a concise summary of the project and step‑by‑step build instructions for creating a resilient mesh network for **Unmanned Aerial Vehicles (UAVs)**.  The goal is to deliver secure telemetry and radio‑over‑IP (RoIP) voice streams over a self‑healing wireless mesh using off‑the‑shelf hardware and open‑source software.

## Project Summary

**Mission:** build a secure, long‑range communication system for drones that can recover from jamming or interference.  The system should:

- Carry drone telemetry data and RoIP voice streams over a **5.8 GHz** Wi‑Fi 6 back‑haul.
- Provide a **failsafe control path** on the **865–868 MHz** (802.11ah HaLow) band.
- Employ **anti‑jamming detection** and **channel agility**, so the mesh can switch to a clean channel without losing connectivity.
- Maintain end‑to‑end privacy using **WireGuard** encryption, even during channel hops.

**Architecture:**

| Component          | Description                                                           |
|--------------------|-----------------------------------------------------------------------|
| **Nodes**          | 2–3 units (ground station ↔ UAV ↔ optional relay node)                |
| **Hardware**       | IPQ95xx single board computer, QCN9074 PCIe radio (5.8 GHz), optional RTL‑SDR for spectrum scanning, optional 802.11ah HaLow module (865–868 MHz) |
| **Software stack** | OpenWrt/QSDK, `wpad‑mesh‑openssl` for 802.11s mesh, `batman‑adv` for layer‑2 routing, `wireguard‑tools` for encryption |

**Anti‑jamming strategy:**

1. **Detect interference** using metrics like clear channel assessment (CCA) busy time, noise floor, packet error rate and retry statistics.  Optionally monitor wideband spectrum with an RTL‑SDR.
2. **Decide** when a channel is jammed and select the next frequency from a preconfigured hop‑set.
3. **Mitigate** by issuing an 802.11h channel switch announcement.  Sessions continue over WireGuard without dropping packets.  An optional HaLow beacon can broadcast the new channel to all nodes.

**Minimal viable product (MVP):** form a 5.8 GHz mesh, bridge it with BATMAN, layer WireGuard encryption on top, stream RoIP and telemetry, run a **jam‑watch** daemon to collect interference metrics, perform manual and automatic channel hops, and log packet loss, latency and recovery times.

## Build Instructions

These instructions are written for readers with a basic understanding of RF concepts and Linux system administration.  They walk you through building a proof‑of‑concept mesh network from scratch.

### 1. Gather hardware

1. **Compute platform:** an IPQ95xx or similar single board computer that supports OpenWrt/QSDK.  Ensure it has at least one mini‑PCIe slot for the Wi‑Fi 6 radio and a free USB port if you plan to add an RTL‑SDR.
2. **Radios:**  
   - **Back‑haul:** a QCN9074 or other Wi‑Fi 6 (802.11ax) radio card covering the 5.825–5.875 GHz band.  Use 4 × 4 MIMO antennas rated for the legal EIRP in your region (e.g. India).  
   - **Optional control link:** an 802.11ah HaLow module operating in the 865–868 MHz band with a suitable antenna.  This channel is limited in transmit power and duty cycle, so use it for low‑bit‑rate control only.  
   - **Spectrum scanner:** an inexpensive RTL‑SDR USB dongle for wideband scanning and interference detection (optional but highly recommended).
3. **Power and enclosures:** flight‑ready battery packs and ruggedized enclosures if you intend to mount the hardware on a UAV.

### 2. Flash and configure OpenWrt

1. Download the appropriate OpenWrt image (or QSDK build) for your SBC.  Follow the manufacturer’s guide to flash the firmware.  
2. Once the board boots, connect via SSH.  Update the package feeds and install the necessary packages:

   ```
   opkg update
   opkg install wpad‑mesh‑openssl batman‑adv wireguard‑tools
   # If using an RTL‑SDR: opkg install rtl‑sdr rtlsdr‑scanner
   # If using HaLow: install drivers provided by the vendor
   ```

3. Enable the 802.11s mesh on the 5.8 GHz radio.  In `/etc/config/wireless` define an interface with `mode 'mesh'`, set the SSID (e.g. `mesh‑backhaul`), channel (e.g. 149), and enable SAE for authentication.  Example:

   ```
   config wifi‑iface 'mesh0'
       option device      'radio0'
       option mode        'mesh'
       option mesh_id     'mesh‑backhaul'
       option channel     '149'
       option encryption  'sae'
       option key         'YourStrongPassphrase'
   ```

4. Bring up the mesh interface and verify neighbours with `iw mesh0 station dump`.

5. Load the **batman‑adv** kernel module and create a bat0 interface:

   ```
   modprobe batman_adv
   batctl if add mesh0
   ip link set up dev bat0
   ```

   Assign IP addresses to `bat0` on all nodes (e.g. 10.0.0.x/24) and verify connectivity with ping.

### 3. Add encryption and services

1. Generate WireGuard keys on each node:

   ```
   wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
   ```

2. Create `/etc/wireguard/wg0.conf` with your peers’ public keys, endpoints and allowed IP ranges.  Bind WireGuard to the `bat0` interface.  Start and enable the service:

   ```
   wg‑quick up wg0
   /etc/init.d/network reload
   ```

3. Set up RoIP by installing a software codec (e.g. `mumble‑server` or `freepbx`) or by bridging analogue radios through an audio interface.  Route the voice traffic through the WireGuard tunnel.

4. Deploy your drone telemetry service (e.g. MAVLink) over the mesh.  Test latency and throughput.

### 4. Implement anti‑jamming measures

1. Write a `jam‑watch` daemon (shell or Python) that periodically samples:
   - Channel busy time via `iw dev mesh0 survey dump`  
   - Packet error and retry statistics via `iw dev mesh0 station dump`  
   - Noise floor via the hardware counters.  
   - Optional wideband scans via an RTL‑SDR (`rtl_power`) to detect continuous wave interference.
2. Compare metrics to baseline thresholds; if interference persists beyond a defined window, mark the channel as jammed.
3. Select the next channel from your hop‑set (e.g. 149 → 153 → 157 → 161) and instruct the mesh interface to switch:

   ```
   iw dev mesh0 set channel 153
   ```

4. Use the 802.11h Channel Switch Announcement (CSA) mechanism to notify all peers.  If using HaLow, broadcast the new channel number over the low‑band link to ensure every node changes together.

5. Verify that the WireGuard session persists and that telemetry and RoIP traffic flow resumes on the new channel.  Log the recovery time and any packet loss.

### 5. Test and iterate

1. Start with a simple two‑node setup on a bench.  Confirm mesh formation, encryption, and service functionality.
2. Introduce artificial interference (e.g. a Wi‑Fi jammer or signal generator) and validate that the `jam‑watch` daemon detects it and triggers a channel switch.
3. Add a third node (a relay) and repeat the tests at greater distances.  Measure throughput, latency, and packet error rates on different channels.
4. Document your findings and tune the thresholds, hop‑set, and daemon logic for reliable performance.

## Notes on regulatory compliance

- In India, the 5.825–5.875 GHz band is unlicensed but subject to EIRP limits and Dynamic Frequency Selection (DFS).  Consult the latest **WPC** guidelines and adjust transmit power accordingly.  
- The 865–868 MHz HaLow band has strict duty‑cycle restrictions; use it sparingly for control signals, not for high‑bandwidth streaming.  
- Always ensure that your antenna and radio certifications comply with local regulations before flying.

## Disclaimer

Thrisdiction.  Please operate responsibly and comply with all applicable laws and safety guidelines.ese instructions are provided for educational and experimental purposes.  Building and operating wireless communication equipment leasemay be regulated in your jurisdiction.  P operate responsibly and comply with all applicable laws and safety guidelin
These instructions are provided for educational and experimental purposes.  Building and operating wireless communication equipment may be regulated in your jurisdiction.  Please operate responsibly and comply with all applicable laws and safety guidelines.es.
