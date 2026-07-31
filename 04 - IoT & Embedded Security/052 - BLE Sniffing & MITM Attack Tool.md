---
tags: [offensive-security, iot-security, btech-project, ble, bluetooth-low-energy, mitm, sniffing, medical-device-security]
category: "IoT & Embedded Security"
difficulty: "Advanced"
real_world_problem: "BLE protocol vulnerabilities in medical devices and fitness trackers enabling eavesdropping and unauthorized data manipulation"
tools: [Ubertooth One, Bettercap, Scapy, Wireshark, GATTtool]
estimated_duration: "6 weeks"
---

# 🎯 BLE Sniffing & MITM Attack Tool
> **Category**: [[04 - IoT & Embedded Security]] | **Difficulty**: ⭐⭐⭐ | **Duration**: 6 weeks

---

## 📋 Problem Statement

> [!CAUTION] Real-World Impact
> Bluetooth Low Energy (BLE 4.x / 5.x) is the dominant short-range wireless standard used by wearable health monitors, insulin pumps, smart glucose meters, and door access tokens. Implementation flaws—such as "Just Works" pairing without authentication, missing link-layer encryption, and static MAC address tracking—expose sensitive health data to passive sniffing and allow active Man-In-The-Middle (MITM) manipulation of critical device settings.

Unlike classic Bluetooth, BLE relies on Generic Access Profile (GAP) for device discovery and Generic Attribute Profile (Gatt) for data transfer organized into Services and Characteristics. Many medical and IoT manufacturers skip pairing entirely or deploy legacy "Just Works" unauthenticated Diffie-Hellman key exchange, which is vulnerable to active interceptors.

An attacker positioned within radio range can passively sniff BLE advertisement and connection packets, or deploy a rogue dual-role proxy (spoofing both peripheral and central roles) to manipulate sensor readings (such as altering reported blood glucose concentrations) before relaying packets to mobile healthcare applications.

This project develops an advanced BLE Sniffing & Man-In-The-Middle (MITM) Attack Framework using Ubertooth One hardware and Bettercap software. The tool demonstrates passive advertisement capturing, active GATT service cloning, credential interception, and dynamic packet manipulation on live BLE connection links.

### 🌍 Real-World Incidents
- **Insulin Pump BLE Command Injection (2019-2020)**: Security advisories revealed unauthenticated BLE control interfaces on commercial insulin pumps allowing unauthorized remote delivery of lethal insulin doses.
- **BLE Smart Glucose Meter Telemetry Spoofing (2021)**: Researchers demonstrated intercepting BLE traffic from wearable continuous glucose monitors, modifying blood sugar telemetry values in real time during MITM proxying.
- **BLE Smart Lock Relay Attacks (2022)**: Commercial automotive and residential BLE keyless entry systems were compromised using low-latency BLE proxy relays, unlocking vehicles while owner keys were far away.

---

## 🔬 Research Paper References

| # | Paper Title | Authors | Year | Source | Key Contribution |
|---|-------------|---------|------|--------|-----------------|
| 1 | BLEDSA: Security Analysis of Bluetooth Low Energy Pairing Implementations | Wu et al. | 2020 | USENIX Security | Uncovering logic vulnerabilities and MITM flaws across commercial BLE protocol stacks |
| 2 | Practical BLE Sniffing and MITM Attacks on IoT Health Devices | Sun et al. | 2021 | IEEE Transactions on Information Forensics and Security | Empirical demonstration of real-time payload modification in connected medical sensor streams |
| 3 | BLUFFS: Bluetooth Forward and Future Secrecy Attacks | Antonioli et al. | 2023 | ACM CCS | Discovery of fundamental architectural flaws in Bluetooth key derivation allowing session key impersonation |

---

