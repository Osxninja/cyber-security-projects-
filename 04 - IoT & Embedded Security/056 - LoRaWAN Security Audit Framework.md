---
tags: [offensive-security, iot-security, btech-project, lorawan, lpwan, security-audit, key-compromise, radio-security]
category: "IoT & Embedded Security"
difficulty: "Advanced"
real_world_problem: "LoRaWAN long-range IoT security flaws leading to root key compromise, replay attacks, and network server impersonation"
tools: [ChirpStack, HackRF One, PyLoraWAN, Wireshark, Scapy]
estimated_duration: "5 weeks"
---

# 🎯 LoRaWAN Security Audit Framework
> **Category**: [[04 - IoT & Embedded Security]] | **Difficulty**: ⭐⭐⭐ | **Duration**: 5 weeks

---

## 📋 Problem Statement

> [!CAUTION] Real-World Impact
> Long Range Wide Area Network (LoRaWAN 1.0.x / 1.1) is the standard wireless protocol powering long-range IoT applications across smart agriculture, utility metering, asset tracking, and smart city sensors. Cryptographic key management flaws—such as hardcoded Application Keys (`AppKey`), static DevNonce reuse, and insecure Over-The-Air Activation (OTAA) handshakes—allow regional radio attackers to compromise session keys, forge uplinks, and inject malicious telemetry into cloud servers.

LoRaWAN networks span up to 15 kilometers using sub-GHz chirp spread spectrum (CSS) radio modulation. Security relies on two distinct AES-128 cryptographic keys: `NwkSKey` (Network Session Key for frame integrity) and `AppSKey` (Application Session Key for payload encryption).

In legacy LoRaWAN 1.0.x deployments, both session keys are derived directly from a single master `AppKey` during the OTAA Join Request / Join Accept exchange. If an attacker captures the initial join handshake or extracts the static `AppKey` from un-hardened node flash memory, they gain complete capability to decrypt all historical and future traffic, craft spoofed sensor readings, or initiate JoinNonce replay attacks to disconnect active sensors.

This project implements an automated LoRaWAN Security Audit Framework (LSAF). The framework integrates SDR radio sniffing, frame parsing, MAC command auditing, key derivation validation, and join request replay testing for enterprise LoRaWAN deployments.

### 🌍 Real-World Incidents
- **Smart City Water Meter Key Compromise (2020)**: Security audits of municipal LoRaWAN smart water meters revealed identical default `AppKey` values hardcoded across 20,000 active meters, allowing full network traffic decryption.
- **LoRaWAN Replay & ACK Injection Vulnerability (2021)**: Researchers demonstrated replaying captured uplink frames on LoRaWAN 1.0.2 networks due to missing frame counter checks on Join Accept frames, resulting in denial-of-service for utility monitoring.
- **Agricultural LoRa Gateway Impersonation (2022)**: Rogue gateway injection attacks intercepted and dropped sensor telemetry from remote agricultural soil monitors, triggering automated crop irrigation failures.

---

## 🔬 Research Paper References

| # | Paper Title | Authors | Year | Source | Key Contribution |
|---|-------------|---------|------|--------|-----------------|
| 1 | Security of LoRaWAN v1.1 in Backward Compatible Deployments | Aras et al. | 2018 | ACM Conference on Security and Privacy in Wireless and Mobile Networks | Comprehensive security evaluation of OTAA activation flaws and nonce replay risks |
| 2 | Systematic Key Reuse Analysis in LoRaWAN Ecosystems | Eldefrawy et al. | 2020 | IEEE Transactions on Information Forensics and Security | Empirical study discovering widespread static AppKey reuse in commercial LPWAN deployments |
| 3 | Practical Bit-Flipping and Replay Attacks on LoRaWAN | Noura et al. | 2021 | IEEE Internet of Things Journal | Demonstration of payload tampering and MAC command injection vulnerabilities in LoRaWAN 1.0.3 |

---

