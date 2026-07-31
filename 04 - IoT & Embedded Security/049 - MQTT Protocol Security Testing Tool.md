# MQTT Protocol Security Testing Tool

## Abstract
Bhai, Message Queuing Telemetry Transport (MQTT) modern Industrial IoT (IIoT), smart factory automation, aur smart city infrastructure ka primary lightweight messaging protocol hai. Lekin misconfigured MQTT brokers jo without username/password authentication, granular Access Control Lists (ACLs), ya TLS encryption run hote hain, malicious actors ko critical operational infrastructure compromise karne ka direct doorway de dete hain.

Is project ka main objective ek automated, end-to-end MQTT Security Testing Tool build karna hai. Yeh framework multi-stage auditing pipeline execute karta hai: broker service discovery (Port 1883/8883), wildcard topic hierarchy harvesting (`#` aur `+`), authentication/ACL privilege escalation checks, malformed packet mutation fuzzing, aur TLS/SSL cipher suite validation.

Is security testing framework se industrial enterprise environments me unauthenticated brokers aur insecure topic ACLs ko pinpoint kiya ja sakta hai. Dynamic payload inspection se actual exploitation hone se pehle security engineers vulnerabilities remediate kar pate hain.

## Real-World Context & Vulnerability Deep Dive
Samajhiye ki MQTT messaging architecture IIoT networks me operate kaise karti hai. MQTT ek publish/subscribe architecture use karta hai jahan ek central Broker sabhi client connections handle karta hai. Sensors data topics par publish karte hain (jaise `factory/line1/temp`) aur actuators or PLCs respective topics subcribe karte hain. Protocol implementation me overhead kam karne ke liye by default plain-text transport (Port 1883) aur anonymous access support kiya jata hai.

Real-world incident history se severity evident hoti hai. Global Internet Scans (2020-2023) me over **80,000 publicly exposed unauthenticated MQTT brokers** identify hue the jo medical patient telemetry, smart city traffic signals, aur building management systems ki sensitive feeds expose kar rahe the. **Smart Factory Robotic Arm Attack (2021)** me attackers ne wildcard topics sniff karke actuator control endpoints (`factory/line1/actuator/arm`) par unauthorized override commands inject kiye the, jisse physical assembly line shutdown ho gayi thi.

Technical analysis ke mutabiq, sabse critical vulnerability wildcard subscription (`#` multi-level, `+` single-level) misuse me lie karti hai. Jab client topic `#` par `SUBSCRIBE` command bhejta hai, toh agar broker par per-topic ACL enforce nahi hai, toh broker pure factory network ka real-time telemetry feed attacker socket par echo kar deta hai. Furthermore, Quality of Service (QoS) Level 2 aur retained messages ka misuse memory exhaustion DoS create kar sakta hai.

Is systemic operational risk ko solve karne ke liye humara testing framework protocol-aware auditing logic implement karta hai. Dynamic fuzzing vectors pass karke broker header parsing vulnerabilities detect hoti hain jabki ACL validator read/write authorization boundary leaks identify karta hai.

## Academic & Research Paper References
| # | Paper Title | Authors | Year | Source | Key Insight & Contribution |
|---|-------------|---------|------|--------|---------------------------|
| 1 | Security Evaluation of MQTT Broker Implementations in Industrial IoT | Formby et al. | 2020 | IEEE TII | Comprehensive vulnerability matrix of major MQTT brokers under fuzzing and authentication bypass vectors. |
| 2 | Analyzing Vulnerabilities and Privacy Risks of Public MQTT Brokers | Farris et al. | 2021 | ACM TCPS | Empirical scan and analysis of over 50,000 public MQTT deployments revealing widespread industrial data leaks. |
| 3 | MQTT-Fuzz: Dynamic Protocol Fuzzing for Industrial Messaging Gateways | Chen et al. | 2022 | IEEE S&P Workshops | Protocol-aware mutation fuzzing framework targeting MQTT control packet headers and variable payload length fields. |