## 🏗️ System Architecture
Link: [[Drawing 2026-07-30 22.31.19.excalidraw#Project 052: 052 - BLE Sniffing & MITM Attack Tool|Excalidraw Architecture Diagram]]


```mermaid
graph TD
    subgraph Target BLE Ecosystem
        PER[Target BLE Peripheral - Medical Sensor] <-->|Original BLE Connection| CEN[Target BLE Central - Mobile App]
    end

    subgraph RF Hardware Layer
        U1[Ubertooth One - Passive Sniffer] -->|Channel Hopping Captures| P1[Raw PCAP Stream]
        B1[Bluetooth 5.0 Adapter 1 - hci0] -->|Active Proxy Socket| MITM[Bettercap / Python MITM Core]
        B2[Bluetooth 5.0 Adapter 2 - hci1] -->|Active Proxy Socket| MITM
    end

    subgraph MITM Proxy & Manipulation Engine
        MITM --> M1[Peripheral GATT Cloner & Spoofer]
        MITM --> M2[Central Connection Hijacker]
        M1 <-->|Intercepted BLE Packets| M3[Dynamic Payload Mutator Engine]
        M2 <-->|Intercepted BLE Packets| M3
    end

    subgraph Analysis & Telemetry Output
        P1 --> OUT1[Wireshark Packet Dissector]
        M3 -->|Log Modified GATT Values| OUT2[CLI Dashboard & Telemetry Logger]
    end
```

---

## 📐 Technical Implementation

### Phase 1: Hardware Setup & BLE Environment (Week 1)
- Setup Linux testing host equipped with Ubertooth One hardware sniffer and two CSR 4.0 / BLE 5.0 USB Bluetooth dongles (`hci0`, `hci1`).
- Install dependencies: `ubertooth`, `kismet`, `bettercap`, `bluez`, `gatttool`, `python-bleak`, `scapy`.
- Configure test BLE peripheral target (e.g., ESP32 running GATT health thermometer service or Nordic nRF52 dev board).

### Phase 2: Passive RF Sniffing & Channel Tracking (Weeks 2-3)
- Build passive packet collection pipeline using `ubertooth-rx` and `ubertooth-btle`:
  - Sniffs advertisement channels (37, 38, 39) to detect device MAC addresses and Advertising Data (AD) flags.
  - Follows connection requests (`CONNECT_REQ`) and tracks adaptive frequency hopping (37 data channels) using CRC initialization and Access Address parameters.
- Pipe captured raw BLE Link Layer frames directly into Wireshark via named pipes (`/tmp/pipe`) for packet dissection.

### Phase 3: Active GATT Enumeration & MITM Proxy (Weeks 4-5)
- **GATT Service Enumerator**:
  - Connects to target peripheral using `gatttool` / `bleak` to clone all Primary Services, Characteristics, Descriptors, and UUIDs.
- **Rogue Peripheral & Central Proxy (Bettercap)**:
  - Adapter `hci0` advertises cloned GATT profile to impersonate the target medical device to the mobile application.
  - Adapter `hci1` connects to the real physical medical device as a central client.
  - Transparently bridges connection packets while maintaining dual GATT sockets.

### Phase 4: Dynamic Payload Manipulation & Hardening (Week 6)
- **Payload Mutator Engine**:
  - Inspects incoming GATT Read/Write/Notification frames in Python.
  - Implements dynamic rule evaluation: e.g., if GATT Characteristic UUID == `0x2A37` (Heart Rate Measurement) or `0x2A18` (Glucose Measurement), byte values are modified in-flight before forwarding.
- **Remediation Plan**:
  - Formulate guidelines for implementing BLE Passkey / Out-Of-Band (OOB) pairing, AES-CCM link encryption, and application-layer payload signing.

---

## 🔧 Tools & Technologies

| Tool | Purpose | Alternative |
|------|---------|-------------|
| **Ubertooth One** | Hardware 2.4 GHz wireless development platform for BLE sniffing | Nordic nRF52840 Dongle / Ellisys |
| **Bettercap** | Modular framework for BLE reconnaissance and active MITM proxying | BLESuite / GATTack |
| **BlueZ (hcitool / gatttool)**| Official Linux Bluetooth protocol stack | Bleak / Noble |
| **Wireshark** | Packet dissector analyzing BLE Link Layer and ATT/GATT protocols | TShark |
| **ESP32 / nRF52** | Hardware targets for building benign medical device emulators | Arduino BLE |

---

## 💡 Key Features
- ✅ **Passive Channel Hopping Tracking**: Follows active BLE connections across all 37 data channels by calculating Access Addresses and hop intervals.
- ✅ **Automated GATT Profile Cloning**: Dynamically mirrors all UUIDs, properties, and handles of target peripherals in under 10 seconds.
- ✅ **In-Flight Payload Mutator**: Real-time modification of notification and write payloads using custom regex and byte-array rules.
- ✅ **Dual-Adapter Transparent Proxy**: Complete separation of central and peripheral radios ensuring low-latency packet relay.
- ✅ **Wireshark Live Stream Integration**: Pipe hardware-level RF traffic directly into Wireshark interface for protocol dissection.

---

## 📊 Expected Results

> [!NOTE] Deliverables
> Python MITM proxy framework, Bettercap BLE attack scripts, Ubertooth capture helper daemons, and medical device hardening guide.

### Performance Metrics
- **Sniffing Hop Reliability**: $\ge 95\%$ packet capture rate on active BLE connection hopping.
- **GATT Cloning Speed**: Sub-10 seconds to discover and replicate full GATT database.
- **MITM Relay Latency**: $< 20 \text{ ms}$ latency added during active packet modification.

### Output Artifacts
1. `ble_mitm_proxy.py`: Python script managing dual HCI sockets and payload mutations.
2. `ubertooth_pcap_stream.sh`: Automation script piping raw RF frames into Wireshark.
3. `cloned_gatt_profile.json`: Captured GATT structure database file.

---

## 🎓 Learning Outcomes
1. 📚 **BLE Protocol Stack**: Deep understanding of GAP, GATT, ATT, SMP (Security Manager Protocol), and Link Layer packet structures.
2. 📚 **Radio Packet Tracking**: Tracking adaptive frequency hopping, access addresses, and connection parameters over the air.
3. 📚 **Active MITM Architectures**: Constructing dual-role wireless proxies for payload interception and alteration.
4. 📚 **Healthcare IoT Defense**: Hardening embedded health monitors against unauthenticated pairing and plain-text telemetry risks.

---

## ⚠️ Ethical Considerations
> [!WARNING] Legal & Ethical Notice
> Intercepting or modifying wireless health monitor communications poses direct physical risks. Never execute BLE MITM attacks against active medical hardware used by patients. Conduct all tests on isolated lab hardware.

---

## 🔗 Related Projects
- [[047 - Smart Home Device Vulnerability Assessment Framework]]
- [[050 - Zigbee & Z-Wave Protocol Attack Simulator]]
- [[057 - Embedded Device Side-Channel Attack Demonstrator]]

---
*📅 Created: 2026-07-30 | 🏷️ Category: IoT & Embedded Security | 🔐 Offensive Security Research*
