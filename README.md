# NMAP
Nmap host discovery leveraging ARP, ICMP, TCP SYN/ACK, UDP scans, and DNS enumeration

By Ramyar Daneshgar 


## Initial Context: Why Host Discovery Matters

In any offensive security engagement or internal network assessment, time and bandwidth efficiency are critical. Scanning inactive hosts wastes resources, increases the attack surface (detection), and can flood logs with false positives. Before performing port scans or service enumeration, I must first identify which IPs are online and responsive. The goal of this room was to explore multiple ways of conducting live host discovery depending on the network segment, firewall rules, and privilege level available.

---

## 1. ARP Discovery (Link Layer – OSI Layer 2)

### Objective

Use ARP requests to enumerate devices on the same subnet as my machine by soliciting their MAC addresses.

### Command

```bash
sudo nmap -PR -sn 10.10.210.0/24
```

* `-PR`: Enables ARP Ping
* `-sn`: Disables port scanning (host discovery only)

### Why I Used This

ARP is a layer-2 protocol used to resolve IP addresses to MAC addresses. Because ARP is non-routable, this method is strictly limited to the local subnet. Nmap leverages ARP by sending broadcast ARP requests for every IP in the range. Any response is a guarantee that the host is online, as only live systems will respond with their MAC address.

This method is extremely accurate on local Ethernet or wireless LANs and bypasses the need for ICMP or TCP altogether, making it ideal in environments with firewall restrictions on higher-layer traffic.

### Limitation

Cannot be used across routers or for remote subnets. Firewalls are irrelevant here because ARP operates below them at Layer 2.

### Interpretation

This method detected three hosts within my /24 subnet, confirming active systems without making assumptions about firewall configurations.

---

## 2. ICMP-Based Discovery (Network Layer – OSI Layer 3)

ICMP is a network-layer protocol used primarily for diagnostic and control messages. ICMP discovery helps identify systems beyond the local subnet and is a fallback when ARP is not viable.

### A. ICMP Echo Scan

#### Command

```bash
sudo nmap -PE -sn 10.10.210.0/24
```

* `-PE`: Use ICMP Echo (Type 8) requests
* `-sn`: Skip port scan

#### Rationale

This is the traditional “ping sweep.” Nmap sends ICMP Echo Requests and listens for Echo Replies. It's one of the most efficient methods, but many systems (especially hardened Windows hosts or firewalls) block ICMP Echo traffic to avoid reconnaissance.

---

### B. ICMP Timestamp Scan

#### Command

```bash
sudo nmap -PP -sn 10.10.210.0/24
```

* `-PP`: ICMP Timestamp Request (Type 13)

#### Rationale

Even when Echo Requests are filtered, systems may still respond to timestamp requests. This serves as an alternative vector for determining host status. It's particularly useful when scanning network appliances or legacy systems that may leave this functionality open.

---

### C. ICMP Address Mask Scan

#### Command

```bash
sudo nmap -PM -sn 10.10.210.0/24
```

* `-PM`: ICMP Address Mask Request (Type 17)

#### Rationale

Rarely used today, but can still be helpful in detecting poorly configured or legacy hosts. It asks the target for its subnet mask—used historically by diskless clients. A reply confirms the host is up.

---

### Limitation

ICMP is often rate-limited or dropped entirely by endpoint firewalls and perimeter devices. These methods are best used in parallel with TCP and ARP for maximum coverage.

---

## 3. TCP-Based Discovery (Transport Layer – OSI Layer 4)

Firewall configurations that block ICMP may still allow TCP probes to specific services. These techniques test if any transport-layer communication is possible.

### A. TCP SYN Ping

#### Command

```bash
sudo nmap -PS22,80,443 -sn 10.10.210.0/24
```

* `-PS`: Sends SYN packets (like the first part of a TCP handshake)

#### Logic

This is a semi-stealthy scan. I sent SYN packets to common ports (SSH, HTTP, HTTPS) and waited for SYN/ACK replies, which indicate the port is open and the host is alive. Even if the port is closed, an RST response still tells me the host is reachable.

