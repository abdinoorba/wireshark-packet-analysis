
---

# **Network Reconnaissance Detection & Analysis Lab**

### *Packet-Level Forensics of a TCP SYN Scan*


*Understanding adversary scanning behavior through hands-on packet-level forensics*

## **Why This Exercise Matters**

Network reconnaissance is the **first stage of the cyberattack kill chain**.
Before attackers exploit vulnerabilities or deploy malware, they first map the environment by probing ports, services, and systems.

This exercise develops the ability to:

* Detect **Indicators of Reconnaissance (IoRs)** in packet captures
* Identify attacker vs. victim hosts through traffic patterns
* Validate scan types (SYN/half-open scans vs. full-connect scans)
* Analyze low-level TCP behavior used by common tools like **Nmap**


Recognizing reconnaissance traffic early allows defenders to **detect intrusions before compromise occurs**, making this a core Blue Team capability.

---


**Tools:** Wireshark, tshark

**PCAP Source:** [Wireshark SampleCaptures](https://wiki.wireshark.org/samplecaptures), `NMap Captures.zip`

**File Analyzed:** [`sample_pcaps/nmap_standard_scan.pcap`](sample_pcaps/nmap_standard_scan.pcap)

---

## Objective

Determine whether the captured network traffic represents reconnaissance activity, identify the attacker and victim hosts, and validate the scan type using packet-level and command-line analysis.

---

## Methods

### 1. Opened Packet Capture

Loaded the packet capture in Wireshark:

**File:**
[`sample_pcaps/nmap_standard_scan.pcap`](sample_pcaps/nmap_standard_scan.pcap)

---

### 2. Applied SYN-Only Filter

Filtered for SYN packets without the ACK flag to detect half-open scan behavior:

```
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

**Screenshot:**
![SYN filter](screenshots/syn-filter.png)

---

### 3. Identified Attacker and Victim Hosts

* **Attacker:** 192.168.100.103
* **Victim:** 192.168.100.102

---

### 4. Isolated Conversation Between Hosts

Applied a two-host filter:

```
ip.addr == 192.168.100.103 && ip.addr == 192.168.100.102
```

**Screenshot:**
![Attacker–Victim Filter](screenshots/attacker-victim-filter.png)

---

### 5. Verified Source of SYN Packets (tshark)

Ran tshark to count SYN packets per source:

```bash
tshark -r sample_pcaps/nmap_standard_scan.pcap \
  -Y "tcp.flags.syn == 1 && tcp.flags.ack == 0" \
  -T fields -e ip.src | sort | uniq -c | sort -rn
```

Output showed:

* **2000 SYN packets**
* **All originating from `192.168.100.103`**

**Screenshot:**
![tshark SYN counts](screenshots/tshark-syn-counts.png)

---

### 6. Counted Destination Ports

Enumerated destination ports targeted by SYN packets:

```bash
tshark -r sample_pcaps/nmap_standard_scan.pcap \
  -Y "tcp.flags.syn == 1 && tcp.flags.ack == 0" \
  -T fields -e tcp.dstport | sort | uniq -c | sort -n
```

The results showed rapid probing of many ports (23, 25, 80, 135, 143, 1723, 8888, etc.).

**Screenshot:**
![Port counts](screenshots/tshark-port-counts.png)

---

### 7. Listed Detailed SYN Packet Characteristics

Displayed frame number, endpoints, MSS, and other TCP options for fingerprinting.

**Screenshot:**
![SYN list](screenshots/tshark-syn-list.png)

---

### 8. Validated Scan Fingerprint

Confirmed Nmap-like behavior using:

* Repeated SYNs
* No full handshakes
* Consistent TCP options (`MSS=1460`, `Window=1024`)
* Sequential port enumeration

**Screenshot:**
![Packet details](screenshots/syn-packet-details.png)

---

### 9. Followed TCP Streams

Followed a selected TCP stream (`tcp.stream eq 5`):

*Stream was empty → no connection established, no payload transfers.*

**Screenshot:**
![Follow stream](screenshots/follow-stream.png)

---

## Findings

* **Attacker:** `192.168.100.103`
* **Victim:** `192.168.100.102`
* **Behavior:** High-volume TCP SYN packets to many ports with no successful handshakes.
* **Scan Type:** **Nmap TCP SYN (half-open) scan**
* **Intent:** Reconnaissance, not exploitation.
* **No payload data** or follow-on activity observed.

---

## Evidence & Reproducibility

### Scripts

* [`scripts/extract_syns.sh`](scripts/extract_syns.sh)
  *Lists all SYN packets and counts per source.*

  **Run:**

  ```
  ./scripts/extract_syns.sh sample_pcaps/nmap_standard_scan.pcap
  ```

* [`scripts/export_attacker_pcap.sh`](scripts/export_attacker_pcap.sh)
  *Exports only attacker traffic to a new PCAP.*

  **Run:**

  ```
  ./scripts/export_attacker_pcap.sh sample_pcaps/nmap_standard_scan.pcap 192.168.100.103 attacker_streams.pcap
  ```

---

## Conclusion & Recommendations

The capture definitively shows **network reconnaissance** using a TCP SYN (half-open) scan, likely performed with Nmap.
No exploitation or lateral movement was detected.

### Recommended:

* Correlate with IDS/firewall logs for `192.168.100.103`
* Review endpoint telemetry for scanning behavior
* Add automated monitoring or blocklisting if repeated
* Continue observing for potential escalation

---

**Author:** Abdinoor Ahmed

*Wireshark Packet Analysis, Cybersecurity Portfolio Lab*

---

