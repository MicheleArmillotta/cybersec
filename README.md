# DDoS Mitigation and Anti-Scan Filtering with eBPF and XDP

This repository provides a summarized technical overview of a project that implements **DDoS mitigation** and **port-scan detection** using **eBPF** and **XDP**.  
The work demonstrates how programmable packet processing in the Linux kernel can serve as an efficient first line of defense against volumetric attacks and reconnaissance activities.


---

## 1. Introduction

The increasing frequency and sophistication of cyberattacks require efficient mechanisms for **monitoring and filtering network traffic**. Traditional filtering tools such as *iptables* and *nftables* introduce significant overhead, especially when dropping large volumes of malicious packets.

This project explores the use of **eBPF** and **XDP (eXpress Data Path)** to mitigate such issues by enabling packet filtering at the earliest stage of packet processing—inside the network driver. The result is a programmable and high-performance solution for detecting DDoS attacks and scanning attempts.

---

## 2. Motivation

The project focuses on two main objectives:

- **Performance and Overhead Reduction**  
  Packet filtering should occur as early as possible. XDP allows decisions to be made before the kernel allocates memory for the packet, reducing overhead.

- **Flexibility and Extensibility**  
  Unlike DPDK-based systems, which trade flexibility for performance, eBPF+XDP offers a balanced approach: high performance without bypassing the Linux kernel stack entirely.

---

## 3. eBPF and XDP Overview

**eBPF** enables safe, verifiable execution of custom programs directly in the Linux kernel without modifying kernel source code. Programs are attached to hooks such as system calls, tracepoints, and network processing points.

**XDP** builds on eBPF by adding an early hook directly in the NIC driver. This enables extremely fast packet processing, often reaching tens of millions of packets per second on commodity hardware.

### eBPF Use Cases  
![eBPF Use Cases](<insert-image-link-here>)

### XDP Placement in the Networking Stack  
![XDP Placement](<insert-image-link-here>)

---

## 4. DDoS Attacks and Port Scanning

### DDoS
A Distributed Denial-of-Service attack overwhelms a target with traffic, disrupting availability and degrading performance. The project focuses on **volumetric attacks**, where the attack is visible through anomalously high packet rates.

### Port Scanning
Port scanning identifies open or closed ports on a target system and often serves as a precursor to further exploitation. Detecting rapid multi-port probing is essential to mitigating reconnaissance activities.

---

## 5. System Architecture

### 5.1 Setup
Experiments were conducted on a Debian 11 VM using the `e1000` network driver.  
Because the driver lacks native XDP support, all tests use **XDP-generic** mode, which emulates XDP in software and results in lower performance compared to XDP-native or hardware offload.

### 5.2 eBPF Maps  
Maps serve as shared data structures between kernel space and user space and are used extensively for configuration, counters, and state management.

![Map Usage Diagram](<insert-image-link-here>)

---

## 6. Architecture and Implementation

### 6.1 DDoS Detector
The DDoS detector identifies malicious IPs by tracking the **number of packets per source IP per second**.  
If the threshold is exceeded, packets from that IP are dropped while legitimate traffic continues.

A userspace daemon:
- loads the eBPF program,
- reads configuration rules,
- updates a logical timestamp,
- receives logs through a ring buffer.

A kernel XDP program:
- parses source IPs,
- counts packets per time bucket,
- drops packets from abusive IPs.

Table: Time synchronization example  
![Time Sync Table](<insert-image-link-here>)

---

### 6.2 Anti-Scan Filtering

The anti-scan filter monitors how many **distinct ports** an IP attempts to contact within a given time frame.  
If the number exceeds a threshold, the IP is classified as a scanner and blacklisted.

The userspace daemon:
- receives events (IP, port),
- updates scanning statistics,
- modifies the blacklist map.

The kernel program:
- inspects packets via XDP,
- checks whether an IP is blacklisted,
- forwards or drops packets accordingly.

---

## 7. Tests and Results

### 7.1 DDoS Detector Results

#### Functional Test  
Three UDP servers were deployed. Two clients remained under the threshold, while one exceeded it.  
Only traffic from the offending client was dropped.

#### Performance Tests  
Tools: `iperf3`, `hping`

Key results:
- During an attack, legitimate traffic suffers heavy degradation; bandwidth dropped from **100 Mbit/s** to under **10 Mbit/s** during attack conditions.
- Packet drop time averaged **4000 ns**, about **250,000 pps**, significantly below native XDP hardware potential.
- XDP-generic and VM virtualization impose severe performance limitations.

Bandwidth impact example:  
![Bandwidth Table](<insert-image-link-here>)

---

### 7.2 Anti-Scan Filter Results

Tests used `iperf3` to measure impact with and without the eBPF anti-scan program.

#### Without eBPF  
- ~3.5 Gbit/s throughput  
- Stable connection, minimal jitter

#### With eBPF  
- Throughput degraded to ~1 Gbit/s  
- Low jitter, manageable packet loss  
- eBPF successfully detects scanning from tools like `nmap`

Performance comparison tables:  
![Anti-Scan Table 1](<insert-image-link-here>)  
![Anti-Scan Table 2](<insert-image-link-here>)

Even at 1 Gbit/s bandwidth cap, the connection remained stable, but packet loss increased due to additional processing overhead.

---

## 8. Conclusions and Future Work

eBPF+XDP proved effective for flexible and programmable filtering. However, due to VM and XDP-generic limitations, performance did not reach native potential.

### Future Enhancements for DDoS Detector
- Test on bare metal with XDP-native  
- Add global thresholds to detect large distributed attacks  
- Implement blacklists and whitelists  
- Improve scalability of IP tracking

### Future Enhancements for Anti-Scanner
- Native XDP testing for realistic performance  
- Smarter scan-pattern classification  
- Rate-limiting mechanisms  
- Enhanced scalability for larger networks

---

## 9. Bibliography

References include documentation for eBPF, libbpf-bootstrap, academic papers on XDP performance, and tools such as `iperf3` and `nmap`.

---

## 10. Authors

- Armillotta Michele  
- De Divitiis Edoardo  
- Raffaelli Andrea  

---

*All figures referenced above correspond to the images embedded in the original GitHub report.*  
*(Replace `<insert-image-link-here>` with the direct GitHub image URLs.)*