SYN scanning is widely supported and effective because it mimics legitimate service connection attempts. It’s particularly useful when firewalls are configured to allow service-specific traffic.

---

### B. TCP ACK Ping

#### Command

```bash
sudo nmap -PA80,443 -sn 10.10.210.0/24
```

* `-PA`: Sends ACK packets instead of SYN

#### Logic

ACK packets don’t initiate a TCP handshake. They're typically used in established connections. When sent to a closed port, most systems reply with RST. If I receive an RST, the host is up—even if the port is closed.

This method is excellent for bypassing stateless firewalls that permit established connections (for troubleshooting). It tests for responsiveness without explicitly probing services.

---

### Privilege Note

Both SYN and ACK scans require root access to send raw packets without completing the full 3-way handshake.

---

## 4. UDP-Based Discovery

### Command

```bash
sudo nmap -PU53 -sn 10.10.210.0/24
```

* `-PU`: Send UDP packets to target port(s)
* Port 53 (DNS) chosen due to widespread availability

### Rationale

UDP scanning is less reliable due to the protocol's connectionless nature. An open UDP port often does not respond at all. However, when I send a UDP packet to a closed port, I expect an **ICMP Type 3 Code 3 (Port Unreachable)** response. That ICMP error confirms the system is online.

This is especially useful for detecting systems behind stateless firewalls or in environments where TCP and ICMP are filtered.

### Caveat

Because UDP lacks handshakes, many systems will drop unsolicited packets silently, leading to false negatives. Use it alongside TCP/ICMP for best results.

---

## 5. Reverse DNS Enumeration

### Purpose

Nmap attempts reverse DNS lookups to map IPs to hostnames, which can reveal naming conventions and operational context (`mail-server01.internal`).

### Commands

```bash
nmap -sL -R 10.10.210.0/24  # Enable reverse DNS, even for offline hosts
nmap -n -sn 10.10.210.0/24  # Disable all DNS resolution
```

### Rationale

I used `-R` to force DNS lookups even if hosts are offline. This could identify systems that share a common naming schema, which can later help in lateral movement or phishing attacks.

On the other hand, when I want to remain stealthy and avoid alerting internal DNS servers, I use `-n` to disable DNS resolution altogether.

---

## Summary of Discovery Techniques

| Protocol       | OSI Layer | Nmap Option | Notes                                                      |
| -------------- | --------- | ----------- | ---------------------------------------------------------- |
| ARP            | Layer 2   | `-PR`       | Only works on local subnet. Most reliable when applicable. |
| ICMP Echo      | Layer 3   | `-PE`       | Blocked in many hardened environments.                     |
| ICMP Timestamp | Layer 3   | `-PP`       | Alternate path when Echo is blocked.                       |
| ICMP Mask      | Layer 3   | `-PM`       | Legacy method, rarely useful today.                        |
| TCP SYN        | Layer 4   | `-PS`       | Effective and stealthy. SYN/ACK or RST implies host is up. |
| TCP ACK        | Layer 4   | `-PA`       | Bypasses some stateless firewalls.                         |
| UDP            | Layer 4   | `-PU`       | Less reliable, best used as supplementary check.           |
| DNS            | N/A       | `-R`, `-n`  | Reveals naming conventions or allows stealth.              |

---

## Lessons Learned

1. **Protocol Diversity Enhances Reliability**
   No single method works universally. Combining ARP, ICMP, TCP, and UDP ensures broader coverage, especially in heterogeneous network environments.

2. **Privileges Expand Capability**
   Root access is mandatory for SYN and ACK probes without full connections. Scanning from unprivileged user accounts leads to degraded accuracy.

3. **Detection Awareness is Crucial**
   While ARP is silent to IDS systems, ICMP and TCP SYN scans may trigger alerts. I must balance thoroughness with operational security, especially in stealth-sensitive engagements.

4. **Subnet Awareness Directly Impacts Technique Selection**
   If I’m scanning a remote subnet, ARP is irrelevant. Choosing the right technique starts with understanding the network topology.

5. **Reverse DNS Can Be High Signal**
   In enterprise environments, hostnames often encode asset roles, departments, or users. Reverse DNS can offer valuable pretexting information or pivot points.