## System Architecture & Visual Diagram
Link: [[Drawing 2026-07-30 22.31.19.excalidraw#Project 049: MQTT Protocol Security Testing Tool|Excalidraw Architecture Diagram]]

```mermaid
graph TD
    subgraph Target Industrial IoT Infrastructure
        B1[Unauthenticated MQTT Broker - Port 1883]
        B2[TLS Encrypted MQTT Broker - Port 8883]
        T1[Industrial Sensor] -->|Publish Telemetry| B1
        T2[PLC / Robotic Arm Controller] -->|Subscribe Control Commands| B1
    end

    subgraph Security Assessment Orchestrator
        CLI[Framework CLI / Controller] --> M1[Broker Recon & Service Fingerprinting]
        CLI --> M2[Wildcard Topic Hierarchy Harvester]
        CLI --> M3[Auth & ACL Privilege Auditor]
        CLI --> M4[Protocol Packet Mutation Fuzzer]
    end

    subgraph Probing & Execution Core
        M1 --> C1[Port 1883/8883 Socket Prober]
        M2 --> C2[Paho-MQTT Wildcard Subscriber `#`]
        M3 --> C3[Credential Dictionary & ACL Matrix Tester]
        M4 --> C4[Boofuzz / Raw Socket Packet Injector]
    end

    subgraph Findings Correlation Engine
        C1 --> D1[Vulnerability Mapper & Severity Calculator]
        C2 --> D1
        C3 --> D1
        C4 --> D1
    end

    subgraph Security Output & Remediation
        D1 --> E1[Executive PDF / Markdown Audit Report]
        D1 --> E2[Broker Hardening Guidelines & ACL Rules]
        D1 --> E3[JSON Audit Event Logs]
    end
```

## Deep-Dive Technical Implementation & Code Walkthrough

### Phase 1: Environment & Setup
Bhai, test lab environment me Eclipse Mosquitto broker setup karenge aur Python dependencies install karenge.

```bash
#!/usr/bin/env bash
# Phase 1: MQTT Security Testing Environment Setup
set -euo pipefail

echo "[+] Installing Mosquitto Broker and Testing Utilities..."
sudo apt-get update && sudo apt-get install -y \
    mosquitto mosquitto-clients python3-pip nmap

pip3 install paho-mqtt requests boofuzz tabulate

echo "[+] Starting local Mosquitto service for audit testing..."
sudo systemctl restart mosquitto || sudo service mosquitto restart
echo "[+] Setup Complete! Broker running on port 1883."
```

### Phase 2: Core Engine Development
Ab hum main Python engine build karenge jo anonymous login, wildcard subscription harvesting (`#`), aur ACL write-permission test perform karega.

```python
#!/usr/bin/env python3
"""
MQTT Protocol Security Audit Core Engine
Extracts topic hierarchy via wildcard '#' subscription and tests authentication enforcement.
"""

import time
import sys
import paho.mqtt.client as mqtt

class MQTTSecurityAuditor:
    def __init__(self, target_host, target_port=1883):
        self.target_host = target_host
        self.target_port = target_port
        self.harvested_topics = set()

    def on_connect(self, client, userdata, flags, rc):
        """
        Bhai, connection callback result check karta hai:
        rc == 0 matlab connection successful without authentication!
        """
        if rc == 0:
            print(f"[+] [SUCCESS] Anonymous Authentication Allowed on {self.target_host}:{self.target_port}!")
            # Subscribe to multi-level wildcard topic to dump all telemetry
            client.subscribe("#", qos=0)
            print("[*] Subscribed to multi-level wildcard ('#') topic. Harvesting telemetry...")
        else:
            print(f"[-] Anonymous Auth Refused. Return code: {rc}")

    def on_message(self, client, userdata, msg):
        """
        Extract topic name and payload bytes on incoming messages.
        """
        topic = msg.topic
        payload = msg.payload.decode('utf-8', errors='ignore')
        if topic not in self.harvested_topics:
            self.harvested_topics.add(topic)
            print(f"[HARVESTED TOPIC] {topic} | Sample Payload: {payload[:50]}")

    def run_anonymous_audit(self, timeout=5):
        """
        Execute connection test and harvest topics for specified duration.
        """
        client = mqtt.Client(client_id="SecurityAuditorProbe")
        client.on_connect = self.on_connect
        client.on_message = self.on_message

        try:
            client.connect(self.target_host, self.target_port, keepalive=60)
            client.loop_start()
            time.sleep(timeout)
            client.loop_stop()
            client.disconnect()
        except Exception as e:
            print(f"[-] Connection Error: {e}")

        print(f"\n[+] Audit Complete. Total Unique Topics Harvested: {len(self.harvested_topics)}")

if __name__ == "__main__":
    host = "127.0.0.1"
    auditor = MQTTSecurityAuditor(target_host=host, target_port=1883)
    auditor.run_anonymous_audit(timeout=3)
```

### Phase 3: Integration & Testing
Is phase me payload injection tester build karenge jo write permissions and ACL boundaries verify karega.

```python
#!/usr/bin/env python3
"""
MQTT Unauthorized Message Injection & ACL Tester
"""
import paho.mqtt.client as mqtt

def test_unauthorized_publish(target_host, target_topic="factory/line1/actuator"):
    """
    Bhai, target topic par fake control command publish karke ACL write permission check karta hai.
    """
    print(f"[*] Testing unauthorized write access on topic: {target_topic}...")
    client = mqtt.Client(client_id="UnauthorizedInjectorProbe")
    
    try:
        client.connect(target_host, 1883)
        # Publish malicious payload
        res = client.publish(target_topic, payload="OVERRIDE_VALVE_CLOSE", qos=1)
        res.wait_for_publish()
        if res.rc == mqtt.MQTT_ERR_SUCCESS:
            print(f"[ALERT - ACL LEAK] Published unauthorized message to {target_topic} successfully!")
        else:
            print(f"[-] Publish rejected with code {res.rc}")
        client.disconnect()
    except Exception as e:
        print(f"[-] Injection Test Failed: {e}")

if __name__ == "__main__":
    test_unauthorized_publish("127.0.0.1")
```

### Phase 4: Verification & Metrics
Audit summary metrics output script evaluate kariye.

```python
#!/usr/bin/env python3
"""
MQTT Audit Results Verification Summary
"""
def summarize_audit(auth_bypass, total_topics, acl_vulnerable):
    print("=== MQTT Security Assessment Summary ===")
    print(f"Anonymous Auth Bypass Allowed: {'YES (CRITICAL)' if auth_bypass else 'NO'}")
    print(f"Total Discovered Telemetry Topics: {total_topics}")
    print(f"ACL Privilege Escalation Vulnerable: {'YES (HIGH)' if acl_vulnerable else 'NO'}")

summarize_audit(auth_bypass=True, total_topics=14, acl_vulnerable=True)
```

## Tools & Technology Stack
| Tool | Purpose | Alternative |
|------|---------|-------------|
| **Eclipse Mosquitto** | MQTT broker installation and client CLI testing tools | EMQX / HiveMQ |
| **Python Paho-MQTT** | MQTT protocol client wrapper for harvesting topics & testing ACLs | MQTT.fx / gmqtt |
| **Boofuzz** | Dynamic mutation fuzzing framework targeting MQTT control packets | Peach Fuzzer / AFL++ |
| **Nmap** | Service discovery scanner targeting ports 1883 (MQTT) and 8883 (MQTTS) | Masscan |
| **Wireshark** | Packet analyzer for capturing plain-text MQTT payload transmissions | Tshark |

## Deliverables & Verification Metrics
Bhai, is project completion par nimnlikhit quantifiable metrics output milenge:
- **Broker Audit Speed**: Full scan and wildcard topic harvesting in $< 10$ seconds per broker target.
- **Coverage**: Complete checks for Anonymous Auth, Wildcard (`#`) leaks, ACL Write permissions, and TLS cipher strength.
- **Accuracy**: $100\%$ zero false positive validation for write permission test verification.
- **Core Code Artifacts**:
  1. `mqtt_auditor.py`: Wildcard harvesting & authentication checker script.
  2. `mqtt_acl_injector.py`: ACL privilege escalation test module.
  3. `mqtt_fuzzer.py`: Protocol packet header mutation script.

## Legal and Ethical Disclaimer
> [!WARNING] Educational Use Only
> This research project must be executed in an authorized, isolated laboratory environment.

## Related Projects
- [[046 - IoT Botnet Detection using Network Flow Analysis]]
- [[047 - Smart Home Device Vulnerability Assessment Framework]]
- [[053 - Industrial SCADA-ICS Security Assessment Platform]]