## 🏗️ System Architecture
Link: [[Drawing 2026-07-30 22.31.19.excalidraw#Project 056: 056 - LoRaWAN Security Audit Framework|Excalidraw Architecture Diagram]]


```mermaid
graph TD
    subgraph LoRaWAN Field RF Environment
        ED1[LoRaWAN End Device - Utility Meter] -->|868/915 MHz CSS RF Uplink| GW[LoRaWAN Gateway - Concentratd]
        ED2[LoRaWAN Sensor - Tracking Node] -->|Join Request Frame| GW
    end

    subgraph RF Capture & SDR Interception Layer
        GW -->|LoRa Packet Stream| SDR[Software Defined Radio / HackRF One / SX1276 Module]
        SDR -->|Raw Baseband / UDP Semtech Packets| PARSER[LoRaWAN Packet Dissector & Parser]
    end

    subgraph Security Audit Engine - LSAF Core
        PARSER --> M1[OTAA Join Handshake Auditor]
        PARSER --> M2[DevNonce Reuse Detector]
        PARSER --> M3[AES-128 Key Derivation & Decryptor]
        PARSER --> M4[Frame Counter & Replay Engine]
        PARSER --> M5[MAC Command Injection Tester]
    end

    subgraph Core Network & Output
        M1 --> NS[ChirpStack Network Server - Testbed]
        M3 -->|Decrypted Payload| OUT1[Decrypted Application Data Feed]
        M4 -->|Replay Finding| OUT2[Security Audit Dashboard & PDF Report]
        M5 --> OUT2
    end
```

---

## 📐 Technical Implementation

### Phase 1: Testbed Setup & RF Infrastructure (Week 1)
- Deploy an isolated LoRaWAN network using `ChirpStack` open-source Network Server running on Docker.
- Setup hardware endpoints: ESP32 with SX1276 LoRa transceiver module (End Device) and a Dragino / RAK Wireless LoRaWAN Gateway.
- Install software stack: `pyLoraWAN`, `scapy`, `GNU Radio`, `wireshark` with LoRaWAN plugin, `hackrf`.

### Phase 2: Frame Dissection & Key Derivation Parser (Weeks 2-3)
- Build a Python LoRaWAN frame parser decoding physical layer MHDR (MAC Header), DevAddr, FCtrl, FCnt, FPort, and MIC (Message Integrity Code):
  - **Join Request Frame Parsing**: Extract $AppEUI$, $DevEUI$, and $DevNonce$.
  - **Join Accept Frame Parsing**: Extract $AppNonce$, $NetID$, $DevAddr$, $DLSettings$, $RxDelay$.
- Implement AES-128 Session Key Derivation matching LoRaWAN 1.0.x spec:
  $$NwkSKey = \text{AES-128}_{AppKey}(0x01 \parallel AppNonce \parallel NetID \parallel DevNonce \parallel pad_{16})$$
  $$AppSKey = \text{AES-128}_{AppKey}(0x02 \parallel AppNonce \parallel NetID \parallel DevNonce \parallel pad_{16})$$

### Phase 3: Attack Execution Modules (Weeks 4-5)
- **Module 1 - DevNonce Reuse & Replay Engine**:
  - Monitors Join Requests for repeated $DevNonce$ values. Replays historical Join Requests to evaluate whether the Network Server enforces nonce tracking or allows session key reset.
- **Module 2 - MIC Bit-Flipping & Tamper Auditor**:
  - Tests MIC validation rigor by altering unencrypted payload bytes and recalculating AES-CMAC integrity tags.
- **Module 3 - MAC Command Injection Tester**:
  - Injects unauthenticated piggybacked MAC commands (e.g., `LinkADRReq` to degrade node radio transmission power or change channel frequencies) into unencrypted downlink streams.

### Phase 4: Reporting & LoRaWAN 1.1 Hardening Guide (Week 5)
- Compile automated JSON and PDF reports categorizing findings against the LoRa Alliance Security Guidelines.
- Formulate a migration plan detailing upgrading nodes to LoRaWAN 1.1 (enforcing `NwkKEY` / `AppKEY` separation and Join Server hardware security modules).

---

## 🔧 Tools & Technologies

| Tool | Purpose | Alternative |
|------|---------|-------------|
| **ChirpStack** | Open-source LoRaWAN Network Server for local testbed hosting | The Things Network (TTN) |
| **PyLoraWAN / Python-Crypto** | Python packet decoding and AES-128 key derivation library | Scapy-LoRaWAN |
| **HackRF One / SX1276** | SDR radio hardware for sub-GHz Chirp Spread Spectrum packet capture | LimeSDR / RTL-SDR |
| **Wireshark (LoRaWAN Plugin)**| Graphical packet dissector for PHY payload and MAC header analysis | TShark |
| **ChirpStack Gateway Bridge** | UDP to MQTT Semtech packet forwarder translator | Semtech Packet Forwarder |

---

## 💡 Key Features
- ✅ **Automated Key Derivation Engine**: Calculates `NwkSKey` and `AppSKey` automatically given a candidate master `AppKey`.
- ✅ **DevNonce Tracking & Replay Evaluator**: Identifies non-compliant end devices generating weak or sequential nonces.
- ✅ **Sub-GHz Radio Sniffing Integration**: Connects via SDR or Semtech UDP packet forwarder sockets to intercept raw radio frames.
- ✅ **MAC Command Tamper Detection**: Audits Network Server handling of piggybacked link management commands (`DutyCycleReq`, `DevStatusReq`).
- ✅ **LoRaWAN 1.0 vs 1.1 Compatibility Validator**: Differentiates protocol version features and highlights legacy fallback vulnerabilities.

---

## 📊 Expected Results

> [!NOTE] Deliverables
> Complete Python LoRaWAN audit toolkit, ChirpStack testbed configuration files, raw PCAP sample captures, and security hardening guide.

### Performance Metrics
- **Packet Decoding Speed**: Sub-10ms frame parsing and MIC integrity verification per uplink.
- **Key Derivation Accuracy**: 100% match with official LoRa Alliance test vectors.
- **Replay Detection Rate**: 100% identification of repeated DevNonce instances.

### Output Artifacts
1. `lorawan_audit_tool.py`: Core CLI audit runner.
2. `key_derivation_engine.py`: Cryptographic key calculator module.
3. `lorawan_security_report.pdf`: Generated assessment audit output.

---

## 🎓 Learning Outcomes
1. 📚 **LPWAN Radio Protocols**: Master Chirp Spread Spectrum (CSS) modulation, spreading factors (SF7-SF12), bandwidth, and sub-GHz frequencies.
2. 📚 **LoRaWAN Cryptographic Architecture**: Deep understanding of AES-CMAC integrity checks, AES-CTR payload encryption, and dual session key derivation.
3. 📚 **OTAA vs ABP Activation**: Evaluating security trade-offs between Over-The-Air Activation and Activation By Personalization.
4. 📚 **LPWAN Threat Modeling**: Mitigating replay attacks, gateway spoofing, and key exposure risks in long-range IoT networks.

---

## ⚠️ Ethical Considerations
> [!WARNING] Legal & Ethical Notice
> Sub-GHz ISM radio bands (868 MHz EU / 915 MHz US) are shared spectrum regulated by regional authorities. All RF transmissions and test replays must strictly comply with duty-cycle limits and be performed within authorized private test environments.

---

## 🔗 Related Projects
- [[050 - Zigbee & Z-Wave Protocol Attack Simulator]]
- [[052 - BLE Sniffing & MITM Attack Tool]]
- [[053 - Industrial SCADA-ICS Security Assessment Platform]]

---
*📅 Created: 2026-07-30 | 🏷️ Category: IoT & Embedded Security | 🔐 Offensive Security Research*
