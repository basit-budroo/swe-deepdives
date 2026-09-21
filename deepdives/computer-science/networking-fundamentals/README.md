# Networking Fundamentals for System Design

A comprehensive guide to networking concepts, protocols, and technologies essential for designing scalable distributed systems.

---

## Table of Contents

1. [Network Models](#1-network-models)
   - [OSI Model](#osi-model)
   - [TCP/IP Model](#tcpip-model)
2. [IP Addressing](#2-ip-addressing)
   - [IPv4](#ipv4)
   - [IPv6](#ipv6)
   - [Subnetting and CIDR](#subnetting-and-cidr)
   - [Private vs Public IP Addresses](#private-vs-public-ip-addresses)
   - [NAT (Network Address Translation)](#nat-network-address-translation)
3. [Transport Layer Protocols](#3-transport-layer-protocols)
   - [TCP (Transmission Control Protocol)](#tcp-transmission-control-protocol)
   - [UDP (User Datagram Protocol)](#udp-user-datagram-protocol)
   - [TCP vs UDP Comparison](#tcp-vs-udp-comparison)
   - [TCP Connection Lifecycle](#tcp-connection-lifecycle)
   - [TCP Flow Control and Congestion Control](#tcp-flow-control-and-congestion-control)
4. [Application Layer Protocols](#4-application-layer-protocols)
   - [HTTP/1.1](#http11)
   - [HTTP/2](#http2)
   - [HTTP/3 and QUIC](#http3-and-quic)
   - [HTTPS and TLS/SSL](#https-and-tlsssl)
   - [WebSocket](#websocket)
   - [gRPC](#grpc)
   - [REST API Design](#rest-api-design)
5. [Domain Name System (DNS)](#5-domain-name-system-dns)
   - [DNS Resolution Process](#dns-resolution-process)
   - [DNS Record Types](#dns-record-types)
   - [DNS Caching](#dns-caching)
   - [Load Balancing with DNS](#load-balancing-with-dns)
6. [Load Balancing](#6-load-balancing)
   - [Layer 4 vs Layer 7 Load Balancing](#layer-4-vs-layer-7-load-balancing)
   - [Load Balancing Algorithms](#load-balancing-algorithms)
   - [Health Checks](#health-checks)
   - [Session Persistence](#session-persistence)
7. [Caching and Content Delivery](#7-caching-and-content-delivery)
   - [CDN Architecture](#cdn-architecture)
   - [Caching Strategies](#caching-strategies)
   - [Cache Invalidation](#cache-invalidation)
8. [Network Security](#8-network-security)
   - [TLS/SSL Handshake](#tlsssl-handshake)
   - [Certificate Authority (CA)](#certificate-authority-ca)
   - [Firewalls](#firewalls)
   - [DDoS Protection](#ddos-protection)
   - [VPN and Tunneling](#vpn-and-tunneling)
9. [Proxy Servers](#9-proxy-servers)
   - [Forward Proxy](#forward-proxy)
   - [Reverse Proxy](#reverse-proxy)
   - [Proxy vs Load Balancer](#proxy-vs-load-balancer)
10. [Advanced Networking Concepts](#10-advanced-networking-concepts)
    - [Connection Pooling](#connection-pooling)
    - [Keep-Alive](#keep-alive)
    - [Circuit Breakers](#circuit-breakers)
    - [Service Mesh](#service-mesh)
    - [API Gateway](#api-gateway)
11. [Performance and Optimization](#11-performance-and-optimization)
    - [Latency vs Bandwidth](#latency-vs-bandwidth)
    - [Network Jitter](#network-jitter)
    - [Packet Loss Handling](#packet-loss-handling)
    - [Latency Numbers](#latency-numbers)

---

## 1. Network Models

### OSI Model

The **Open Systems Interconnection (OSI) Model** is a conceptual framework that standardizes network communication into seven distinct layers. Developed by ISO in 1984, it provides a universal standard for network design and troubleshooting.

**Why it matters for System Design:**
Understanding the OSI model helps you:
- Debug network issues by isolating problems to specific layers
- Choose the right protocols for your use case
- Design systems that properly separate concerns
- Communicate effectively with network engineers
- Understand where load balancers, proxies, and firewalls operate

| Layer | Name | Function | Examples | System Design Relevance |
|-------|------|----------|----------|-------------------------|
| 7 | Application | User interface and application services | HTTP, FTP, SMTP, DNS | API design, protocol selection for microservices |
| 6 | Presentation | Data translation, encryption, compression | SSL/TLS, JPEG, ASCII | Data serialization (JSON, Protobuf), encryption strategies |
| 5 | Session | Session management, dialog control | NetBIOS, RPC | Session management in distributed systems, connection pooling |
| 4 | Transport | End-to-end communication, reliability | TCP, UDP | Choosing TCP vs UDP, connection handling, load balancing |
| 3 | Network | Routing, logical addressing | IP, ICMP, OSPF, BGP | IP addressing, subnetting, network segmentation |
| 2 | Data Link | Physical addressing, error detection | Ethernet, Wi-Fi (802.11), PPP | Network switches, VLANs, MAC address filtering |
| 1 | Physical | Raw bit transmission over physical medium | Cables, fiber optics, radio signals | Hardware selection, bandwidth planning |

**Detailed Layer Explanations:**

**Layer 7 - Application:**
- This is where your applications live and interact with users
- Protocols here are what you'll work with most directly
- **System Design Impact:** When designing APIs, you're working at Layer 7. Choosing between REST, GraphQL, gRPC happens here.

**Layer 6 - Presentation:**
- Handles data format translation (e.g., EBCDIC to ASCII)
- Encryption/decryption (SSL/TLS operates here)
- Compression (gzip, deflate)
- **System Design Impact:** Data serialization formats (JSON, XML, Protobuf) are presentation layer concerns. This affects payload size and parsing performance.

**Layer 5 - Session:**
- Establishes, maintains, and terminates connections
- Handles authentication and authorization
- **System Design Impact:** Session state management, connection pooling, and session persistence in load balancers are session-layer concerns.

**Layer 4 - Transport:**
- Ensures reliable data delivery (TCP) or fast delivery (UDP)
- Port-based addressing (80, 443, 22, etc.)
- **System Design Impact:** Critical for deciding between TCP and UDP, implementing connection pooling, and designing transport-level load balancing.

**Layer 3 - Network:**
- IP addressing and routing
- Packet forwarding across networks
- **System Design Impact:** Subnetting, VPC design, network segmentation, and choosing between IPv4 and IPv6.

**Layer 2 - Data Link:**
- MAC addressing
- Error detection and correction
- **System Design Impact:** Switch configuration, VLANs for network isolation, and understanding broadcast domains.

**Layer 1 - Physical:**
- Physical transmission medium
- **System Design Impact:** Bandwidth planning, choosing between copper, fiber, or wireless, and understanding physical limitations.

**Mnemonic for remembering layers:** "**P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way" (Physical → Data Link → Network → Transport → Session → Presentation → Application)

**Real-World Example - Web Request Through OSI Layers:**
```
User clicks link in browser
↓
Layer 7: HTTP GET request created
↓
Layer 6: Data formatted (UTF-8), encrypted (HTTPS), compressed
↓
Layer 5: Session established with server
↓
Layer 4: TCP connection opened, data segmented
↓
Layer 3: IP addresses added, routing determined
↓
Layer 2: MAC addresses added, frame created
↓
Layer 1: Bits converted to electrical/optical signals
```

### TCP/IP Model

The **TCP/IP Model** (Internet Protocol Suite) is the practical implementation used in modern networking, combining some OSI layers. It was developed by DARPA in the 1970s and became the foundation of the modern Internet.

**Why it matters for System Design:**
- This is the model actually used in real-world networks
- Understanding it helps you debug actual network issues
- Most cloud providers (AWS, GCP, Azure) operate at these layers
- Your systems will interact with these protocols daily

| Layer | OSI Equivalent | Protocols | System Design Relevance |
|-------|----------------|-----------|-------------------------|
| Application (4) | Session + Presentation + Application | HTTP, FTP, SMTP, DNS, SSH | API design, protocol selection, application-level load balancing |
| Transport (3) | Transport | TCP, UDP, SCTP | Connection handling, reliability vs speed decisions |
| Internet (2) | Network | IP, ICMP, ARP, IGMP | IP addressing, routing, subnetting, VPC design |
| Network Access (1) | Data Link + Physical | Ethernet, Wi-Fi, ARP | Physical network configuration, hardware selection |

**Key Differences Between OSI and TCP/IP:**

| Aspect | OSI Model | TCP/IP Model |
|--------|-----------|--------------|
| **Purpose** | Theoretical reference model | Practical implementation |
| **Layers** | 7 distinct layers | 4 layers (combines some) |
| **Development** | Standardized by ISO | Developed by DARPA, evolved organically |
| **Usage** | Teaching and reference | Actual Internet implementation |
| **Flexibility** | Rigid layer separation | More flexible, protocols can span layers |

**Why TCP/IP Won:**
- **Simpler implementation:** Fewer layers meant faster deployment
- **Early adoption:** Used in ARPANET before OSI was formalized
- **Practical focus:** Built for actual use, not theoretical purity
- **Vendor support:** Major vendors implemented it early

**Real-World Example - Email Through TCP/IP Layers:**
```
User sends email via SMTP
↓
Application Layer: SMTP protocol formats email message
↓
Transport Layer: TCP segments the message, ensures delivery
↓
Internet Layer: IP routes packets through Internet to mail server
↓
Network Access Layer: Ethernet frames transmitted over local network
```

---

## 2. IP Addressing

### IPv4

**IPv4 (Internet Protocol version 4)** uses 32-bit addresses, providing approximately 4.3 billion unique addresses. Designed in 1981, IPv4 was the first widely deployed version of IP.

**Why it matters for System Design:**
- Understanding IPv4 is essential for network configuration in most existing systems
- Private IP ranges are used in VPCs and internal networks
- Subnetting affects network segmentation and security
- NAT is used in most cloud deployments

**Format:** Four octets (0-255) separated by dots
```
Example: 192.168.1.1
Binary:  11000000.10101000.00000001.00000001
Decimal: 192      .168    .1      .1
```

**IPv4 Exhaustion:**
- Total addresses: 2^32 = 4,294,967,296
- Usable addresses: ~3.7 billion (many reserved)
- **Problem:** Not enough for every device on Earth
- **Solution:** NAT (Network Address Translation) and IPv6

**Address Classes (Historical):**
Class-based addressing was the original scheme but has been replaced by CIDR.

| Class | First Octet Range | Default Subnet Mask | Network Size | Use Case |
|-------|-------------------|---------------------|--------------|----------|
| A | 1-126 | 255.0.0.0 (/8) | 16.7M hosts | Large networks (e.g., ISP backbone) |
| B | 128-191 | 255.255.0.0 (/16) | 65,534 hosts | Medium networks (e.g., universities) |
| C | 192-223 | 255.255.255.0 (/24) | 254 hosts | Small networks (e.g., companies) |
| D | 224-239 | N/A | N/A | Multicast (streaming, routing protocols) |
| E | 240-255 | N/A | N/A | Reserved/Experimental |

**Special IPv4 Addresses:**

| Address | Purpose | System Design Relevance |
|----------|---------|------------------------|
| `127.0.0.1` | Loopback/localhost | Testing services locally without network access |
| `0.0.0.0` | Default route / all interfaces | Server binding (listen on all interfaces) |
| `255.255.255.255` | Limited broadcast | Network discovery, DHCP |
| `169.254.0.0/16` | Link-local (APIPA) | Auto-configuration when DHCP fails |
| `224.0.0.0/4` | Multicast | Streaming, routing protocols |

**Real-World Example - AWS VPC IPv4:**
```
AWS VPC CIDR: 10.0.0.0/16
- Subnet A (public): 10.0.1.0/24 (254 usable IPs)
- Subnet B (private): 10.0.2.0/24 (254 usable IPs)
- Subnet C (database): 10.0.3.0/24 (254 usable IPs)

Reserved IPs in each subnet:
- First IP (.0): Network address
- Second IP (.1): AWS router
- Last IP (.255): Broadcast
```

### IPv6

**IPv6 (Internet Protocol version 6)** uses 128-bit addresses, providing 340 undecillion (3.4 × 10^38) addresses. Designed to solve IPv4 exhaustion.

**Why it matters for System Design:**
- Future-proofing systems for global deployment
- Eliminates need for NAT, simplifying network architecture
- Better multicast support for scalable messaging
- Built-in security with IPsec
- Many cloud providers support IPv6 natively

**Format:** Eight groups of four hexadecimal digits, separated by colons
```
Full:    2001:0db8:85a3:0000:0000:8a2e:0370:7334
Compressed: 2001:db8:85a3::8a2e:370:7334
```

**Compression Rules:**
1. **Leading zeros:** Can be omitted (0db8 → db8)
2. **Consecutive zero groups:** Can be replaced with `::` (only once)
```
2001:0db8:0000:0000:0000:0000:0000:0001
     → 2001:db8::1

2001:0db8:0000:0000:0000:ff00:0042:8329
     → 2001:db8::ff00:42:8329
```

**Key IPv6 Features:**

| Feature | IPv4 | IPv6 | System Design Impact |
|----------|------|------|---------------------|
| **Address Space** | 32-bit (4.3B) | 128-bit (340 undecillion) | Every device can have unique address |
| **Header Size** | 20-60 bytes (variable) | 40 bytes (fixed) | Predictable overhead, faster routing |
| **Fragmentation** | Routers can fragment | Source only (PMTU discovery) | Better performance, simpler routers |
| **Security** | Optional IPsec | Mandatory IPsec | Built-in encryption and authentication |
| **Configuration** | Manual or DHCP | SLAAC (auto-configuration) | Plug-and-play device deployment |
| **NAT** | Required | Not needed | True end-to-end connectivity |
| **Multicast** | Optional | Native support | Efficient one-to-many communication |

**IPv6 Address Types:**

| Type | Prefix | Description | System Design Relevance |
|------|--------|-------------|------------------------|
| Global Unicast | 2000::/3 | Routable on Internet | Public-facing services |
| Unique Local | fc00::/7 | Private networks (like RFC 1918) | Internal VPCs, similar to 10.0.0.0/8 |
| Link-Local | fe80::/10 | Local segment only | Neighbor discovery, auto-configuration |
| Loopback | ::1 | Local host | Testing, equivalent to 127.0.0.1 |
| Multicast | ff00::/8 | One-to-many | Streaming, service discovery |
| Anycast | Same as unicast | Nearest of multiple receivers | DNS anycast, CDN edge selection |

**Real-World Example - AWS IPv6 VPC:**
```
AWS VPC with IPv6:
- IPv4 CIDR: 10.0.0.0/16 (for compatibility)
- IPv6 CIDR: 2600:1f13:1234::/56 (from Amazon's pool)

Subnet configuration:
- Subnet A: 2600:1f13:1234:1::/64
- Subnet B: 2600:1f13:1234:2::/64

Each subnet: 2^64 addresses (more than enough)

Instance gets:
- IPv4: 10.0.1.15 (private)
- IPv6: 2600:1f13:1234:1::15 (public, no NAT needed)
```

**IPv6 Transition Strategies:**

1. **Dual Stack:** Run both IPv4 and IPv6 simultaneously
   - Most common approach
   - Gradual migration
   - Requires maintaining both protocols

2. **Tunneling:** Encapsulate IPv6 in IPv4 packets
   - 6to4, Teredo, ISATAP
   - Temporary solution

3. **Translation:** NAT64 (translate IPv6 to IPv4)
   - For IPv6-only clients accessing IPv4 services
   - Used by mobile networks

### Subnetting and CIDR

**CIDR (Classless Inter-Domain Routing)** notation represents IP addresses and their subnet masks. Introduced in 1993 to replace class-based addressing.

**Why it matters for System Design:**
- Efficient IP address allocation (no waste like class-based)
- Network segmentation for security (VPC subnets)
- Route summarization for scalable routing
- Cloud VPC design (AWS, GCP, Azure all use CIDR)
- Calculating network capacity for planning

**Format:** `IP_Address/Prefix_Length`
```
192.168.1.0/24 means:
- Network: 192.168.1.0
- Subnet Mask: 255.255.255.0 (24 bits set to 1, 8 bits for hosts)
- Total IPs: 256 (2^8)
- Usable hosts: 254 (2^8 - 2, minus network and broadcast)
- First usable: 192.168.1.1
- Last usable: 192.168.1.254
- Broadcast: 192.168.1.255
```

**Common CIDR Blocks:**

| CIDR | Subnet Mask | Total IPs | Usable IPs | Typical Use |
|------|-------------|-----------|------------|-------------|
| /32 | 255.255.255.255 | 1 | 1 | Single host route |
| /31 | 255.255.255.254 | 2 | 2* | Point-to-point links |
| /30 | 255.255.255.252 | 4 | 2 | Small subnet (2 hosts) |
| /29 | 255.255.255.248 | 8 | 6 | Small office (6 hosts) |
| /28 | 255.255.255.240 | 16 | 14 | Small subnet (14 hosts) |
| /27 | 255.255.255.224 | 32 | 30 | Medium subnet (30 hosts) |
| /26 | 255.255.255.192 | 64 | 62 | Medium subnet (62 hosts) |
| /25 | 255.255.255.128 | 128 | 126 | Large subnet (126 hosts) |
| /24 | 255.255.255.0 | 256 | 254 | Standard subnet (class C equivalent) |
| /23 | 255.255.254.0 | 512 | 510 | Two /24 combined |
| /22 | 255.255.252.0 | 1024 | 1022 | Four /24 combined |
| /21 | 255.255.248.0 | 2048 | 2046 | Eight /24 combined |
| /20 | 255.255.240.0 | 4096 | 4094 | Sixteen /24 combined |
| /19 | 255.255.224.0 | 8192 | 8190 | AWS VPC subnet |
| /18 | 255.255.192.0 | 16384 | 16382 | Large subnet |
| /17 | 255.255.128.0 | 32768 | 32766 | Very large subnet |
| /16 | 255.255.0.0 | 65,536 | 65,534 | Class B equivalent |
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 | Class A equivalent |

*/31 Note:* Traditionally, first and last IP were reserved. /31 is special case for point-to-point links where both IPs are usable.

**Subnetting Calculations - Step by Step:**

**Example 1: Subnetting a /24 into smaller subnets**
```
Given: 192.168.1.0/24
Need: 4 subnets

Step 1: Calculate bits to borrow
- 2^x >= 4, so x = 2 bits
- New prefix: 24 + 2 = /26

Step 2: Calculate new subnet mask
- /26 = 255.255.255.192
- Block size: 256 - 192 = 64

Step 3: List subnets
- Subnet 1: 192.168.1.0/26 (192.168.1.0 - 192.168.1.63)
- Subnet 2: 192.168.1.64/26 (192.168.1.64 - 192.168.1.127)
- Subnet 3: 192.168.1.128/26 (192.168.1.128 - 192.168.1.191)
- Subnet 4: 192.168.1.192/26 (192.168.1.192 - 192.168.1.255)

Step 4: Usable IPs per subnet
- Total: 64 IPs
- Usable: 62 IPs (subtract network and broadcast)
```

**Example 2: Determine if IP is in subnet**
```
Is 192.168.1.100 in 192.168.1.64/26?

Method 1: Binary
- 192.168.1.100 = 11000000.10101000.00000001.01100100
- /26 mask        = 11111111.11111111.11111111.11000000
- Network bits    = 11000000.10101000.00000001.01000000 = 192.168.1.64
- Yes, it's in the subnet

Method 2: Block size
- /26 block size = 256 / 2^(32-26) = 256 / 64 = 4 blocks of 64
- Block 0: 0-63
- Block 1: 64-127 ← 100 is here
- Yes, it's in 192.168.1.64/26
```

**Example 3: Required subnets calculation**
```
Need: 3 subnets with 50, 25, and 10 hosts

Subnet 1 (50 hosts):
- Need 2^x - 2 >= 50, so x = 6 (62 usable)
- Prefix: 32 - 6 = /26

Subnet 2 (25 hosts):
- Need 2^x - 2 >= 25, so x = 5 (30 usable)
- Prefix: 32 - 5 = /27

Subnet 3 (10 hosts):
- Need 2^x - 2 >= 10, so x = 4 (14 usable)
- Prefix: 32 - 4 = /28

Allocation from 192.168.1.0/24:
- Subnet 1: 192.168.1.0/26 (0-63)
- Subnet 2: 192.168.1.64/27 (64-95)
- Subnet 3: 192.168.1.96/28 (96-111)
- Remaining: 192.168.1.112/25 (112-255) for future use
```

### Private vs Public IP Addresses

**Why it matters for System Design:**
- Private IPs are used in VPCs and internal networks
- Public IPs are required for Internet-facing services
- NAT bridges the gap between private and public
- Understanding this is crucial for security (don't expose private services)
- Cost optimization (public IPs often cost money)

**Private IP Ranges (RFC 1918):**

| Range | CIDR | Usable IPs | Typical Use |
|-------|------|------------|-------------|
| 10.0.0.0 - 10.255.255.255 | 10.0.0.0/8 | 16,777,216 | Large enterprises, AWS VPCs |
| 172.16.0.0 - 172.31.255.255 | 172.16.0.0/12 | 1,048,576 | Medium enterprises |
| 192.168.0.0 - 192.168.255.255 | 192.168.0.0/16 | 65,536 | Small offices, home networks |

**Characteristics:**

| Aspect | Private IPs | Public IPs |
|--------|-------------|------------|
| **Routing** | Not routable on Internet | Globally routable |
| **Uniqueness** | Can be reused across networks | Must be globally unique |
| **Allocation** | Free to use within private network | Allocated by IANA/RIRs |
| **NAT Required** | Yes, for Internet access | No, direct Internet access |
| **Cost** | Free | Often costs money (cloud providers) |
| **Security** | Not directly exposed to Internet | Exposed, need security measures |

**Real-World Example - AWS VPC Design:**
```
AWS VPC: 10.0.0.0/16 (private)

Public Subnet (Internet-facing):
- CIDR: 10.0.1.0/24
- Instances: Web servers, load balancers
- Each instance: Private IP + Public IP (or Elastic IP)
- Internet Gateway provides NAT to Internet

Private Subnet (Internal):
- CIDR: 10.0.2.0/24
- Instances: Application servers, databases
- Each instance: Private IP only
- NAT Gateway for outbound Internet access
- No direct inbound from Internet

Database Subnet (Isolated):
- CIDR: 10.0.3.0/24
- Instances: Databases only
- No Internet Gateway access
- Maximum security

Flow:
Internet → Public IP (NAT) → Private IP → Internal Service
```

**Public IP Allocation Hierarchy:**
```
IANA (Internet Assigned Numbers Authority)
  ↓ Allocates to
RIRs (Regional Internet Registries)
  ├─ ARIN (North America)
  ├─ RIPE NCC (Europe)
  ├─ APNIC (Asia-Pacific)
  ├─ LACNIC (Latin America)
  └─ AfriNIC (Africa)
      ↓ Allocates to
LIRs (Local Internet Registries) / ISPs
      ↓ Allocates to
End Users / Organizations
```

### NAT (Network Address Translation)

**NAT** allows multiple devices on a private network to share a single public IP address. NAT was created to address IPv4 exhaustion.

**Why it matters for System Design:**
- Almost all cloud deployments use NAT
- Understanding NAT is crucial for debugging connectivity issues
- NAT affects logging, security, and peer-to-peer applications
- Different NAT types impact application design
- NAT traversal techniques needed for certain applications

**Types of NAT:**

**1. Static NAT (Full Cone NAT):**
One-to-one mapping between private and public IP
```
192.168.1.10 ↔ 203.0.113.10 (always)

Use Case: Hosting public service on internal server
Pros: Predictable, easy to configure
Cons: Wastes public IPs, one public IP per private IP
```

**2. Dynamic NAT:**
Maps private IP to a pool of public IPs (first-come, first-served)
```
192.168.1.10 → 203.0.113.10 (first available)
192.168.1.11 → 203.0.113.11 (next available)
192.168.1.12 → 203.0.113.12 (next available)

When connection closes, IP returns to pool

Use Case: Multiple users need Internet access
Pros: Efficient use of public IPs
Cons: Limited to pool size, no guaranteed mapping
```

**3. PAT/NAPT (Port Address Translation):**
Most common; multiple private IPs share one public IP using different ports
```
Translation Table:
192.168.1.10:12345 → 203.0.113.10:50001
192.168.1.11:12345 → 203.0.113.10:50002
192.168.1.12:12345 → 203.0.113.10:50003
192.168.1.10:54321 → 203.0.113.10:50004

Use Case: Home router, office Internet access
Pros: Thousands of devices can share one IP
Cons: Port exhaustion possible (65,535 ports max)
```

**NAT Types (for P2P applications):**

| Type | Behavior | P2P Difficulty |
|------|----------|---------------|
| **Full Cone** | Any external host can send to mapped port | Easy (least restrictive) |
| **Restricted Cone** | Only hosts you've sent to can respond | Moderate |
| **Port Restricted Cone** | Only specific host:port you've sent to can respond | Hard |
| **Symmetric** | Different mappings for different destinations | Very hard (requires STUN/TURN) |

**How NAT Works - Step by Step:**
```
Internal Device (192.168.1.10) wants to access google.com (142.250.185.78)

Step 1: Outbound Packet
Source: 192.168.1.10:54321
Destination: 142.250.185.78:443

Step 2: NAT Translation
Router: "I'll map this to my public IP + new port"
Source: 203.0.113.10:50001 (translated)
Destination: 142.250.185.78:443

Step 3: Translation Table Entry
{internal: 192.168.1.10:54321, external: 203.0.113.10:50001, dest: 142.250.185.78:443}

Step 4: Response from Google
Source: 142.250.185.78:443
Destination: 203.0.113.10:50001

Step 5: NAT Reverse Lookup
Router: "Port 50001 maps to 192.168.1.10:54321"
Source: 142.250.185.78:443
Destination: 192.168.1.10:54321

Step 6: Internal Device receives response
```

**NAT Limitations and Issues:**

| Issue | Description | Impact on System Design |
|-------|-------------|--------------------------|
| **End-to-end connectivity** | Breaks direct connections | P2P applications need STUN/TURN/ICE |
| **IP address in payload** | Some protocols embed IP in data | FTP, SIP need ALG (Application Layer Gateway) |
| **Port exhaustion** | Limited ports (~65K) | High-traffic systems need multiple public IPs |
| **Logging** | Source IP appears as NAT IP | Need X-Forwarded-For headers |
| **SSL/TLS** | Certificates bound to IP | SNI (Server Name Indication) required |
| **Performance** | Translation overhead | Hardware NAT recommended for high throughput |

**NAT Traversal Techniques:**

**1. STUN (Session Traversal Utilities for NAT):**
- Discover public IP and port mapping
- Works for Full Cone, Restricted Cone
- Doesn't work for Symmetric NAT

**2. TURN (Traversal Using Relays around NAT):**
- Relay server for when direct connection fails
- Works for all NAT types
- Higher latency, more server cost

**3. ICE (Interactive Connectivity Establishment):**
- Try multiple methods: direct, STUN, TURN
- Choose best working method
- Used by WebRTC

**Real-World Example - AWS NAT Gateway:**
```
AWS VPC: 10.0.0.0/16

Public Subnet: 10.0.1.0/24
  - NAT Gateway (has Elastic IP: 203.0.113.10)
  - Internet Gateway

Private Subnet: 10.0.2.0/24
  - Application servers (10.0.2.10, 10.0.2.11)
  - No public IPs

Flow:
1. App server (10.0.2.10) makes request to api.example.com
2. Route table sends to NAT Gateway
3. NAT Gateway translates to 203.0.113.10:random_port
4. NAT Gateway forwards to Internet Gateway
5. Response comes back to NAT Gateway
6. NAT Gateway reverses translation
7. Response delivered to 10.0.2.10

Benefits:
- Private servers can access Internet
- No public IP needed for each server
- Security (servers not directly exposed)
- Cost savings (fewer Elastic IPs)
```

---

## 3. Transport Layer Protocols

### TCP (Transmission Control Protocol)

**TCP** is a connection-oriented, reliable transport protocol. Designed in 1974, TCP provides reliable, ordered, and error-checked delivery of data between applications.

**Why it matters for System Design:**
- TCP is the default for most web applications (HTTP, HTTPS, SMTP, etc.)
- Understanding TCP helps debug connection issues and performance problems
- TCP's reliability comes with overhead that affects system performance
- Connection pooling and keep-alive settings directly impact scalability
- TCP's congestion control affects how your system behaves under load
- TCP window size tuning can improve throughput for high-latency connections

**Key Characteristics:**

| Characteristic | Description | System Design Impact |
|----------------|-------------|----------------------|
| **Connection-oriented** | Three-way handshake establishes connection | Connection setup overhead (1 RTT minimum) |
| **Reliable** | Acknowledgments, retransmissions, sequencing | Guaranteed delivery but slower than UDP |
| **Ordered delivery** | Sequence numbers ensure correct order | Data arrives in order sent |
| **Flow control** | Sliding window prevents overwhelming receiver | Prevents buffer overflow at receiver |
| **Congestion control** | Adjusts transmission rate based on network conditions | Prevents network collapse, affects throughput |
| **Full duplex** | Simultaneous two-way communication | Efficient bidirectional data transfer |
| **Byte-stream** | No message boundaries | Application must define message boundaries |

**When to Use TCP:**
- **Web applications:** HTTP/HTTPS, FTP, SMTP
- **File transfer:** Need guaranteed delivery
- **Databases:** Transaction integrity
- **Remote shell:** SSH, Telnet
- **Email:** SMTP, IMAP, POP
- **Any application where data loss is unacceptable**

**TCP Header Structure (20-60 bytes):**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window               |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (optional)                    |Pad|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key TCP Fields Explained:**

| Field | Size | Purpose | System Design Relevance |
|-------|------|---------|--------------------------|
| **Source Port** | 16 bits | Sender's port | Identifies which application sent data |
| **Destination Port** | 16 bits | Receiver's port | Identifies which application should receive data |
| **Sequence Number** | 32 bits | Byte position in data stream | Enables ordered delivery and reassembly |
| **Acknowledgment Number** | 32 bits | Next expected byte | Confirms receipt and what to send next |
| **Data Offset** | 4 bits | Header length | Indicates where data starts (20-60 bytes) |
| **Flags** | 9 bits | Control flags (SYN, ACK, FIN, etc.) | Controls connection state |
| **Window Size** | 16 bits | Receiver's buffer capacity | Flow control - how much sender can send |
| **Checksum** | 16 bits | Error detection | Detects corrupted packets |
| **Urgent Pointer** | 16 bits | Urgent data location | Rarely used (legacy feature) |
| **Options** | Variable | Extensions (MSS, window scaling, etc.) | Performance tuning capabilities |

**TCP Flags Explained:**

| Flag | Name | Purpose | When Set |
|------|------|---------|----------|
| **SYN** | Synchronize | Initiate connection | Handshake step 1 & 2 |
| **ACK** | Acknowledge | Confirm receipt | Most packets after connection |
| **FIN** | Finish | Close connection | Graceful shutdown |
| **RST** | Reset | Abort connection immediately | Error, reject connection |
| **PSH** | Push | Deliver to application immediately | Interactive data |
| **URG** | Urgent | Urgent data pointer | Rarely used |
| **ECE** | ECN-Echo | Explicit congestion notification | Congestion signaling |
| **CWR** | Congestion Window Reduced | Response to ECN | Congestion response |
| **NS** | Nonce Sum | ECN protection | Security feature |

**Well-Known Ports:**

| Port | Service | Protocol | System Design Relevance |
|------|---------|----------|------------------------|
| 20/21 | FTP | TCP | File transfer (legacy, consider SFTP) |
| 22 | SSH | TCP | Remote administration, secure tunneling |
| 23 | Telnet | TCP | Legacy remote access (insecure, avoid) |
| 25 | SMTP | TCP | Email delivery (outbound) |
| 53 | DNS | UDP/TCP | Domain resolution (UDP for queries, TCP for zone transfers) |
| 67/68 | DHCP | UDP | Automatic IP configuration |
| 80 | HTTP | TCP | Unencrypted web traffic |
| 110 | POP3 | TCP | Email retrieval (legacy) |
| 123 | NTP | UDP | Time synchronization |
| 143 | IMAP | TCP | Email retrieval |
| 443 | HTTPS | TCP | Encrypted web traffic |
| 465 | SMTPS | TCP | Encrypted email submission |
| 587 | SMTP | TCP | Email submission (with STARTTLS) |
| 993 | IMAPS | TCP | Encrypted email retrieval |
| 995 | POP3S | TCP | Encrypted email retrieval (legacy) |
| 3306 | MySQL | TCP | Database connections |
| 3389 | RDP | TCP | Windows remote desktop |
| 5432 | PostgreSQL | TCP | Database connections |
| 5672 | AMQP | TCP | Message queue (RabbitMQ) |
| 6379 | Redis | TCP | In-memory cache/database |
| 27017 | MongoDB | TCP | NoSQL database |
| 6380 | Redis | TCP | Redis (cluster mode) |
| 9092 | Kafka | TCP | Event streaming |
| 9200 | Elasticsearch | TCP | Search engine |
| 11211 | Memcached | TCP/UDP | Distributed cache |

**Port Ranges:**
- **Well-known ports:** 0-1023 (require root/admin to bind)
- **Registered ports:** 1024-49151 (assigned by IANA)
- **Dynamic/Private ports:** 49152-65535 (ephemeral ports)

**Real-World Example - Port Selection:**
```
Microservice Architecture:
- API Gateway: 8080 (HTTP), 8443 (HTTPS)
- User Service: 8001
- Order Service: 8002
- Payment Service: 8003
- Notification Service: 8004

Database Ports:
- PostgreSQL: 5432
- Redis: 6379
- MongoDB: 27017

Internal vs External:
- External: 80, 443 (public-facing)
- Internal: 8000-9000 (microservices)
- Database: 3306, 5432, 6379 (restricted access)
```

### UDP (User Datagram Protocol)

**UDP** is a connectionless, unreliable transport protocol. Designed in 1980, UDP provides minimal, message-oriented transport with no guarantees.

**Why it matters for System Design:**
- UDP is essential for real-time applications (gaming, streaming, VoIP)
- Understanding UDP helps choose the right protocol for your use case
- UDP's speed comes at the cost of reliability (application must handle errors)
- UDP supports multicast, enabling efficient one-to-many communication
- QUIC (HTTP/3) uses UDP as transport layer
- DNS uses UDP for fast queries

**Key Characteristics:**

| Characteristic | Description | System Design Impact |
|----------------|-------------|----------------------|
| **Connectionless** | No handshake; send immediately | Zero connection setup overhead |
| **Unreliable** | No acknowledgments or retransmissions | Application must handle loss and errors |
| **No ordering** | No sequence numbers | Application must handle ordering if needed |
| **No flow control** | No window mechanism | Can overwhelm receiver or network |
| **No congestion control** | No rate adjustment | Can cause network congestion |
| **Low overhead** | 8-byte header vs 20+ bytes for TCP | Less bandwidth overhead, faster |
| **Fast** | Minimal latency | Best for latency-sensitive applications |
| **Message-oriented** | Preserves message boundaries | Application receives complete messages |
| **Multicast support** | One-to-many communication | Efficient for streaming, discovery |

**When to Use UDP:**
- **Real-time streaming:** Video, audio (can tolerate loss, latency critical)
- **Online gaming:** Low latency prioritized, can handle some loss
- **DNS queries:** Fast lookups, small queries
- **IoT telemetry:** Frequent small messages, can tolerate loss
- **VoIP:** Real-time voice, latency critical
- **Broadcast/Multicast:** Network discovery, streaming
- **QUIC/HTTP/3:** Modern web protocol over UDP
- **Any application where speed is more important than reliability**

**UDP Header Structure (8 bytes):**

```
 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|     Source      |   Destination   |
|      Port       |      Port       |
+--------+--------+--------+--------+
|                 |                 |
|     Length      |    Checksum     |
+--------+--------+--------+--------+
```

**Key UDP Fields Explained:**

| Field | Size | Purpose | System Design Relevance |
|-------|------|---------|--------------------------|
| **Source Port** | 16 bits | Sender's port (optional) | Identifies sender, can be 0 if not needed |
| **Destination Port** | 16 bits | Receiver's port | Identifies which application receives data |
| **Length** | 16 bits | UDP header + data length | Includes 8-byte header |
| **Checksum** | 16 bits | Error detection (optional in IPv4) | Detects corrupted packets; mandatory in IPv6 |

**UDP vs TCP Overhead Comparison:**
```
Sending 100 bytes of data:

TCP:
- Header: 20 bytes (minimum)
- Data: 100 bytes
- Total: 120 bytes
- Overhead: 16.7%

UDP:
- Header: 8 bytes
- Data: 100 bytes
- Total: 108 bytes
- Overhead: 7.4%

For small packets, UDP overhead is significantly lower.
```

**Real-World UDP Use Cases:**

| Application | Why UDP? | System Design Considerations |
|-------------|----------|------------------------------|
| **DNS** | Fast queries, small data | Implement retry logic, fallback to TCP for large responses |
| **Video Streaming** | Latency critical, can tolerate loss | Implement buffering, adaptive bitrate |
| **Online Gaming** | Low latency, frequent updates | Implement interpolation, prediction, reconciliation |
| **VoIP** | Real-time, can tolerate loss | Implement jitter buffer, error concealment |
| **IoT Sensors** | Frequent small messages | Implement batching, retry logic |
| **QUIC (HTTP/3)** | Modern web, needs reliability | QUIC adds reliability on top of UDP |
| **NTP** | Time synchronization, small data | Multiple servers for accuracy |

### TCP vs UDP Comparison

**Why this matters for System Design:**
Choosing between TCP and UDP is one of the most important architectural decisions. The choice affects:
- Performance and latency characteristics
- Reliability guarantees (or lack thereof)
- Complexity of application code
- Scalability under load
- User experience

| Feature | TCP | UDP | System Design Decision |
|---------|-----|-----|------------------------|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless (send immediately) | TCP: 1 RTT overhead; UDP: zero overhead |
| **Reliability** | Guaranteed delivery via ACKs | Best-effort (no ACKs) | TCP: Use when data loss unacceptable |
| **Ordering** | Ordered delivery via sequence numbers | No ordering | TCP: Data arrives in order; UDP: app must order |
| **Speed** | Slower (20-60 byte header + overhead) | Faster (8-byte header, minimal overhead) | UDP: Use for latency-sensitive apps |
| **Header Size** | 20-60 bytes (variable) | 8 bytes (fixed) | UDP: Better for small packets |
| **Congestion Control** | Yes (slow start, congestion avoidance) | No | TCP: Better for shared networks; UDP can flood |
| **Flow Control** | Yes (sliding window) | No | TCP: Prevents overwhelming receiver |
| **Broadcasting** | Not supported | Supported (unicast, multicast, anycast) | UDP: Use for discovery, streaming |
| **Message Boundaries** | Byte-stream (no boundaries) | Message-oriented (preserves boundaries) | UDP: Easier for discrete messages |
| **Error Detection** | Checksum + retransmission | Checksum only (optional in IPv4) | TCP: Automatic recovery; UDP: app must handle |
| **Setup Time** | 1 RTT (handshake) | 0 RTT | UDP: Better for short-lived connections |
| **Resource Usage** | Higher (connection state, buffers) | Lower (minimal state) | UDP: Better for many clients |
| **Use Case** | File transfer, web, email, databases | Streaming, gaming, DNS, VoIP | Match protocol to requirements |

**Decision Tree - TCP vs UDP:**
```
Need to send data
│
├─ Is data loss acceptable?
│  ├─ Yes → Can you tolerate some latency?
│  │           ├─ Yes → Consider UDP (gaming, streaming)
│  │           └─ No → Consider TCP with optimizations
│  └─ No → Use TCP (file transfer, databases, web)
│
├─ Is ordering important?
│  ├─ Yes → Use TCP (or add ordering to UDP)
│  └─ No → UDP acceptable
│
├─ Is latency critical?
│  ├─ Yes → Consider UDP (real-time applications)
│  └─ No → TCP acceptable
│
├─ Need to send to multiple receivers?
│  ├─ Yes → UDP (multicast/broadcast)
│  └─ No → Both support unicast
│
└─ Need congestion control?
   ├─ Yes → TCP (or implement in UDP)
   └─ No → UDP acceptable
```

**Real-World Examples - Protocol Selection:**

**Example 1: Video Streaming Service**
```
Requirements:
- Low latency (real-time)
- Can tolerate some packet loss
- High bandwidth
- Many simultaneous viewers

Decision: UDP
- Use UDP for video stream
- Implement application-level reliability for key frames
- Use adaptive bitrate based on loss
- Buffer at client to smooth playback
```

**Example 2: File Transfer Service**
```
Requirements:
- Zero data loss
- Ordering critical
- Reliability over speed
- Large files

Decision: TCP
- Use TCP for guaranteed delivery
- Implement resume capability
- Use compression to reduce transfer time
- Parallel connections for large files
```

**Example 3: Online Multiplayer Game**
```
Requirements:
- Ultra-low latency (sub-100ms)
- Frequent small updates
- Can tolerate some loss
- Position updates more critical than chat

Decision: Mixed
- UDP for game state updates (position, actions)
- TCP for chat, inventory, transactions
- Implement interpolation/prediction in UDP
- Implement reconciliation for mismatched states
```

### TCP Connection Lifecycle

**Why it matters for System Design:**
- TCP handshake adds latency (1 RTT for setup, 1 RTT for teardown)
- Understanding connection states helps debug connection issues
- TIME_WAIT state can exhaust available ports under high load
- Connection pooling reduces handshake overhead
- Keep-alive connections improve performance
- Connection errors indicate specific problems based on state

**Three-Way Handshake (Connection Establishment):**

```
Client                              Server
   |                                   |
   | --------- SYN (seq=x) ----------> |  Step 1: Client wants to connect
   |                                   |         (SYN flag set, ISN = x)
   |                                   |
   | <---- SYN-ACK (seq=y, ack=x+1) ---|  Step 2: Server accepts connection
   |                                   |         (SYN+ACK, ISN = y, ACK = x+1)
   |                                   |
   | --------- ACK (ack=y+1) --------> |  Step 3: Client confirms
   |                                   |         (ACK, ACK = y+1)
   |                                   |
   |        [Connection Established]   |
   |                                   |
   | <--- Data can now flow both ways --> |
```

**Detailed Handshake Explanation:**

**Step 1 - SYN (Synchronize):**
- Client sends TCP segment with SYN flag set
- Includes Initial Sequence Number (ISN) - randomly chosen for security
- Client state: SYN_SENT
- Purpose: Initiate connection and inform server of client's sequence numbers

**Step 2 - SYN-ACK:**
- Server responds with SYN and ACK flags set
- Server chooses its own ISN
- ACK number = client's ISN + 1 (confirming receipt)
- Server state: SYN_RCVD
- Purpose: Accept connection and inform client of server's sequence numbers

**Step 3 - ACK:**
- Client sends ACK flag set
- ACK number = server's ISN + 1
- Both sides: ESTABLISHED
- Purpose: Confirm receipt of server's SYN

**Why Random ISN?**
- Prevents sequence number prediction attacks
- Prevents old packets from being accepted as new
- ISN generation uses cryptographic random number generators

**Four-Way Handshake (Connection Termination):**

```
Client                              Server
   |                                   |
   | --------- FIN -------------------> |  Step 1: Client wants to close
   |                                   |         (FIN flag set)
   |                                   |
   | <--------- ACK ------------------- |  Step 2: Server acknowledges
   |                                   |         (ACK, ACK = seq+1)
   |                                   |
   | <--------- FIN ------------------- |  Step 3: Server wants to close
   |                                   |         (FIN flag set)
   |                                   |
   | --------- ACK -------------------> |  Step 4: Client acknowledges
   |                                   |         (ACK, ACK = seq+1)
   |                                   |
   |        [Connection Closed]        |
```

**Detailed Termination Explanation:**

**Step 1 - FIN (Finish):**
- Client sends segment with FIN flag set
- Indicates client has no more data to send
- Client state: FIN_WAIT_1
- Client can still receive data from server

**Step 2 - ACK:**
- Server acknowledges receipt of FIN
- Server state: CLOSE_WAIT
- Server can still send data to client

**Step 3 - FIN:**
- Server sends its own FIN when done
- Server state: LAST_ACK
- Indicates server has no more data to send

**Step 4 - ACK:**
- Client acknowledges server's FIN
- Client state: TIME_WAIT (2MSL)
- Server state: CLOSED
- Client waits 2MSL before closing

**TCP State Diagram:**
```
                        LISTEN
                           |
                SYN       | SYN+ACK
                           ↓
SYN_SENT ------------→ SYN_RCVD ←───────┐
   |                   /|                |
   |         SYN+ACK  / | ACK            |
   |                 /  |                |
   |                /   |                |
   |               ↓    |                |
   |            ESTABLISHED ←─────────────┘
   |               |
   |               | FIN
   |               ↓
   |            FIN_WAIT_1
   |               |
   |        ACK   | FIN
   |        or    ↓
   |        FIN  FIN_WAIT_2
   |               |
   |               | FIN
   |               ↓
   |            TIME_WAIT
   |               |
   |            2MSL ↓
   |               |
   └────────→ CLOSED

Simultaneous Close:
Both sides send FIN at same time → CLOSING → TIME_WAIT → CLOSED

Passive Close (server initiated):
ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

**Important TCP States Explained:**

| State | Description | System Design Relevance |
|-------|-------------|------------------------|
| **CLOSED** | Connection not active | Initial state, final state |
| **LISTEN** | Waiting for connection requests | Server waiting for clients |
| **SYN_SENT** | SYN sent, waiting for SYN-ACK | Client attempting to connect |
| **SYN_RCVD** | SYN received, sent SYN-ACK | Server waiting for final ACK |
| **ESTABLISHED** | Connection ready for data transfer | Normal operating state |
| **FIN_WAIT_1** | FIN sent, waiting for ACK | Client closing, first phase |
| **FIN_WAIT_2** | ACK received, waiting for FIN | Client closing, second phase |
| **CLOSE_WAIT** | FIN received, waiting to close | Server received close, can still send |
| **CLOSING** | Both sides sent FIN simultaneously | Rare, simultaneous close |
| **LAST_ACK** | FIN sent, waiting for final ACK | Server closing, waiting for confirmation |
| **TIME_WAIT** | Both sides closed, waiting | Critical for preventing issues |

**TIME_WAIT State - Critical Details:**

**Purpose:**
1. Ensures final ACK is received (retransmit if lost)
2. Prevents old duplicate segments from being accepted
3. Allows delayed segments to arrive and be discarded

**Duration:** 2MSL (Maximum Segment Lifetime)
- MSL is typically 30-120 seconds (default 60s)
- TIME_WAIT = 2 × 60 = 120 seconds
- Can be tuned but not recommended

**System Design Impact:**
```
Problem: Port Exhaustion
- Each connection uses a unique 4-tuple (src_ip, src_port, dst_ip, dst_port)
- TIME_WAIT holds port for 2 minutes
- High connection rate can exhaust ephemeral ports

Example:
- 10,000 requests/second
- Each request uses new connection (no keep-alive)
- Each connection stays in TIME_WAIT for 120s
- Total ports in TIME_WAIT = 10,000 × 120 = 1.2M
- Ephemeral port range: ~28,000 (49152-65535)
- Result: Port exhaustion, connection failures

Solutions:
1. Use connection pooling/keep-alive
2. Increase ephemeral port range
3. Reduce TIME_WAIT (not recommended)
4. Use SO_REUSEADDR (careful with implications)
5. Load balance across multiple IPs
```

**Real-World Example - Connection Pooling:**
```
Without Pooling:
Request 1: TCP handshake → Data → Close → TIME_WAIT
Request 2: TCP handshake → Data → Close → TIME_WAIT
Request 3: TCP handshake → Data → Close → TIME_WAIT
...
Overhead: 1 RTT per request + TIME_WAIT accumulation

With Pooling:
Connection 1: TCP handshake → Data → Keep Alive → Data → Keep Alive → ...
Request 1: Use existing connection
Request 2: Use existing connection
Request 3: Use existing connection
Overhead: 1 RTT for initial connection only
Benefits: Reduced latency, fewer ports used, better performance
```

### TCP Flow Control and Congestion Control

**Why it matters for System Design:**
- Flow control prevents overwhelming receivers (crucial for microservices)
- Congestion control affects throughput under load
- TCP window size tuning can improve performance for high-latency connections
- Understanding these helps debug performance bottlenecks
- Different congestion algorithms behave differently (choose wisely)
- Buffer bloat can cause latency issues

**Flow Control (Receiver-Side):**

Flow control prevents the sender from overwhelming the receiver's buffer. It's a mechanism to match sender speed with receiver processing capability.

**How it works:**
1. Receiver advertises available buffer space in Window Size field
2. Sender can only send unacknowledged data up to window size
3. Receiver updates window size as buffer space frees up
4. Zero window means sender must wait (sender probes periodically)

**Sliding Window Mechanism:**
```
Sender's Perspective:
[Sent & ACKed] [Sent, Not ACKed] [Can Send] [Can't Send Yet]
                <-------- Window Size --------->
                
Example:
- Sequence numbers: 1-1000
- Window size: 500
- Sent & ACKed: 1-300
- Sent, Not ACKed: 301-500
- Can Send: 501-800
- Can't Send Yet: 801-1000

When 301-400 is ACKed:
- Window slides right
- Can Send: 501-900
```

**Window Size Field:**
- 16 bits in TCP header
- Maximum value: 65,535 bytes
- Window scaling option (RFC 1323) allows values up to 1GB
- rwnd = receiver's advertised window
- cwnd = congestion window (sender's limit)
- Actual window = min(rwnd, cwnd)

**Zero Window Problem:**
```
Receiver buffer full → Window = 0
Sender stops sending
Sender sends zero-window probe periodically
Receiver responds with updated window when space available

System Design Impact:
- Slow receiver can stall entire connection
- Backpressure propagates through system
- Need to size buffers appropriately
```

**Congestion Control (Network-Side):**

Congestion control prevents network congestion collapse by adjusting sender's transmission rate based on network conditions.

**Why Congestion Control Matters:**
```
Without Congestion Control:
- All senders send as fast as possible
- Routers overflow buffers
- Packets dropped everywhere
- Network becomes unusable
- Congestion collapse

With Congestion Control:
- Senders detect congestion
- Senders reduce transmission rate
- Network recovers
- Fair sharing of bandwidth
```

**TCP Congestion Control Algorithms:**

**1. Slow Start:**
```
Purpose: Quickly ramp up transmission rate when connection starts

Behavior:
- Start with cwnd = 1 MSS (Maximum Segment Size)
- Double cwnd for each ACK received (exponential growth)
- After 1 RTT: cwnd = 2 MSS
- After 2 RTT: cwnd = 4 MSS
- After 3 RTT: cwnd = 8 MSS
- Continue until cwnd >= ssthresh (slow start threshold) or loss detected

Example:
MSS = 1460 bytes
RTT = 100ms

Time 0ms:    cwnd = 1 MSS, send 1460 bytes
Time 50ms:   ACK received, cwnd = 2 MSS, send 2920 bytes
Time 100ms:  ACKs received, cwnd = 4 MSS, send 5840 bytes
Time 150ms:  ACKs received, cwnd = 8 MSS, send 11680 bytes
...
```

**2. Congestion Avoidance:**
```
Purpose: Gently increase transmission rate to avoid causing congestion

Behavior:
- Activated when cwnd >= ssthresh
- Increase cwnd by 1 MSS per RTT (additive increase)
- More precisely: cwnd += MSS * (MSS / cwnd) per ACK
- Linear growth instead of exponential

Example:
MSS = 1460 bytes
ssthresh = 16 MSS
Current cwnd = 16 MSS

After 1 RTT: cwnd = 17 MSS
After 2 RTT: cwnd = 18 MSS
After 3 RTT: cwnd = 19 MSS
...
Growth is slow but steady
```

**3. Fast Retransmit:**
```
Purpose: Detect and recover from packet loss faster than timeout

Behavior:
- Receiver sends duplicate ACK for each out-of-order segment
- 3 duplicate ACKs = packet loss detected
- Sender retransmits lost packet immediately (don't wait for RTO)
- Indicates network still functioning (ACKs getting through)

Why 3 duplicate ACKs?
- Not all packet loss means congestion
- Could be random error
- 3 ACKs suggests persistent problem

Example:
Sender sends: 1, 2, 3, 4, 5
Receiver gets: 1, 2, 4, 5 (3 lost)
Receiver sends: ACK(2), ACK(2), ACK(2), ACK(2)
Sender detects 3 duplicate ACKs
Sender retransmits 3 immediately
```

**4. Fast Recovery:**
```
Purpose: Quickly recover from packet loss without going to slow start

Behavior (Traditional TCP Reno):
- On 3 duplicate ACKs:
  1. Set ssthresh = cwnd / 2
  2. Set cwnd = ssthresh + 3 MSS
  3. Retransmit lost packet
  4. Enter fast recovery state
- For each additional duplicate ACK: cwnd += 1 MSS
- On ACK for new data: cwnd = ssthresh (enter congestion avoidance)
- On timeout: ssthresh = cwnd / 2, cwnd = 1 MSS (enter slow start)

Why not slow start?
- Network still functioning (ACKs getting through)
- Congestion not severe
- Slow start would be too conservative
```

**Congestion Control Variants:**

| Algorithm | Description | Growth Behavior | Best For |
|-----------|-------------|-----------------|----------|
| **Tahoe** | Original TCP | Slow start after any loss | Simple, conservative |
| **Reno** | Adds fast recovery | Fast recovery after 3 dup ACKs | General purpose |
| **NewReno** | Improved Reno | Better handling of multiple losses | Modern networks |
| **CUBIC** | Default in Linux | Cubic function of time | High-bandwidth, long RTT |
| **BBR** | Google's algorithm | Model-based, not loss-based | Variable networks, cellular |
| **Vegas** | RTT-based | Adjust based on RTT increase | Low-latency networks |

**CUBIC (Default in Linux):**
```
Characteristics:
- Uses cubic function for window growth
- More aggressive than Reno for long RTT
- Better for high-bandwidth, long RTT networks
- Fair to other flows

Window Growth:
W(t) = C × (t - K)^3 + W_max

Where:
- C = scaling constant
- t = time since last loss
- K = time to reach W_max from W_max/2
- W_max = window size before last loss

Advantages:
- Scalable to 10Gbps+ networks
- Fairness among flows
- Stable in steady state
```

**BBR (Bottleneck Bandwidth and RTT):**
```
Characteristics:
- Model-based, not loss-based
- Measures actual bandwidth and RTT
- Doesn't use packet loss as congestion signal
- Google's algorithm, open-sourced

Key Insight:
- Loss ≠ congestion
- Modern networks have large buffers (bufferbloat)
- Loss-based algorithms fill buffers, causing latency

How BBR Works:
1. Measure BDP (Bandwidth-Delay Product)
2. Set sending rate to BDP
3. Continuously probe for higher bandwidth
4. Reduce RTT by not filling buffers

Advantages:
- Higher throughput on lossy networks
- Lower latency (doesn't fill buffers)
- Better for cellular networks

Disadvantages:
- Can be unfair to loss-based flows
- More complex to implement
```

**Real-World Example - TCP Tuning for High Performance:**
```
Scenario: High-speed data transfer over 100ms RTT link

Default TCP:
- Window size: 64KB (default)
- BDP = Bandwidth × RTT = 1Gbps × 100ms = 12.5MB
- Window size (64KB) << BDP (12.5MB)
- Result: Link underutilized, slow transfer

Tuned TCP:
- Window size: 16MB (window scaling enabled)
- BDP = 12.5MB
- Window size (16MB) > BDP (12.5MB)
- Result: Link fully utilized, fast transfer

Linux Tuning Commands:
sysctl -w net.ipv4.tcp_window_scaling=1
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem='4096 87380 16777216'
sysctl -w net.ipv4.tcp_wmem='4096 65536 16777216'
```

---

## 4. Application Layer Protocols

### HTTP/1.1

**HTTP/1.1** (1997, RFC 2616) introduced persistent connections and pipelining to address performance issues in HTTP/1.0.

**Why it matters for System Design:**
- HTTP/1.1 is still widely used and supported
- Understanding its limitations helps explain why HTTP/2 and HTTP/3 exist
- Persistent connections are crucial for performance
- Keep-alive settings directly affect resource usage
- Head-of-line blocking is a key limitation to understand

**Key Features:**

| Feature | Description | System Design Impact |
|---------|-------------|----------------------|
| **Persistent Connections** | Reuse TCP connection for multiple requests | Reduces TCP handshake overhead, improves latency |
| **Pipelining** | Send multiple requests without waiting for responses | Theoretically improves performance, rarely used |
| **Chunked Transfer Encoding** | Stream data of unknown size | Enables streaming responses without knowing size upfront |
| **Host Header** | Required for virtual hosting | Enables multiple domains on one IP |
| **Cache Control Headers** | `Cache-Control`, `ETag`, `Last-Modified` | Reduces server load, improves user experience |
| **Range Requests** | Request partial content | Enables resumable downloads, video seeking |
| **Upgrade Header** | Protocol upgrade mechanism | Enables WebSocket, HTTP/2 negotiation |

**Persistent Connections (Keep-Alive):**
```
HTTP/1.0 (without Keep-Alive):
Request 1: TCP handshake → HTTP request → response → close
Request 2: TCP handshake → HTTP request → response → close
Request 3: TCP handshake → HTTP request → response → close
Overhead: 3 TCP handshakes (3 RTTs)

HTTP/1.1 (with Keep-Alive):
TCP handshake → Request 1 → response → Request 2 → response → Request 3 → response → close
Overhead: 1 TCP handshake (1 RTT)

Benefits:
- Reduced latency (fewer handshakes)
- Reduced CPU usage (fewer TLS handshakes)
- Better TCP performance (connection warm-up)

Configuration:
Connection: keep-alive (default in HTTP/1.1)
Keep-Alive: timeout=5, max=100
```

**Limitations of HTTP/1.1:**

| Limitation | Description | Impact on System Design |
|------------|-------------|------------------------|
| **Head-of-Line Blocking** | One slow response blocks subsequent requests | Limits parallelism, hurts performance |
| **Uncompressed Headers** | Redundant data sent repeatedly | Wastes bandwidth, especially on mobile |
| **Text-Based Protocol** | Inefficient parsing | Higher CPU usage, slower processing |
| **One Request at a Time** | Without pipelining, requests are sequential | Limits throughput, requires multiple connections |
| **Pipelining Issues** | Rarely implemented correctly | Can't rely on pipelining for performance |
| **No Server Push** | Client must request each resource | Slower page load times |

**Head-of-Line Blocking (HOL) Explained:**
```
Scenario: Client requests 3 resources (A, B, C) on same connection

Timeline:
Client sends: A, B, C (pipelined)

Case 1 - All fast:
Server responds: A (10ms), B (10ms), C (10ms)
Total: 30ms

Case 2 - A is slow:
Server responses: A (1000ms), B (10ms), C (10ms)
Total: 1020ms

Problem: B and C are blocked waiting for A
Solution: Use multiple connections (but wastes resources)
HTTP/2 Solution: Multiplexing (no blocking)
```

**Real-World Example - HTTP/1.1 Optimization:**
```
Web Application Optimization:

Problem:
- Page loads 100 resources (CSS, JS, images)
- HTTP/1.1 serializes requests
- HOL blocking causes slow load times

Solution 1: Multiple Connections
- Open 6 connections to domain (browser limit)
- Parallelize requests across connections
- Trade-off: More TCP connections, more server resources

Solution 2: Domain Sharding
- Use multiple subdomains: static1.example.com, static2.example.com
- Browsers open 6 connections per domain
- Effectively 12+ parallel connections
- Trade-off: Complex configuration, DNS overhead

Solution 3: Resource Bundling
- Bundle CSS files into one
- Bundle JS files into one
- Use image sprites
- Trade-off: Less granular caching, larger downloads

Solution 4: HTTP/2
- Single connection with multiplexing
- No HOL blocking
- Header compression
- Trade-off: Requires server/browser support
```

### HTTP/2

**HTTP/2** (2015, RFC 7540) addresses HTTP/1.1 performance issues using binary framing and multiplexing. Originated from Google's SPDY protocol.

**Why it matters for System Design:**
- HTTP/2 significantly improves web performance
- Multiplexing eliminates head-of-line blocking
- Header compression reduces bandwidth usage
- Server push can reduce latency further
- Understanding HTTP/2 helps optimize web applications
- Most modern browsers and servers support HTTP/2

**Key Features:**

| Feature | Description | System Design Impact |
|---------|-------------|----------------------|
| **Binary Protocol** | Binary framing instead of text | More efficient parsing, fewer errors |
| **Multiplexing** | Multiple streams on single connection | Eliminates HOL blocking, fewer connections needed |
| **Header Compression (HPACK)** | Huffman encoding + static/dynamic tables | Reduces bandwidth, especially on mobile |
| **Server Push** | Server proactively sends resources | Reduces RTTs for critical resources |
| **Stream Prioritization** | Client sets stream dependencies | Better resource allocation |
| **Flow Control** | Per-stream and connection-level flow control | Prevents overwhelming receiver |
| **Security Requirement** | Requires TLS in practice (browsers enforce) | HTTPS is mandatory for HTTP/2 |

**1. Binary Protocol:**
```
HTTP/1.1 (Text-based):
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0


Problems:
- Inefficient parsing
- Text parsing errors
- Hard to implement correctly

HTTP/2 (Binary):
Frame header (9 bytes) + Frame payload
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Length (24)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                        |
+-+-------------------------------------------------------------+
|                   Frame Payload (0...)                       ...

Benefits:
- Efficient binary parsing
- No ambiguity in parsing
- Easier to implement correctly
- Enables multiplexing
```

**2. Multiplexing:**
```
HTTP/1.1 (Serial):
Connection 1: Request A → Response A → Request B → Response B
Connection 2: Request C → Response C

HTTP/2 (Multiplexed):
Connection 1:
  Stream 1: Request A → Response A
  Stream 3: Request B → Response B
  Stream 5: Request C → Response C
  (All interleaved on single connection)

Benefits:
- Single TCP connection for all requests
- No HOL blocking at HTTP layer
- Better TCP utilization
- Fewer server resources

Stream States:
IDLE → RESERVED (local/remote) → OPEN → HALF-CLOSED → CLOSED
```

**3. Header Compression (HPACK):**
```
Problem: Headers are repetitive
HTTP/1.1:
Request 1: User-Agent: Mozilla/5.0..., Accept: text/html, ...
Request 2: User-Agent: Mozilla/5.0..., Accept: text/html, ...
Request 3: User-Agent: Mozilla/5.0..., Accept: text/html, ...

Solution: HPACK Compression

Static Table (pre-defined):
:method: GET
:scheme: https
:path: /
:authority: example.com
user-agent: (empty)
accept: */*
...

Dynamic Table (learned during connection):
After Request 1: user-agent, accept, etc.
After Request 2: Reference dynamic table entries
After Request 3: Reference dynamic table entries

Compression Example:
Original: User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Compressed: [Index 42] (reference to dynamic table)

Benefits:
- 50-90% header size reduction
- Especially beneficial on mobile
- Faster page loads
```

**4. Server Push:**
```
Scenario: Client requests index.html

Without Server Push:
Client: GET /index.html
Server: index.html
Client parses, discovers style.css
Client: GET /style.css
Server: style.css
Client parses, discovers script.js
Client: GET /script.js
Server: script.js
Total: 3 RTTs

With Server Push:
Client: GET /index.html
Server: index.html + PUSH_PROMISE(style.css) + style.css
Server: PUSH_PROMISE(script.js) + script.js
Total: 1 RTT

Benefits:
- Reduced latency
- Fewer round trips
- Better perceived performance

Caveats:
- Can push resources client already has
- Client can cancel pushed streams
- Deprecated in HTTP/3 (use preload hints instead)
```

**5. Stream Prioritization:**
```
Stream Dependencies:
Stream 1 (index.html)
  ├─ Stream 3 (style.css, weight 16)
  └─ Stream 5 (script.js, weight 1)

Meaning:
- Stream 1 is parent
- Stream 3 and 5 depend on 1
- Stream 3 gets 16x more bandwidth than 5
- Resources are loaded in dependency order

Use Case:
- CSS should load before JS
- Critical CSS before non-critical
- Above-the-fold content first
```

**HTTP/2 Frame Types:**

| Frame Type | Purpose | System Design Relevance |
|------------|---------|------------------------|
| **DATA** | HTTP payload | Carries actual response/request body |
| **HEADERS** | HTTP headers | Opens/ends streams, carries metadata |
| **PRIORITY** | Stream priority | Sets dependencies and weights |
| **RST_STREAM** | Stream termination | Cancel requests, handle errors |
| **SETTINGS** | Configuration exchange | Negotiate connection parameters |
| **PUSH_PROMISE** | Server push | Indicate pushed resources |
| **PING** | Round-trip measurement | Health checks, keep-alive |
| **GOAWAY** | Connection termination | Graceful shutdown, error signaling |
| **WINDOW_UPDATE** | Flow control | Update window size |
| **CONTINUATION** | Continue headers | Split large header blocks |

**HTTP/2 Connection Lifecycle:**
```
Client                              Server
   |                                   |
   | --------- TCP Handshake --------> |  1 RTT
   |                                   |
   | --------- TLS Handshake --------> |  1-2 RTTs
   |         (ALPN negotiates h2)        |
   |                                   |
   | ===== HTTP/2 Connection Preface ==> |
   |         "PRI * HTTP/2.0\r\n\r\n..." |
   |         (magic string)             |
   |                                   |
   | <------- SETTINGS ---------------- |
   | ------- SETTINGS ---------------> |  Exchange settings
   | <------- SETTINGS ACK ----------- |
   |                                   |
   | === Multiplexed Streams begin === |
   |  Stream 1: HEADERS (GET /index.html) |
   |  Stream 3: HEADERS (GET /style.css) |
   |  Stream 5: HEADERS (GET /script.js) |
   |                                   |
   | <------- DATA (Stream 1) --------- |
   | <------- DATA (Stream 3) --------- |
   | <------- DATA (Stream 5) --------- |
   |                                   |
   | ------- END_STREAM (all) -------> |
   |                                   |
   | ------- GOAWAY -------------------> |  Connection close
```

**Real-World Example - HTTP/2 Performance Improvement:**
```
E-commerce Website:

HTTP/1.1:
- 100 resources per page
- 6 parallel connections (browser limit)
- ~17 round trips per connection
- Total: ~17 RTTs
- With 100ms RTT: 1.7 seconds

HTTP/2:
- 100 resources per page
- 1 connection with multiplexing
- All requests sent in first RTT
- Total: ~2-3 RTTs (TCP + TLS + data)
- With 100ms RTT: 200-300ms

Improvement: 5-8x faster page load

Configuration (Nginx):
server {
    listen 443 ssl http2;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
}

Verification:
curl -I --http2 https://example.com
```

### HTTP/3 and QUIC

**HTTP/3** (2022, RFC 9114) uses QUIC instead of TCP+TLS, eliminating TCP head-of-line blocking. QUIC was developed by Google and standardized by IETF.

**Why it matters for System Design:**
- HTTP/3 eliminates TCP-level head-of-line blocking entirely
- Connection migration supports mobile devices switching networks
- 0-RTT connection resumption dramatically reduces latency
- Better performance on lossy networks (cellular, satellite)
- Future of web protocol, increasingly adopted
- Understanding QUIC helps design high-performance systems

**QUIC (Quick UDP Internet Connections) Characteristics:**

| Feature | Description | System Design Impact |
|---------|-------------|----------------------|
| **Built on UDP** | Uses UDP as transport layer | Bypasses TCP limitations, works on port 443 |
| **Built-in Encryption** | TLS 1.3 integrated into protocol | Encryption is mandatory, no separate TLS handshake |
| **0-RTT Resumption** | Send data with first packet | Dramatically reduces latency for repeat visits |
| **Connection Migration** | Survives IP/port changes | Seamless mobile network handoffs |
| **Stream Multiplexing** | Independent streams without HOL blocking | No blocking at any layer |
| **Improved Congestion Control** | Pluggable congestion control | Better performance on variable networks |
| **Connection ID** | 64-bit connection identifier | NAT-friendly, connection migration |

**Why QUIC Uses UDP:**
```
Problem with TCP:
- TCP is implemented in OS kernel
- Hard to modify TCP (requires OS updates)
- TCP head-of-line blocking is fundamental

Solution: QUIC on UDP:
- UDP is simple and widely supported
- QUIC implements reliability in user space
- Easy to update (application layer)
- Can fix TCP limitations without OS changes
- Eliminates HOL blocking at TCP layer
```

**HTTP/3 vs HTTP/2 vs HTTP/1.1:**

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **Transport** | TCP | TCP | QUIC (UDP) |
| **Encryption** | Optional TLS | TLS 1.2+ (required in practice) | Built-in TLS 1.3 |
| **Multiplexing** | No (sequential) | Yes (streams) | Yes (streams) |
| **HOL Blocking** | TCP + HTTP | TCP only | None |
| **Header Compression** | None | HPACK | QPACK |
| **Server Push** | No | Yes | Deprecated (use preload) |
| **Connection Migration** | No | No | Yes |
| **Handshake RTTs** | 2-3 (TCP + TLS) | 2-3 (TCP + TLS) | 0-1 (0-RTT) |
| **Binary Protocol** | No | Yes | Yes |

**Connection Establishment Comparison:**
```
HTTP/1.1 (First Request):
TCP handshake: 1 RTT
TLS handshake: 1-2 RTTs
HTTP request: 1 RTT
Total: 3-4 RTTs

HTTP/2 (First Request):
TCP handshake: 1 RTT
TLS handshake (with ALPN): 1-2 RTTs
HTTP/2 preface + settings: 1 RTT
HTTP request: 1 RTT
Total: 4-5 RTTs

HTTP/3 (First Request, New Connection):
QUIC handshake: 1 RTT (includes TLS 1.3)
HTTP request: 1 RTT
Total: 2 RTTs

HTTP/3 (First Request, Resumed Connection):
QUIC 0-RTT: 0 RTTs (data in first packet)
Total: 0-1 RTTs

Example (100ms RTT):
HTTP/1.1: 300-400ms
HTTP/2: 400-500ms
HTTP/3 (new): 200ms
HTTP/3 (resumed): 0-100ms
```

**0-RTT Connection Resumption:**
```
First Visit:
Client → Server: ClientHello (no data)
Server → Client: ServerHello, Certificate, Finished
Client → Server: Finished, HTTP Request
Total: 2 RTTs

Subsequent Visits (with PSK):
Client → Server: ClientHello + PSK + HTTP Request (0-RTT)
Server → Client: ServerHello + Response
Total: 0-1 RTTs

Benefits:
- Dramatically reduced latency
- Better user experience for repeat visitors
- Critical for mobile apps

Trade-offs:
- Slightly less forward secrecy on 0-RTT data
- Server must validate 0-RTT data carefully
- Replay attacks possible (need anti-replay)
```

**Connection Migration:**
```
Scenario: Mobile user on cellular network switches to WiFi

HTTP/1.1 / HTTP/2:
- Connection breaks on network change
- Must establish new connection
- In-flight data lost
- User experience: glitch or pause

HTTP/3:
- Connection survives network change
- Connection ID remains same
- In-flight data continues
- User experience: seamless

How it works:
1. Client has connection ID: 0x1234
2. Network changes (IP: 10.0.0.1 → 192.168.1.1)
3. Client sends packet with connection ID 0x1234 from new IP
4. Server recognizes connection ID, updates mapping
5. Connection continues seamlessly

Use Cases:
- Mobile devices switching networks
- Load balancer failover
- NAT rebinding
```

### HTTPS and TLS/SSL

**HTTPS** is HTTP over TLS (Transport Layer Security), providing encryption, authentication, and integrity. SSL (Secure Sockets Layer) was the predecessor to TLS.

**Why it matters for System Design:**
- HTTPS is mandatory for modern web applications (security, SEO, browser warnings)
- TLS handshake adds latency (1-2 RTTs)
- TLS termination affects load balancer design
- Certificate management is operational overhead
- TLS 1.3 significantly reduces handshake latency
- Understanding TLS helps debug connection issues

**TLS/SSL Evolution:**

| Version | Year | Status | Key Features |
|---------|------|--------|--------------|
| SSL 1.0 | 1994 | Never released | Proprietary Netscape |
| SSL 2.0 | 1995 | Deprecated | First public version |
| SSL 3.0 | 1996 | Deprecated (POODLE attack) | Improvements over SSL 2.0 |
| TLS 1.0 | 1999 | Deprecated | Based on SSL 3.0 |
| TLS 1.1 | 2006 | Deprecated | Minor improvements |
| TLS 1.2 | 2008 | Widely supported | Cipher suite flexibility, SHA-256 |
| TLS 1.3 | 2018 | Recommended | 1-RTT handshake, 0-RTT resumption, improved security |

**Security Recommendations:**
```
Current Best Practices:
- Disable SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1
- Use TLS 1.2 minimum, TLS 1.3 preferred
- Use strong cipher suites only
- Enable HSTS (HTTP Strict Transport Security)
- Use perfect forward secrecy (ECDHE key exchange)
- Use certificate pinning for mobile apps
- Implement certificate rotation
```

**TLS 1.2 Handshake (Full):**
```
Client                              Server
   |                                   |
   | -------- ClientHello ---------->  |  1. Client sends supported versions
   |  - TLS versions (1.0, 1.1, 1.2)   |     cipher suites, random, extensions
   |  - Cipher suites (ordered preference)|
   |  - Random (32 bytes)              |
   |  - Session ID (for resumption)    |
   |  - Extensions (SNI, etc.)         |
   |                                   |
   | <------- ServerHello ------------ |  2. Server chooses version and cipher
   |  - Chosen version (TLS 1.2)       |
   |  - Chosen cipher suite            |
   |  - Server random (32 bytes)       |
   |  - Session ID (new or resumed)    |
   |                                   |
   | <------- Certificate ------------ |  3. Server proves identity
   |  - Server certificate chain       |
   |  - (optional intermediate CAs)    |
   |                                   |
   | <------- ServerHelloDone -------- |  4. Server done sending
   |                                   |
   | -------- ClientKeyExchange -----> |  5. Client generates pre-master secret
   |  - Pre-master secret (encrypted)    |     using server's public key
   |  - Encrypted with server's public key|
   |                                   |
   | -------- ChangeCipherSpec -------> |  6. Client switches to encryption
   | -------- Finished --------------> |  7. Client verifies handshake
   |  (encrypted with new keys)        |
   |                                   |
   | <------- ChangeCipherSpec ------- |  8. Server switches to encryption
   | <------- Finished --------------- |  9. Server verifies handshake
   |                                   |
   | ==== Encrypted Application Data === | 10. Secure communication begins

Total: 2 RTTs (ClientHello→ServerHello, ClientKeyExchange→Finished)
```

**TLS 1.3 Handshake (Simplified, 1-RTT):**
```
Client                              Server
   |                                   |
   | -------- ClientHello ---------->  |  1. Client sends supported versions
   |  - TLS versions (1.2, 1.3)       |     cipher suites, key_share, random
   |  - Cipher suites                  |
   |  - key_share (public keys for ECDHE) |
   |  - Random (32 bytes)              |
   |  - PSK (if resuming)              |
   |                                   |
   | <------- ServerHello ------------ |  2. Server responds with chosen params
   | <------- EncryptedExtensions ---- |
   | <------- Certificate ------------ |  3. Server certificate
   | <------- CertificateVerify ------ |
   | <------- Finished --------------- |
   |                                   |
   | -------- Finished --------------> |  4. Client verifies handshake
   |                                   |
   | ==== Encrypted Application Data === |  5. Secure communication begins

Total: 1 RTT (vs 2 RTTs in TLS 1.2)
Improvements:
- Combined ServerHello + Certificate + Finished
- Key exchange in ClientHello (0-RTT possible)
- Removed insecure features (compression, renegotiation)
- Only secure cipher suites allowed
```

**TLS 1.3 0-RTT (Resumption):**
```
First Visit (1-RTT):
Client → Server: ClientHello (no data)
Server → Client: ServerHello + Certificate + Finished
Client → Server: Finished + HTTP Request
Total: 1 RTT

Subsequent Visits (0-RTT):
Client → Server: ClientHello + PSK + HTTP Request (data in first packet)
Server → Client: ServerHello + Response
Total: 0-1 RTTs

Benefits:
- Dramatically reduced latency for repeat connections
- Better user experience
- Critical for mobile apps and high-frequency requests

Trade-offs:
- Slightly less forward secrecy on 0-RTT data
- Replay attacks possible (need anti-replay mechanisms)
- Server must validate 0-RTT data carefully
```

**Cipher Suites Explained:**

**Format:** `TLS_KeyExchange_Authentication_Encryption_MAC`

**TLS 1.2 Examples:**
```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
│  │    │    │   │     │     │
│  │    │    │   │     │     └─ HMAC-SHA256
│  │    │    │   │     └─ AES-GCM (128-bit)
│  │    │    │   └─ With
│  │    │    └─ RSA authentication
│  │    └─ ECDHE key exchange (ephemeral)
│  └─ TLS
└─ Protocol

Components:
- Key Exchange: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
  - Provides perfect forward secrecy
  - Ephemeral = new keys for each connection
- Authentication: RSA or ECDSA
  - Verifies server identity
- Encryption: AES-GCM or AES-CBC
  - AES-GCM is authenticated encryption (preferred)
- MAC: SHA-256 or SHA-384
  - Message authentication code
```

**TLS 1.3 Examples:**
```
TLS_AES_256_GCM_SHA384
│  │   │     │    │
│  │   │     │    └─ SHA-384
│  │   │     └─ AES-GCM (256-bit)
│  │   └─ With
│  └─ AES
└─ TLS

Simpler format (key exchange and auth integrated):
- AES_256_GCM_SHA384: AES-256 with GCM mode, SHA-384
- ChaCha20_Poly1305_SHA256: ChaCha20 stream cipher

All TLS 1.3 cipher suites provide:
- Perfect forward secrecy
- Authenticated encryption
- No weak algorithms
```

**Recommended Cipher Suites (2024):**

| Priority | TLS 1.3 Cipher Suite | TLS 1.2 Cipher Suite | Notes |
|----------|---------------------|---------------------|-------|
| 1 | TLS_AES_256_GCM_SHA384 | TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 | Strongest encryption |
| 2 | TLS_CHACHA20_POLY1305_SHA256 | TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 | Mobile-friendly |
| 3 | TLS_AES_128_GCM_SHA256 | TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 | Faster, still secure |

**Perfect Forward Secrecy (PFS):**
```
Without PFS:
- Server private key compromised
- Attacker can decrypt all past sessions
- Server private key is long-term

With PFS (ECDHE):
- Each session uses unique ephemeral keys
- Server private key compromise doesn't affect past sessions
- Only current/future sessions at risk

System Design Impact:
- PFS is critical for security
- Must use ECDHE or DHE key exchange
- RSA key exchange does NOT provide PFS
```

### WebSocket

**WebSocket** provides full-duplex, persistent communication over a single TCP connection. Standardized in RFC 6455 (2011).

**Why it matters for System Design:**
- WebSocket enables real-time bidirectional communication
- Essential for chat, notifications, collaborative apps
- Lower latency than HTTP polling
- Reduces server overhead (no repeated handshakes)
- Understanding WebSocket helps design real-time features
- WebSocket scaling requires special considerations

**Key Characteristics:**

| Characteristic | Description | System Design Impact |
|-----------------|-------------|----------------------|
| **Persistent connection** | Long-lived TCP connection | Reduces handshake overhead, consumes server resources |
| **Full-duplex** | Both directions simultaneously | Enables real-time bidirectional communication |
| **Low latency** | No repeated handshakes | Sub-millisecond latency for messages |
| **Binary and text frames** | Efficient data transfer | Supports various data types efficiently |
| **Subprotocol support** | Application-specific protocols | Enables custom protocols over WebSocket |
| **HTTP upgrade** | Starts as HTTP, upgrades to WebSocket | Works through most proxies/firewalls |

**WebSocket vs HTTP Polling:**
```
HTTP Polling (Short Polling):
Client: GET /messages
Server: [] (no messages)
Client: GET /messages
Server: [] (no messages)
Client: GET /messages
Server: ["msg1", "msg2"]
Problem: Wasteful, high latency, server load

HTTP Long Polling:
Client: GET /messages (keep connection open)
Server: (waits up to 30s) → ["msg1"]
Client: GET /messages
Server: (waits up to 30s) → []
Problem: Still wasteful, connection overhead

WebSocket:
Client: WebSocket handshake (once)
Server: Connected
Client ←→ Server: Bidirectional messages
Benefits: Low latency, efficient, real-time
```

**WebSocket Handshake (HTTP Upgrade):**
```
Client Request:
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com

Server Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

Handshake Validation:
Sec-WebSocket-Accept = base64(sha1(
  Sec-WebSocket-Key +
  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
))

Prevents cross-protocol attacks
```

**WebSocket Frame Structure:**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+

FIN: Last fragment of message
RSV1-3: Reserved (must be 0)
Opcode: Frame type
MASK: Payload is masked (required from client)
Payload len: 7, 7+16, or 7+64 bits
Masking key: 4 bytes (if MASK=1)
Payload data: Actual message data
```

**OpCodes:**

| OpCode | Name | Direction | Use Case |
|--------|------|-----------|----------|
| `0x0` | Continuation | Both | Continuation of fragmented message |
| `0x1` | Text | Both | UTF-8 text data |
| `0x2` | Binary | Both | Binary data |
| `0x3-7` | Non-control | Both | Reserved |
| `0x8` | Close | Both | Close connection |
| `0x9` | Ping | Both | Keep-alive/health check |
| `0xA` | Pong | Both | Response to ping |
| `0xB-F` | Control | Both | Reserved |

**Real-World Use Cases:**

| Application | Why WebSocket? | System Design Considerations |
|-------------|----------------|------------------------------|
| **Real-time Chat** | Instant message delivery | Need message queuing, persistence for offline users |
| **Live Notifications** | Push updates without polling | Broadcast to many clients efficiently |
| **Collaborative Editing** | Real-time document sync | Operational transformation (OT) or CRDT for conflict resolution |
| **Online Gaming** | Low-latency state updates | Need interpolation, prediction, reconciliation |
| **Financial Tickers** | Real-time price updates | Need reliable delivery, ordering guarantees |
| **IoT Device Communication** | Device control/telemetry | Need authentication, rate limiting, security |

**WebSocket Scaling Challenges:**
```
Problem: Long-lived connections consume server resources

Scenario:
- 100,000 concurrent WebSocket connections
- Each connection: 1 MB memory
- Total memory: 100 GB
- Single server can't handle

Solutions:
1. Horizontal Scaling
   - Load balance connections across multiple servers
   - Use Redis pub/sub for cross-server messaging
   - Challenge: Connection stickiness not needed (pub/sub)

2. Connection Limits
   - Limit connections per server
   - Implement graceful degradation
   - Queue new connections when full

3. Message Batching
   - Batch multiple messages in single frame
   - Reduce frame overhead
   - Trade-off: Slightly higher latency

4. Compression
   - Enable per-message compression
   - Reduce bandwidth usage
   - Trade-off: CPU overhead
```

### gRPC

**gRPC** (Google Remote Procedure Call) is a high-performance RPC framework using Protocol Buffers and HTTP/2. Open-sourced by Google in 2015.

**Why it matters for System Design:**
- gRPC is ideal for microservice communication
- Protocol Buffers provide efficient serialization
- Built-in streaming enables real-time communication
- Code generation ensures type safety
- Interoperability across languages
- Better performance than REST for internal services

**Key Characteristics:**

| Characteristic | Description | System Design Impact |
|-----------------|-------------|----------------------|
| **Protocol Buffers** | Binary serialization | Faster, smaller payloads than JSON |
| **HTTP/2 transport** | Multiplexing, streaming, header compression | Efficient network usage |
| **Streaming** | Client, server, bidirectional streaming | Real-time communication patterns |
| **Code generation** | Stubs generated from `.proto` files | Type safety, less boilerplate |
| **Load balancing** | Built-in client-side load balancing | Better resource utilization |
| **Deadlines/Timeouts** | Context-aware timeout propagation | Proper timeout handling across services |
| **Cancellation** | Request cancellation propagation | Resource cleanup, prevents wasted work |
| **Interceptors** | Middleware for logging, auth, metrics | Cross-cutting concerns |

**Protocol Buffers vs JSON:**
```
JSON Example:
{
  "user_id": "12345",
  "name": "John Doe",
  "email": "john@example.com"
}
Size: ~80 bytes (text)
Parsing: Slow (text parsing)
Human-readable: Yes

Protocol Buffers Example:
message User {
  string user_id = 1;
  string name = 2;
  string email = 3;
}

Serialized: Binary (e.g., 0x0A 0x05 0x31 0x32 0x33 0x34 0x35)
Size: ~30 bytes (binary)
Parsing: Fast (binary parsing)
Human-readable: No

Performance Comparison:
- Serialization: 5-10x faster than JSON
- Deserialization: 5-10x faster than JSON
- Size: 2-5x smaller than JSON
```

**Service Definition (.proto):**
```protobuf
syntax = "proto3";

package user.v1;

service UserService {
  // Unary: Single request, single response
  rpc GetUser(GetUserRequest) returns (User);
  
  // Server Streaming: Single request, stream of responses
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client Streaming: Stream of requests, single response
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
  
  // Bidirectional Streaming: Both sides stream independently
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string user_id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUsersResponse {
  repeated User users = 1;
}

message ChatMessage {
  string user_id = 1;
  string message = 2;
  int64 timestamp = 3;
}
```

**gRPC Communication Patterns:**

**1. Unary RPC:**
```
Client                              Server
   |                                   |
   | -------- Request --------------> |
   |                                   |
   | <------- Response ------------ |
   |                                   |

Use Case: Simple query, single operation
Example: GetUser, UpdateProfile
```

**2. Server Streaming RPC:**
```
Client                              Server
   |                                   |
   | -------- Request --------------> |
   |                                   |
   | <------- Response 1 ------------ |
   | <------- Response 2 ------------ |
   | <------- Response 3 ------------ |
   | <------- Response 4 ------------ |
   | <------- (stream end) ---------- |
   |                                   |

Use Case: Large result set, real-time updates
Example: ListUsers, StockTicker
```

**3. Client Streaming RPC:**
```
Client                              Server
   |                                   |
   | -------- Request 1 -----------> |
   | -------- Request 2 -----------> |
   | -------- Request 3 -----------> |
   | -------- Request 4 -----------> |
   | -------- (stream end) ---------> |
   |                                   |
   | <------- Response ------------ |
   |                                   |

Use Case: Batch processing, file upload
Example: UploadFile, BatchCreate
```

**4. Bidirectional Streaming RPC:**
```
Client                              Server
   |                                   |
   | -------- Request 1 -----------> |
   | <------- Response 1 ------------ |
   | -------- Request 2 -----------> |
   | <------- Response 2 ------------ |
   | -------- Request 3 -----------> |
   | <------- Response 3 ------------ |
   | -------- (stream end) ---------> |
   | <------- (stream end) ---------- |
   |                                   |

Use Case: Real-time bidirectional communication
Example: Chat, VideoCall, CollaborativeEditing
```

**gRPC vs REST Comparison:**

| Feature | gRPC | REST | System Design Decision |
|---------|------|------|------------------------|
| **Protocol** | HTTP/2 | HTTP/1.1 or HTTP/2 | gRPC leverages HTTP/2 features |
| **Serialization** | Protocol Buffers (binary) | JSON/XML (text) | gRPC: faster, smaller; REST: human-readable |
| **Performance** | High (binary, HTTP/2) | Moderate (text, overhead) | gRPC for internal services; REST for external APIs |
| **Streaming** | Native (4 patterns) | Limited (WebSocket/SSE) | gRPC: built-in streaming; REST: requires additional setup |
| **Browser Support** | Requires gRPC-Web | Native | REST for browser; gRPC for backend |
| **Human Readable** | No (binary) | Yes | REST for debugging; gRPC for production |
| **Code Generation** | Built-in from .proto | Swagger/OpenAPI (optional) | gRPC: type-safe; REST: flexible |
| **Contract-First** | Yes (.proto file) | Optional (OpenAPI) | gRPC: strict contract; REST: flexible |
| **Evolution** | Strict (backward compatibility) | Flexible | gRPC: careful versioning; REST: easier evolution |
| **Tooling** | Strong (protoc, plugins) | Varied | gRPC: comprehensive tooling |

**When to Use gRPC:**
```
Use gRPC when:
- Microservice-to-microservice communication
- Need high performance (low latency, high throughput)
- Need streaming (real-time communication)
- Polyglot environment (multiple languages)
- Strong typing and code generation desired
- Internal APIs (not exposed to public)

Use REST when:
- Public-facing APIs
- Browser clients
- Need human-readable payloads
- Simple CRUD operations
- Caching important (HTTP caching)
- Webhooks needed
```

**Real-World Example - Microservice Architecture with gRPC:**
```
Architecture:
API Gateway (REST) → User Service (gRPC)
                              → Order Service (gRPC)
                              → Payment Service (gRPC)

Benefits:
- Internal services use gRPC (fast, efficient)
- External clients use REST (standard, browser-friendly)
- API Gateway translates REST to gRPC
- Streaming for real-time features

Example Flow:
1. Client: GET /api/orders/123 (REST)
2. API Gateway: GetUser (gRPC) → User Service
3. API Gateway: GetOrder (gRPC) → Order Service
4. Order Service: GetPayment (gRPC) → Payment Service
5. Response aggregated and returned as JSON

.proto File Evolution:
// v1
message User {
  string name = 1;
}

// v2 (backward compatible)
message User {
  string name = 1;
  string email = 2;  // New field (optional)
}

// Old clients work (email ignored)
// New clients get email field
```

### REST API Design

**REST (Representational State Transfer)** is an architectural style for designing networked applications. Defined by Roy Fielding in 2000.

**Why it matters for System Design:**
- REST is the de facto standard for public APIs
- Understanding REST principles helps design better APIs
- Proper REST design improves API usability and maintainability
- RESTful APIs are easier to cache, scale, and integrate
- HTTP status codes and methods have specific semantics

**REST Architectural Principles:**

| Principle | Description | System Design Impact |
|-----------|-------------|----------------------|
| **Stateless** | Each request contains all information needed | Easier to scale, no server-side session state |
| **Client-Server** | Separation of concerns | Independent evolution of client and server |
| **Cacheable** | Responses can be cached | Reduced server load, better performance |
| **Uniform Interface** | Standard methods and formats | Consistent API, easier to learn and use |
| **Layered System** | Client can't tell if connected directly or through intermediary | Enables load balancers, proxies, CDNs |
| **Resource-Based** | Everything is a resource identified by URI | Intuitive API design |

**HTTP Methods Explained:**

| Method | Idempotent | Safe | Request Body | Response Body | Purpose | System Design Use |
|--------|------------|------|--------------|----------------|---------|-------------------|
| **GET** | Yes | Yes | No | Yes | Read resource | Retrieve data, cacheable |
| **POST** | No | No | Yes | Yes | Create resource | Create new resource, trigger action |
| **PUT** | Yes | No | Yes | Yes | Update/replace resource | Full update, idempotent |
| **PATCH** | No | No | Yes | Yes | Partial update | Partial update, non-idempotent |
| **DELETE** | Yes | No | No | Yes | Remove resource | Delete resource |
| **HEAD** | Yes | Yes | No | No | Get headers only | Check existence, cache validation |
| **OPTIONS** | Yes | Yes | No | Yes | Get supported methods | CORS preflight, API discovery |
| **TRACE** | Yes | Yes | No | Yes | Echo request | Debugging (rarely used) |

**Idempotency Explained:**
```
Idempotent: Multiple identical requests have same effect as single request

GET /users/123 (idempotent)
- Request 1: Get user 123
- Request 2: Get user 123 (same result)
- Safe to retry

DELETE /users/123 (idempotent)
- Request 1: Delete user 123
- Request 2: Delete user 123 (already deleted, same result)
- Safe to retry

POST /users (not idempotent)
- Request 1: Create user → User 1 created
- Request 2: Create user → User 2 created
- Not safe to retry

PUT /users/123 (idempotent)
- Request 1: Set user 123 to {name: "John"}
- Request 2: Set user 123 to {name: "John"} (same result)
- Safe to retry

PATCH /users/123 (not idempotent)
- Request 1: Increment counter by 1
- Request 2: Increment counter by 1 (different result)
- Not safe to retry
```

**HTTP Status Codes Explained:**

**2xx Success:**
| Code | Meaning | Use Case |
|------|---------|----------|
| `200 OK` | Request succeeded | Standard success response |
| `201 Created` | Resource created | POST successful, Location header with new resource URL |
| `202 Accepted` | Request accepted for processing | Async processing, background job |
| `204 No Content` | Success, no body | DELETE successful, PUT with no return |
| `206 Partial Content` | Partial response | Range requests, resumable downloads |

**3xx Redirection:**
| Code | Meaning | Use Case |
|------|---------|----------|
| `301 Moved Permanently` | Resource relocated permanently | SEO-friendly, update bookmarks |
| `302 Found` | Resource temporarily moved | Temporary redirect |
| `304 Not Modified` | Use cached version | Conditional GET, cache validation |
| `307 Temporary Redirect` | Temporary redirect (preserve method) | POST redirects |
| `308 Permanent Redirect` | Permanent redirect (preserve method) | POST redirects |

**4xx Client Error:**
| Code | Meaning | Use Case |
|------|---------|----------|
| `400 Bad Request` | Invalid syntax | Malformed JSON, missing required fields |
| `401 Unauthorized` | Authentication required | Missing or invalid credentials |
| `403 Forbidden` | No permission | Authenticated but not authorized |
| `404 Not Found` | Resource doesn't exist | Invalid resource ID |
| `405 Method Not Allowed` | Method not supported | Trying POST on GET-only endpoint |
| `409 Conflict` | State conflict | Duplicate resource, version conflict |
| `422 Unprocessable Entity` | Semantic error | Valid syntax but invalid data |
| `429 Too Many Requests` | Rate limit exceeded | API rate limiting |
| `499 Client Closed Request` | Client disconnected | Client timeout or cancel |

**5xx Server Error:**
| Code | Meaning | Use Case |
|------|---------|----------|
| `500 Internal Server Error` | Generic server error | Unhandled exception |
| `502 Bad Gateway` | Upstream server error | Load balancer can't reach backend |
| `503 Service Unavailable` | Server overloaded/down | Maintenance mode, capacity issues |
| `504 Gateway Timeout` | Upstream timeout | Backend took too long |
| `507 Insufficient Storage` | Server can't store data | Disk full, quota exceeded |

**URL Design Best Practices:**
```
✓ Good Design:
- GET /users                    # List users
- GET /users/123                # Get specific user
- GET /users/123/orders         # Get user's orders
- POST /users                   # Create user
- PUT /users/123                # Update user (full)
- PATCH /users/123              # Update user (partial)
- DELETE /users/123             # Delete user
- GET /users?limit=10&offset=20 # Pagination
- GET /users?role=admin         # Filtering
- GET /users?sort=name&order=asc # Sorting

✗ Bad Design:
- GET /getUser?id=123           # Action in URL
- GET /deleteUser/123           # Action in URL
- POST /users/create            # Redundant verb
- GET /api/v1/getUserById/123  # Too verbose
- GET /users/123/delete         # Action in URL (should be DELETE)
- POST /users/123               # Ambiguous (create or update?)

Versioning Strategies:
- URL versioning: /api/v1/users, /api/v2/users
- Header versioning: Accept: application/vnd.api.v1+json
- Content negotiation: Accept: application/json; version=1

Recommendation: URL versioning is clearest
```

---

## 5. Domain Name System (DNS)

### DNS Resolution Process

**DNS** (Domain Name System) translates human-readable domain names to IP addresses. It's the phonebook of the Internet, distributed across thousands of servers worldwide.

**Why it matters for System Design:**
- DNS is critical for all web applications
- DNS resolution latency affects first-byte time
- DNS caching affects how quickly changes propagate
- DNS-based load balancing is common for global systems
- Understanding DNS helps debug connectivity issues
- DNS security (DNSSEC, DDoS) is increasingly important

**Resolution Steps:**

```
User types: www.example.com in browser

Step 1: Browser Cache Check
└─ Browser checks if recently resolved
└─ If found and not expired → Use cached IP
└─ If not found → Proceed to step 2

Step 2: OS Cache Check
└─ Check operating system DNS cache
└─ Check /etc/hosts file
└─ If found and not expired → Use cached IP
└─ If not found → Proceed to step 3

Step 3: Local DNS Resolver (Recursive Resolver)
└─ Contact configured DNS server (e.g., 8.8.8.8, 1.1.1.1)
└─ This is typically your ISP's DNS or public DNS
└─ If resolver has cached answer → Return IP
└─ If not cached → Proceed to step 4

Step 4: Root Nameserver (.)
└─ 13 root servers worldwide (anycast)
└─ Query: "Where is .com TLD server?"
└─ Response: "I don't know, ask .com TLD server"
└─ Returns .com TLD server addresses

Step 5: TLD Nameserver (.com)
└─ Query: "Where is example.com nameserver?"
└─ Response: "I don't know, ask authoritative server"
└─ Returns example.com nameserver addresses (ns1.example.com, ns2.example.com)

Step 6: Authoritative Nameserver
└─ Query: "What is www.example.com IP?"
└─ Response: "www.example.com is 93.184.216.34"

Step 7: Return to Client
└─ IP address returned and cached at each level
└─ Each cache respects TTL (Time To Live)

Total: Typically 2-4 queries, 50-200ms total
```

**DNS Query Types:**

| Query Type | Description | Use Case |
|------------|-------------|----------|
| **Recursive Query** | DNS resolver does all the work | Client asks resolver, resolver gets answer |
| **Iterative Query** | Each server returns best answer it has | Resolver asks servers one by one |

**Root Servers:**
```
13 Root Servers (anycast for redundancy):
- a.root-servers.net (Verisign)
- b.root-servers.net (USC ISI)
- c.root-servers.net (Cogent)
- d.root-servers.net (University of Maryland)
- e.root-servers.net (NASA Ames Research)
- f.root-servers.net (ISC)
- g.root-servers.net (US DoD)
- h.root-servers.net (US Army Research Lab)
- i.root-servers.net (Netnod)
- j.root-servers.net (Verisign)
- k.root-servers.net (RIPE NCC)
- l.root-servers.net (ICANN)
- m.root-servers.net (WIDE Project)

Anycast: Same IP address routes to nearest server
- Reduces latency
- Improves reliability
- Handles DDoS better
```

### DNS Record Types

| Record | Purpose | Example | System Design Relevance |
|--------|---------|---------|------------------------|
| **A** | IPv4 address | `www IN A 93.184.216.34` | Maps domain to IPv4 address |
| **AAAA** | IPv6 address | `www IN AAAA 2606:2800:220:1::` | Maps domain to IPv6 address |
| **CNAME** | Canonical name (alias) | `blog IN CNAME www.example.com.` | Alias one domain to another |
| **MX** | Mail exchange | `IN MX 10 mail.example.com.` | Email delivery routing |
| **TXT** | Text records (SPF, DKIM) | `IN TXT "v=spf1 include:_spf.google.com ~all"` | Email security, verification |
| **NS** | Nameserver | `IN NS ns1.example.com.` | Delegates DNS for subdomain |
| **SOA** | Start of authority | Zone administrative info | Zone management, transfer |
| **PTR** | Reverse DNS | `34.216.184.93.in-addr.arpa. IN PTR www.example.com.` | IP to domain mapping |
| **SRV** | Service location | `_sip._tcp IN SRV 10 5 5060 sipserver.example.com.` | Service discovery |
| **CAA** | Certificate Authority Authorization | `IN CAA 0 issue "letsencrypt.org"` | Restricts which CAs can issue certificates |
| **SPF** | Sender Policy Framework (TXT) | `v=spf1 include:_spf.google.com ~all` | Email anti-spam |
| **DKIM** | DomainKeys Identified Mail (TXT) | `v=DKIM1; k=rsa; p=...` | Email authentication |
| **DMARC** | Domain-based Message Auth (TXT) | `v=DMARC1; p=quarantine; rua=...` | Email policy enforcement |

**Common DNS Record Configurations:**
```
Example Domain: example.com

@ IN SOA ns1.example.com. admin.example.com. (
    2024010101 ; Serial (YYYYMMDDNN)
    3600       ; Refresh (1 hour)
    600        ; Retry (10 minutes)
    604800     ; Expire (7 days)
    86400      ; Minimum TTL (1 day)
)

@ IN NS ns1.example.com.
@ IN NS ns2.example.com.

@ IN A 192.0.2.1
@ IN AAAA 2001:db8::1

www IN A 192.0.2.1
www IN AAAA 2001:db8::1

api IN CNAME www.example.com.

@ IN MX 10 mail.example.com.

mail IN A 192.0.2.10

@ IN TXT "v=spf1 include:_spf.google.com ~all"

_dmarc.example.com. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"

_dkim._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
```

### DNS Caching

**TTL (Time To Live)** controls how long DNS responses are cached.

**Why TTL Matters:**
- Long TTL: Less DNS traffic, faster resolution, harder to change quickly
- Short TTL: More DNS queries, slower resolution, easier to change
- TTL affects how quickly DNS changes propagate

**Caching Locations:**

| Location | TTL Control | Typical TTL | System Design Impact |
|-----------|-------------|-------------|------------------------|
| **Browser Cache** | Respects DNS TTL | Browser-specific | First visit slower, subsequent visits faster |
| **OS Cache** | Respects DNS TTL | OS-specific | System-wide caching |
| **Local DNS Resolver** | Respects DNS TTL (usually) | Configurable | ISP or public DNS caching |
| **Intermediate Servers** | May ignore TTL | Variable | Can cause stale DNS issues |

**Viewing DNS Cache:**
```
Browser (Chrome):
chrome://net-internals/#dns

Windows:
ipconfig /displaydns

Linux (systemd-resolved):systemd-resolve --statistics

Linux (dnsmasq):cat /var/cache/dnsmasq/dnsmasq.cache

macOS:
scutil --dns
sudo killall -HUP mDNSResponder (clear cache)

Linux (nscd):nscd -i hosts
```

**TTL Considerations:**
```
Scenario: Moving website to new IP

Problem:
- Current TTL: 86400 (24 hours)
- Change DNS to new IP
- Some users still resolve to old IP for 24 hours
- Downtime for some users

Solution: TTL Rollback
1. Day 1: Lower TTL to 300 (5 minutes)
2. Day 2: Wait for old TTL to expire
3. Day 3: Change DNS to new IP
4. Day 4: Restore TTL to 86400

Trade-offs:
- Short TTL: More DNS queries, higher cost
- Long TTL: Harder to make changes
- Balance based on change frequency

Recommended TTLs:
- Static resources (CDN): 1 hour to 1 day
- Application servers: 5-30 minutes
- Frequently changing: 1-5 minutes
```

### Load Balancing with DNS

**DNS-Based Load Balancing** uses DNS to distribute traffic across multiple servers.

**Why it matters for System Design:**
- Simple way to distribute traffic globally
- No additional infrastructure needed
- Works with any protocol (not just HTTP)
- Limited by DNS caching
- No real-time health awareness

**DNS-Based Load Balancing Methods:**

**1. Multiple A/AAAA Records (Round Robin):**
```
$ORIGIN example.com.
www IN A 192.0.2.1
www IN A 192.0.2.2
www IN A 192.0.2.3
www IN A 192.0.2.4

Behavior:
- DNS server returns all IPs in random order
- Client typically uses first IP
- Round-robin at DNS level
- Not true load balancing (client choice)

Limitations:
- Clients may cache and always use first IP
- No health awareness
- Uneven distribution possible
```

**2. Geographic DNS (GeoDNS):**
```
us-east.example.com → 192.0.2.1 (US East datacenter)
us-west.example.com → 192.0.2.2 (US West datacenter)
eu-west.example.com → 198.51.100.1 (Europe datacenter)
ap-south.example.com → 203.0.113.1 (Asia Pacific datacenter)

Behavior:
- DNS server determines client location
- Returns IP of nearest datacenter
- Reduces latency
- Improves user experience

Implementation:
- Use GeoDNS provider (e.g., AWS Route 53, Cloudflare)
- Configure geographic regions
- Set up health checks per region

Benefits:
- Reduced latency
- Better user experience
- Reduced bandwidth costs
```

**3. Weighted Records:**
```
www IN A 192.0.2.1 (weight 10)
www IN A 192.0.2.2 (weight 5)
www IN A 192.0.2.3 (weight 1)

Behavior:
- Server 1 gets 62.5% of traffic
- Server 2 gets 31.25% of traffic
- Server 3 gets 6.25% of traffic

Use Cases:
- Canary deployments (small traffic to new server)
- Gradual migration
- Different server capacities
```

**4. Latency-Based Routing:**
```
Behavior:
- DNS server measures latency to each server
- Returns IP of fastest server
- Dynamically adjusts based on network conditions

Benefits:
- Always routes to fastest server
- Adapts to network issues
- Better user experience

Limitations:
- Requires active probing
- Increased DNS server load
```

**5. Health Check-Based Routing:**
```
Behavior:
- DNS server monitors backend health
- Removes unhealthy IPs from responses
- Automatically adds healthy IPs back

Health Checks:
- TCP port check (port 80/443)
- HTTP check (GET /health)
- Custom check (application-specific)

Configuration:
- Check interval: 30 seconds
- Unhealthy threshold: 3 consecutive failures
- Healthy threshold: 2 consecutive successes

Benefits:
- Automatic failover
- No manual intervention
- Improved reliability
```

**DNS Load Balancing Limitations:**

| Limitation | Description | Mitigation |
|------------|-------------|------------|
| **Caching** | DNS caching can stick clients to unhealthy servers | Low TTL, health checks, client-side load balancing |
| **No Real-Time Load Awareness** | DNS doesn't know current server load | Combine with application-level load balancing |
| **Client Behavior Unpredictable** | Clients may not follow DNS ordering | Use consistent hashing at application level |
| **Session Persistence Difficult** | DNS doesn't track sessions | Use cookies or application-level stickiness |
| **Slow Failover** | TTL must expire before changes take effect | Low TTL, but increases DNS load |

---

## 6. Load Balancing

### Layer 4 vs Layer 7 Load Balancing

**Why it matters for System Design:**
- Load balancers are critical for scalability and availability
- Choosing between L4 and L7 affects performance and capabilities
- Understanding load balancing helps design resilient architectures
- Load balancers are single points of failure (need HA)
- Health checks ensure traffic only goes to healthy servers

**Layer 4 (Transport Layer) Load Balancing:**

L4 load balancers operate at the transport layer (TCP/UDP). They make routing decisions based on IP addresses and ports.

| Characteristic | Description | System Design Impact |
|---------------|-------------|----------------------|
| **Decision Basis** | IP address and TCP/UDP port | Simple routing, no content inspection |
| **Protocol Agnostic** | Works for any TCP/UDP protocol | Supports HTTP, TCP, UDP, gRPC, WebSocket |
| **Performance** | Higher (less processing) | Lower latency, higher throughput |
| **Visibility** | No visibility into HTTP headers/content | Can't route based on URL or headers |
| **SSL Termination** | Pass-through only (can't terminate TLS) | Backend servers must handle TLS |
| **Session Persistence** | IP-based stickiness | Limited session management |
| **Features** | Basic load balancing | Simple, reliable |

**Use Cases:**
- Database load balancing (MySQL, PostgreSQL)
- Non-HTTP services (gRPC, custom protocols)
- High-performance requirements (minimal latency)
- Pass-through SSL (backend handles TLS)
- Examples: AWS NLB, HAProxy (in TCP mode)

```
Client (192.168.1.100:54321) ──┐
                               ├──> L4 LB (10.0.0.1:80)
Client (192.168.1.101:54322) ──┘       ├──> Server A (10.0.0.10:80)
                                        ├──> Server B (10.0.0.11:80)
                                        └──> Server C (10.0.0.12:80)
Connection: L4 LB maintains NAT table mapping client:port to server:port
```

**Layer 7 (Application Layer) Load Balancing:**

L7 load balancers operate at the application layer (HTTP/HTTPS). They make routing decisions based on HTTP content.

| Characteristic | Description | System Design Impact |
|---------------|-------------|----------------------|
| **Decision Basis** | HTTP headers, URL, cookies, content | Advanced routing capabilities |
| **Protocol Specific** | HTTP/HTTPS only | Not suitable for non-HTTP protocols |
| **Performance** | Lower (more processing) | Higher latency, lower throughput |
| **Visibility** | Full visibility into HTTP requests/content | Can inspect and modify requests |
| **SSL Termination** | Can terminate TLS at load balancer | Reduces load on backend servers |
| **Session Persistence** | Cookie-based, header-based | Flexible session management |
| **Features** | Advanced (routing, rewriting, caching) | Powerful traffic management |

**Use Cases:**
- Web application load balancing
- Content-based routing (different paths to different services)
- API gateway functionality
- SSL termination at edge
- A/B testing, canary deployments
- Examples: AWS ALB, Nginx, HAProxy (in HTTP mode)

```
Client ──> L7 LB ──> Backend Servers
         (Inspects: Host header, URL path, Cookies, Query params)
         
/api/*        → API servers
/static/*     → Static file servers
/websocket    → WebSocket servers
Mobile users  → Mobile-optimized servers
```

**Comparison:**
| Feature | L4 LB | L7 LB | System Design Decision |
|---------|-------|-------|------------------------|
| **Layer** | Transport (TCP/UDP) | Application (HTTP) | L4 for non-HTTP, L7 for HTTP |
| **Decision Basis** | IP, port | URL, headers, cookies | L7 for content-based routing |
| **Protocol Agnostic** | Yes | No (HTTP only) | L4 for multi-protocol support |
| **Performance** | Higher (1-10ms overhead) | Lower (10-50ms overhead) | L4 for performance-critical |
| **SSL Termination** | Pass-through only | Can terminate TLS | L7 to offload TLS from backends |
| **Session Persistence** | IP-based | Cookie/header-based | L7 for flexible session management |
| **Features** | Basic (load balancing) | Advanced (routing, caching, rewriting) | L7 for advanced features |
| **Cost** | Lower | Higher | L4 is cheaper per request |

### Load Balancing Algorithms

**Why it matters for System Design:**
- Algorithm choice affects resource utilization
- Different algorithms suit different use cases
- Understanding algorithms helps troubleshoot load distribution issues
- Some algorithms provide session persistence
- Weight-based algorithms handle heterogeneous server capacities

| Algorithm | Description | Best For | System Design Impact |
|-----------|-------------|----------|------------------------|
| **Round Robin** | Sequential distribution | Equal capacity servers | Simple, even distribution |
| **Weighted Round Robin** | Round robin with capacity weighting | Heterogeneous servers | Accounts for capacity differences |
| **Least Connections** | To server with fewest active connections | Long-lived connections | Accounts for current load |
| **Weighted Least Connections** | Least connections with weights | Varied capacity + long connections | Capacity + load awareness |
| **Least Response Time** | Combines connections + latency | Latency-sensitive apps | Performance-based routing |
| **IP Hash** | Hash of client IP determines server | Session persistence needed | Sticky sessions |
| **Random** | Random selection | Distributed evenly at scale | Simple, no state needed |
| **Geographic** | Closest geographic server | Global applications | Reduced latency |

**Detailed Algorithm Explanations:**

**1. Round Robin:**
```
Description: Requests distributed evenly across servers

Servers: [A, B, C]

Request 1 → A
Request 2 → B
Request 3 → C
Request 4 → A
Request 5 → B
Request 6 → C
...

Pros:
- Simple to implement
- Even distribution (assuming equal server capacity)
- No server state needed

Cons:
- Doesn't account for server capacity differences
- Doesn't account for current load
- Doesn't account for request duration

Use Case:
- Homogeneous servers with equal capacity
- Similar request durations
- Simple scenarios
```

**2. Least Connections:**
```
Description: New requests sent to server with fewest active connections

Servers: [A: 10 connections, B: 5 connections, C: 8 connections]

New request → B (fewest connections)

Pros:
- Accounts for current load
- Better for varying request durations
- More balanced than round robin

Cons:
- Requires tracking connection state
- May not account for request processing time
- Can oscillate under burst traffic

Use Case:
- Varying request durations
- Long-running connections
- Better load distribution than round robin
```

**3. IP Hash (Consistent Hashing):**
```
Description: Consistent hashing based on client IP

Formula: hash(client_ip) % num_servers

Example:
Client IP: 192.168.1.10
Hash: 12345678
Servers: 3
Server = 12345678 % 3 = 2 (Server C)

Pros:
- Session persistence (same client to same server)
- Predictable routing
- Good for stateful applications

Cons:
- Uneven distribution (some IPs may hash to same server)
- Adding/removing servers affects many clients
- Doesn't account for server capacity

Use Case:
- Stateful applications requiring session persistence
- Sticky sessions
- Cache affinity
```

**Consistent Hashing:**
```
Hash(client IP) % number_of_servers = server_index

Server pool changes:
- Traditional hash: Most clients remap (bad)
- Consistent hash: Minimal remapping (good)

Virtual nodes improve distribution:
- Each physical server = multiple virtual nodes on ring
- Better load distribution
- Smoother additions/removals
```

### Health Checks

**Why it matters for System Design:**
- Health checks prevent traffic to unhealthy servers
- Automatic failover improves availability
- Graceful degradation when servers fail
- Health check configuration affects failover speed
- Too aggressive = flapping, too passive = slow failover

**Health Check Types:**

| Type | Description | Use Case | System Design Impact |
|------|-------------|----------|------------------------|
| **Active** | Load balancer periodically probes servers | Explicit health verification | Fast detection, adds load |
| **Passive** | Monitor actual request responses | Infer health from traffic | No extra load, slower detection |

**1. Active Health Checks:**
```
Load balancer periodically probes servers:

- TCP connect check (port 80/443)
- HTTP GET/HEAD request to /health endpoint
- HTTPS request with TLS verification
- Custom protocol checks (application-specific)

Pros:
- Explicit health verification
- Fast detection
- Can check specific components

Cons:
- Adds load to servers
- May not reflect actual application state
```

**2. Passive Health Checks:**
```
Monitor actual request responses:

- Mark unhealthy on 5xx errors
- Mark unhealthy on connection failures
- Mark unhealthy on timeouts
- Outlier detection (slow responses)

Pros:
- No extra load
- Reflects actual application state
- Simpler configuration

Cons:
- Slower detection
- Requires actual traffic
- May not detect issues until traffic flows
```

**Health Check Configuration:**
```
Key Parameters:

1. Interval (Check Frequency)
   - How often to perform health check
   - Typical: 5-30 seconds
   - Trade-off: Frequent = faster detection, more load

2. Timeout
   - How long to wait for response
   - Typical: 2-10 seconds
   - Trade-off: Shorter = faster failover, more false positives

3. Unhealthy Threshold
   - Failures before marking server unhealthy
   - Typical: 2-5 consecutive failures
   - Trade-off: Lower = faster failover, more flapping

4. Healthy Threshold
   - Successes before marking server healthy
   - Typical: 2-5 consecutive successes
   - Trade-off: Lower = faster recovery, more flapping

Example Configuration:
- Interval: 10 seconds
- Timeout: 5 seconds
- Unhealthy threshold: 3 failures
- Healthy threshold: 2 successes

Timeline:
T=0s:   Health check (pass)
T=10s:  Health check (pass)
T=20s:  Health check (fail)
T=30s:  Health check (fail)
T=40s:  Health check (fail) → Mark unhealthy (3 failures)
T=50s:  Health check (pass)
T=60s:  Health check (pass) → Mark healthy (2 successes)
```

**Health Check Best Practices:**
```
1. Use dedicated health endpoint
   - /health (lightweight)
   - /health/ready (ready to receive traffic)
   - /health/live (process is alive)

2. Keep health checks lightweight
   - Check critical dependencies only
   - Avoid expensive operations
   - Return quickly (< 100ms)

3. Return appropriate HTTP status codes
   - 200 OK: Healthy
   - 503 Service Unavailable: Unhealthy

4. Include health check details
   {
     "status": "healthy",
     "checks": {
       "database": "ok",
       "cache": "ok",
       "external_api": "degraded"
     }
   }

5. Implement circuit breakers
   - Stop checking unhealthy servers temporarily
   - Prevents cascading failures

6. Consider passive health checks
   - Monitor actual traffic success/failure
   - Complement active health checks
```

**Health Check Endpoints:**
```
GET /health
Response: 200 OK
{
  "status": "healthy",
  "checks": {
    "database": "connected",
    "cache": "connected",
    "disk": "ok"
  }
}

GET /health/ready
Response: 200 OK (ready to receive traffic)
or 503 Service Unavailable (not ready)

GET /health/live
Response: 200 OK (process is alive)
or 503 Service Unavailable (process is dead)
```

### Session Persistence (Sticky Sessions)

**Methods:**

1. **Source IP Affinity:**
   - Same client IP → Same server
   - Simple, but problematic with NAT

2. **Cookie-Based:**
   - Load balancer inserts cookie (e.g., `SERVERID=A123`)
   - Subsequent requests include cookie
   - Works through NAT

3. **Application Session:**
   - Session stored in shared cache (Redis)
   - Any server can handle any request
   - Best for scalability

**Recommendation:**
- Prefer stateless design with shared session store
- Use sticky sessions only when necessary
- Session replication scales poorly

---

## 7. Caching and Content Delivery

### CDN Architecture

**CDN (Content Delivery Network)** distributes content geographically for lower latency.

**CDN Architecture:**
```
User in Tokyo ──┐
                ├──> Internet
User in London ─┘        │
                         ▼
               ┌─────────────────┐
               │   Origin Server │
               │  (Central Data) │
               └────────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Edge -  │    │ Edge -  │    │ Edge -  │
   │ Tokyo   │    │ London  │    │  NYC    │
   └────┬────┘    └────┬────┘    └────┬────┘
        │               │               │
   Tokyo User      London User      NYC User
```

**CDN Tiers:**
1. **Edge Locations:** Closest to users; cache hot content
2. **Regional Edges:** Mid-tier cache; larger capacity
3. **Origin Shield:** Single CDN POP that talks to origin

**CDN Benefits:**
- Reduced latency (closer to users)
- Reduced origin load
- DDoS protection
- Global availability
- Bandwidth savings

### Caching Strategies

**Cache Types:**

1. **Browser Cache:**
   ```http
   Cache-Control: max-age=3600
   ETag: "abc123"
   Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
   ```

2. **CDN Cache:**
   - Pull CDN: Fetches on first request
   - Push CDN: Origin pushes to edges

3. **Application Cache:**
   - In-memory (Guava, Caffeine)
   - Distributed (Redis, Memcached)

4. **Database Cache:**
   - Query cache (MySQL - deprecated in 8.0)
   - Buffer pool (InnoDB)

**Cache Headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| `Cache-Control` | Modern caching directives | `max-age=3600, public` |
| `Expires` | Absolute expiration date | `Mon, 01 Jan 2025 00:00:00 GMT` |
| `ETag` | Resource version identifier | `"33a64df5"` |
| `Last-Modified` | Modification timestamp | `Mon, 01 Jan 2024 00:00:00 GMT` |
| `Vary` | Cache key variations | `Accept-Encoding` |
| `Age` | Time in cache | `3600` |

**Cache-Control Directives:**
```
Public/Private:
- public: Can be cached by anyone
- private: Only browser cache

Expiration:
- max-age=3600: Cache for 3600 seconds
- s-maxage=3600: CDN cache duration (overrides max-age for shared caches)
- no-cache: Must revalidate before using
- no-store: Never cache
- must-revalidate: Strict expiration enforcement

Stale Content:
- stale-while-revalidate=60: Serve stale while fetching fresh
- stale-if-error=86400: Serve stale on origin error
```

### Cache Invalidation

**Cache Invalidation Methods:**

1. **Time-Based (TTL):**
   - Automatic expiration after set duration
   - Simple, but content may be stale

2. **Active Invalidation:**
   - CDN/API call to purge specific URLs
   - Immediate, but requires integration
   ```bash
   # CloudFront invalidation
   aws cloudfront create-invalidation \
     --distribution-id E1234567890ABC \
     --paths "/images/*" "/index.html"
   ```

3. **Versioned URLs:**
   - Include version/hash in filename
   - New version = new URL
   - Old versions still valid (immutable)
   ```
   /css/styles.v123.css
   /js/app.a3f4b2c.js
   ```

4. **Cache Busting:**
   - Query parameters with version
   ```
   /styles.css?v=123
   ```

**Cache Invalidation Patterns:**

**Write-Through:**
```
Update DB ──> Update Cache ──> Return
Consistent but slower writes
```

**Write-Around:**
```
Update DB ──> Invalidate Cache ──> Return
Read will repopulate cache
```

**Write-Back (Write-Behind):**
```
Update Cache ──> Return ──> Async write to DB
Fast but risk of data loss
```

---

## 8. Network Security

### TLS/SSL Security Best Practices

**Why it matters for System Design:**
- TLS is mandatory for modern web applications
- Proper TLS configuration prevents common attacks
- Certificate management is operational overhead
- Understanding TLS helps debug connection issues
- Security best practices protect user data

See [HTTPS and TLS/SSL](#https-and-tlsssl) section for detailed handshake.

**Additional Security Considerations:**

**Certificate Pinning:**
```
Description: Hardcode expected certificate/public key in application

Purpose:
- Prevents MITM attacks with rogue CAs
- Ensures app only talks to specific server
- Used in mobile apps and high-security applications

Implementation:
- Public key pinning: Pin server's public key
- Certificate pinning: Pin entire certificate
- HPKP (HTTP Public Key Pinning): Deprecated due to complexity

Example (Mobile App):
// Android
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build();

// iOS (Swift)
let pins: [String] = [
    "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
]

Trade-offs:
- Strong security
- Difficult to rotate certificates
- Can break app if certificate changes
- Requires careful planning
```

**Perfect Forward Secrecy (PFS):**
```
Description: Session keys derived from ephemeral DH exchange

Why it matters:
- Compromised long-term key can't decrypt past sessions
- Each session uses unique ephemeral keys
- Critical for high-security applications

Requirements:
- ECDHE or DHE key exchange (not RSA key exchange)
- TLS 1.2 or TLS 1.3 (TLS 1.3 requires PFS)

Without PFS:
- Server private key compromised
- Attacker can decrypt all past sessions
- Historical data exposed

With PFS:
- Server private key compromised
- Attacker can only decrypt current/future sessions
- Past sessions remain secure

System Design Impact:
- Must use ECDHE cipher suites
- Slightly higher CPU overhead (ephemeral key generation)
- Required for compliance in many industries
```

**HSTS (HTTP Strict Transport Security):**
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

```
Purpose:
- Forces HTTPS for specified duration
- Prevents SSL stripping attacks
- Protects users from downgrade attacks

Directives:
- max-age: How long to remember (in seconds)
- includeSubDomains: Apply to all subdomains
- preload: Include in browser preload list

Example:
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Behavior:
- Browser remembers for 1 year
- All subdomains must use HTTPS
- Browser blocks HTTP requests automatically

Preload List:
- Submit domain to hstspreload.org
- Browsers preload HSTS configuration
- First visit is already protected

System Design Impact:
- Once enabled, HTTPS is mandatory
- Difficult to disable (max-age must expire)
- Test thoroughly before enabling
- Consider includeSubDomains carefully
```

### Certificate Authority (CA)

**Why it matters for System Design:**
- Certificates establish trust between clients and servers
- Certificate management is operational overhead
- Certificate expiration causes outages
- Understanding certificate chain helps debug TLS issues
- Different certificate types suit different use cases

**Certificate Chain:**
```
Root CA (Self-signed, in trust store)
    └── Intermediate CA
            └── Intermediate CA
                    └── Leaf Certificate (example.com)

Why chain matters:
- Trust is delegated from root to intermediate to leaf
- Browsers trust root CAs
- Intermediate CAs sign leaf certificates
- Compromised intermediate can be revoked without affecting root
```

**Certificate Validation Steps:**
```
1. Verify certificate chain up to trusted root
   - Each certificate must be signed by its issuer
   - Chain must end in trusted root CA

2. Check certificate not expired
   - Valid from: start date
   - Valid to: expiration date
   - Current time must be within range

3. Verify domain name matches
   - Common Name (CN) or Subject Alternative Name (SAN)
   - Must match requested hostname
   - Wildcard: *.example.com matches www.example.com

4. Check certificate not revoked (CRL/OCSP)
   - CRL: Certificate Revocation List (downloaded list)
   - OCSP: Online Certificate Status Protocol (real-time check)

5. Verify signature
   - Cryptographic signature verification
   - Ensures certificate hasn't been tampered with

6. Check certificate purpose
   - Server authentication, client authentication, code signing
   - Key usage and extended key usage
```

**Types of Certificates:**

| Type | Validation | Use Case | Example | System Design Impact |
|------|------------|----------|---------|------------------------|
| **DV** | Domain ownership only | Standard websites | Let's Encrypt | Quick issuance, low cost |
| **OV** | Organization identity | Business sites | DigiCert | Higher trust, more verification |
| **EV** | Extended validation | Banking, high-trust | Bank websites | Highest trust, expensive |
| **Wildcard** | *.example.com | Subdomains | *.example.com | One cert for all subdomains |
| **SAN** | Multiple domains | Multi-domain apps | example.com, example.net | One cert for multiple domains |
| **Multi-SAN** | Multiple domains (100+) | Large organizations | Many subdomains | Consolidated certificate management |

**Certificate Revocation:**
```
CRL (Certificate Revocation List):
- Downloaded list of revoked certificates
- Cached by client
- Can be large (millions of revoked certs)
- Slower to check

OCSP (Online Certificate Status Protocol):
- Real-time status check
- Queries CA directly
- Faster than CRL
- Privacy concerns (CA knows what sites you visit)

OCSP Stapling:
- Server attaches OCSP response to TLS handshake
- Reduces client overhead
- Improves privacy
- Requires server configuration

System Design Impact:
- OCSP stapling recommended for performance
- CRL fallback for clients without OCSP
- Certificate revocation affects availability
```

### Firewalls

**Why it matters for System Design:**
- Firewalls are critical for network security
- Defense in depth strategy requires multiple firewall layers
- Firewall rules affect connectivity and must be carefully designed
- Understanding firewall types helps design secure architectures
- Misconfigured firewalls can cause outages

**Firewall Types:**

| Type | Layer | Function | Examples | System Design Use |
|------|-------|----------|----------|-------------------|
| **Network Firewall** | L3/L4 | Packet filtering based on IP/port | iptables, AWS Security Groups | Perimeter security, network segmentation |
| **Web Application Firewall (WAF)** | L7 | Inspects HTTP traffic, protects against OWASP Top 10 | AWS WAF, Cloudflare WAF, ModSecurity | Application-layer security |
| **Host Firewall** | All | Runs on individual servers | Windows Firewall, ufw, firewalld | Server-level security, defense in depth |

**1. Network Firewall (L3/L4):**
```
Function:
- Packet filtering based on IP address and port
- Stateful inspection (tracks connection state)
- Can perform NAT, port forwarding

Features:
- Allow/deny rules based on IP, port, protocol
- State tracking (allow related traffic)
- Rate limiting
- Logging

Examples:
- iptables (Linux)
- pfSense (open source firewall appliance)
- AWS Security Groups (stateful, cloud-native)
- Azure Network Security Groups

System Design Impact:
- Perimeter security
- Network segmentation (VPC subnets)
- Defense in depth
- Can be bottleneck if not sized properly
```

**2. Web Application Firewall (WAF):**
```
Function:
- Inspects HTTP traffic
- Protects against OWASP Top 10 vulnerabilities
- Can block malicious requests

Protection against:
- SQL injection
- Cross-site scripting (XSS)
- Command injection
- Path traversal
- File inclusion attacks
- DDoS attacks (application layer)

Examples:
- AWS WAF (cloud-native)
- Cloudflare WAF (CDN-integrated)
- ModSecurity (open source)
- F5 BIG-IP ASM (hardware appliance)

System Design Impact:
- Application-layer security
- Can add latency (inspect each request)
- Requires tuning to avoid false positives
- Critical for public-facing web applications
```

**3. Host Firewall:**
```
Function:
- Runs on individual servers
- Controls inbound/outbound traffic per host
- Defense in depth

Examples:
- Windows Firewall (Windows)
- ufw (Ubuntu)
- firewalld (CentOS/RHEL)
- iptables (Linux)

System Design Impact:
- Server-level security
- Protects compromised services from spreading
- Last line of defense
- Must be configured consistently across fleet
```

**Firewall Rules (Example):**
```
Inbound Rules:
- Allow TCP 22 from 10.0.0.0/8 (SSH admin access)
- Allow TCP 80 from 0.0.0.0/0 (HTTP public)
- Allow TCP 443 from 0.0.0.0/0 (HTTPS public)
- Allow TCP 3306 from 10.0.0.0/8 (MySQL from app servers)
- Allow TCP 6379 from 10.0.0.0/8 (Redis from app servers)
- Deny all other inbound

Outbound Rules:
- Allow TCP 80 to 0.0.0.0/0 (HTTP for updates)
- Allow TCP 443 to 0.0.0.0/0 (HTTPS for updates)
- Allow TCP 53 to 8.8.8.8/32 (DNS)
- Allow TCP 3306 to 10.0.0.10/32 (Database)
- Deny all other outbound (if security required)

Best Practices:
- Default deny (whitelist approach)
- Least privilege (only allow what's needed)
- Document all rules
- Regularly audit rules
- Use security groups in cloud (easier management)
```

### DDoS Protection

**Why it matters for System Design:**
- DDoS attacks can take down any service
- DDoS protection is critical for availability
- Understanding DDoS types helps design resilient architectures
- DDoS protection adds cost and complexity
- Layered defense is required for effective protection

**DDoS Types:**

| Type | Description | Attack Vector | Mitigation Strategy |
|------|-------------|---------------|---------------------|
| **Volumetric** | Saturate bandwidth | UDP floods, ICMP floods | CDN, scrubbing centers, traffic scrubbing |
| **Protocol** | Exhaust server resources | SYN floods, Ping of Death | Rate limiting, connection tracking |
| **Application** | Exhaust application resources | HTTP floods, slowloris | WAF, rate limiting, CAPTCHA |

**1. Volumetric Attacks:**
```
Description:
- Overwhelm network bandwidth
- UDP floods, ICMP floods, amplification attacks
- Goal: Saturate link to target

Amplification Attacks:
- DNS amplification: Small query → Large response
- NTP amplification: Small query → Large response
- SSDP amplification: Small query → Large response

Example:
Attacker sends 1GB/s to DNS reflector
DNS reflector sends 100GB/s to target
Target's 10Gbps link is saturated

Mitigation:
- CDN to absorb traffic (Cloudflare, AWS CloudFront, Akamai)
- Scrubbing centers (route traffic through scrubbing)
- Blackhole routing (drop traffic at ISP level)
- Anycast (distribute attack across multiple locations)

System Design Impact:
- Need CDN or DDoS protection service
- Additional cost
- May affect legitimate traffic during attacks
```

**2. Protocol Attacks:**
```
Description:
- Exhaust server resources (connections, memory)
- SYN floods, Ping of Death, fragmented packet attacks
- Goal: Exhaust server resources, not bandwidth

SYN Flood:
- Attacker sends many SYN packets
- Server allocates resources for each connection
- Server never completes handshake
- Server runs out of connection slots

Mitigation:
- SYN cookies (don't allocate resources until handshake completes)
- Rate limiting on SYN packets
- Connection tracking with timeouts
- Increase backlog queue

System Design Impact:
- Need SYN cookies enabled
- Proper connection timeout configuration
- May need connection tracking at load balancer
```

**3. Application Layer Attacks:**
```
Description:
- Exhaust application resources (CPU, database, memory)
- HTTP floods, Slowloris, request floods
- Goal: Crash application or make it unresponsive

HTTP Flood:
- Attacker sends many HTTP requests
- Each request requires processing
- Application becomes overloaded

Slowloris:
- Attacker opens many connections
- Sends requests very slowly
- Keeps connections open
- Server runs out of connection slots

Mitigation:
- WAF to filter malicious requests
- Rate limiting per IP, per endpoint
- CAPTCHA for suspicious traffic
- Request size limits
- Timeout for slow connections
- Auto-scaling to handle load

System Design Impact:
- Need WAF for application-layer protection
- Rate limiting at multiple layers
- Auto-scaling to handle legitimate traffic
- May block legitimate users during attacks
```

**DDoS Mitigation Strategies:**

**1. CDN/Edge Protection:**
```
Description: Absorb and distribute attack traffic at edge

Providers:
- Cloudflare (free and paid tiers)
- AWS Shield (integrated with AWS)
- Akamai (enterprise)
- Fastly (enterprise)

Benefits:
- Massive bandwidth capacity (Tbps+)
- Global distribution
- Anycast routing
- Integrated WAF and rate limiting

System Design Impact:
- Route all traffic through CDN
- Additional cost (but cheaper than downtime)
- May add latency (but can improve with caching)
- Need to configure DNS to point to CDN
```

**2. Rate Limiting:**
```
Description: Limit request rate per IP, per endpoint

Implementation (nginx example):
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
}
```

Strategies:
- Per IP rate limiting
- Per endpoint rate limiting
- Global rate limiting
- Token bucket algorithm

System Design Impact:
- Need rate limiting at multiple layers
- May block legitimate users (false positives)
- Requires tuning for each endpoint
- Use Redis for distributed rate limiting
```

**3. Auto-scaling:**
```
Description: Automatically scale resources to handle load

Strategies:
- Horizontal scaling (add more servers)
- Vertical scaling (increase server capacity)
- Database read replicas
- Queue-based architecture

Benefits:
- Handle legitimate traffic spikes
- Absorb some DDoS attacks
- Pay only for resources used

System Design Impact:
- Need auto-scaling configuration
- Scaling takes time (not instant)
- May increase cost during attacks
- Requires monitoring and alerts
```

**4. Challenge Pages (CAPTCHA):**
```
Description: Challenge suspicious traffic with CAPTCHA

When to use:
- High request rate from single IP
- Suspicious user agent
- Bot-like behavior

Types:
- reCAPTCHA (Google)
- hCaptcha
- Custom challenges

System Design Impact:
- Can frustrate legitimate users
- Adds friction
- Need fallback for accessibility
```

---

5. **Blackholing:**
```
Description: Route attack traffic to null (last resort)

When to use:
- Attack is overwhelming all defenses
- Protecting other customers (shared infrastructure)
- Mitigating collateral damage

Trade-offs:
- Service becomes unavailable for everyone
- Legitimate traffic also blocked
- Last resort only

System Design Impact:
- Emergency measure only
- Coordinate with ISP/cloud provider
- Have incident response plan ready
```

### VPN and Tunneling

**Why it matters for System Design:**
- VPNs enable secure remote access to internal resources
- Tunneling protocols affect performance and security
- Understanding VPN types helps design secure remote access
- VPN configuration affects network architecture
- VPNs are critical for distributed teams and remote work

**VPN Types:**

| Type | Layer | Use Case | Pros | Cons | System Design Impact |
|------|-------|----------|------|------|------------------------|
| **IPsec VPN** | L3 | Site-to-site connectivity | Strong security, network layer | Complex configuration | Corporate network interconnection |
| **SSL/TLS VPN** | L7 | Remote access | Easy to use, clientless option | Application layer only | Remote employee access |
| **WireGuard** | L3 | Modern VPN | Fast, simple, secure | Newer protocol | High-performance VPN |
| **OpenVPN** | L3/L7 | Flexible VPN | Highly configurable | Complex setup | Custom VPN solutions |

**1. IPsec VPN:**
```
Description: Site-to-site connectivity with network layer encryption

Protocols:
- IKE (Internet Key Exchange): Key management
- AH (Authentication Header): Authentication only
- ESP (Encapsulating Security Payload): Encryption + authentication

Modes:
- Transport mode: Encrypts payload only (host-to-host)
- Tunnel mode: Encrypts entire packet (gateway-to-gateway)

Use Cases:
- Connecting branch offices
- Data center interconnection
- Cloud VPC to on-premises connectivity

System Design Impact:
- Requires VPN gateways
- Complex configuration (IKE policies, crypto maps)
- Performance overhead (encryption/decryption)
- Firewall rules to allow IPsec traffic
```

**2. SSL/TLS VPN:**
```
Description: Remote access VPN using SSL/TLS

Types:
- Clientless VPN: Browser-based, limited to web applications
- Full client VPN: Requires client software, full network access

Use Cases:
- Remote employee access
- Partner access
- Temporary access

System Design Impact:
- Easier deployment than IPsec
- Works through most firewalls (uses port 443)
- Application layer only (not full network)
- Requires MFA for security
```

**3. WireGuard:**
```
Description: Modern, lightweight VPN protocol

Features:
- Fewer lines of code (~4,000 vs OpenVPN ~100,000)
- Faster performance
- Simpler configuration
- Built into Linux kernel (5.6+)
- Perfect forward secrecy

Use Cases:
- High-performance VPN
- Site-to-site connectivity
- Remote access

System Design Impact:
- Easier configuration than IPsec/OpenVPN
- Better performance
- Newer protocol (less battle-tested)
- Requires kernel support or userspace implementation
```

**SSH Tunneling:**

**Why it matters for System Design:**
- SSH tunneling enables secure access to remote services
- Useful for development, debugging, and accessing restricted services
- Understanding tunneling helps design secure access patterns
- SSH tunnels can bypass firewall restrictions

**Local Port Forwarding:**
```bash
# Forward local port 8080 to remote server's localhost:80
ssh -L 8080:localhost:80 user@remote-server

# Access: http://localhost:8080 (accesses remote server's port 80)

Use Cases:
- Access remote database from local machine
- Debug remote services locally
- Access internal web applications
- Bypass firewall restrictions

System Design Impact:
- Temporary access for development/debugging
- Not suitable for production (use VPN instead)
- Requires SSH access to remote server
```

**Remote Port Forwarding:**
```bash
# Forward remote server's 8080 to local machine's localhost:3000
ssh -R 8080:localhost:3000 user@remote-server

# Access on remote server: http://localhost:8080 (accesses local machine's port 3000)

Use Cases:
- Expose local service to remote server
- Webhook development (receive callbacks)
- Share local development environment
- NAT traversal (access local machine from behind NAT)

System Design Impact:
- Useful for development and testing
- Security risk (exposes local service to remote)
- Enable GatewayPorts on remote server for external access
```

**Dynamic Port Forwarding (SOCKS Proxy):**
```bash
ssh -D 1080 user@remote-server
# Use localhost:1080 as SOCKS proxy

Use Cases:
- Tunnel all traffic through SSH
- Bypass corporate proxy
- Access geo-restricted content
- Secure browsing on public WiFi

System Design Impact:
- Acts as local SOCKS proxy
- All applications can use the proxy
- Performance overhead (SSH encryption)
```

---

## 9. Proxy Servers

### Forward Proxy

**Why it matters for System Design:**
- Forward proxies control and monitor outbound traffic
- Used for security, compliance, and content filtering
- Understanding forward proxies helps design secure network architectures
- Forward proxies can affect application behavior and performance

**Forward Proxy** sits between clients and the internet, acting on behalf of clients.

```
Client ──> Forward Proxy ──> Internet
          (Filters, logs, caches)
```

**Use Cases:**
- Access control (whitelist/blacklist sites)
- Content filtering (block inappropriate content)
- Caching frequently accessed content
- Logging and monitoring (audit trail)
- Bypassing geo-restrictions
- Anonymity (hide client IP)

**Implementation Examples:**
- Squid (open source)
- Blue Coat (enterprise)
- Microsoft Forefront TMG (Windows)

**System Design Impact:**
- Single point of control for outbound traffic
- Can add latency (additional hop)
- Requires configuration on client devices
- May break applications that assume direct internet access
- Useful for corporate environments

### Reverse Proxy

**Why it matters for System Design:**
- Reverse proxies are essential for scalable web architectures
- They provide security, performance, and flexibility
- Understanding reverse proxies helps design robust applications
- Reverse proxies are single points of entry (need HA)

**Reverse Proxy** sits between clients and backend servers, acting on behalf of servers.

```
Internet ──> Reverse Proxy ──> Backend Servers
              (Nginx, HAProxy, Apache)
```

**Functions:**

| Function | Description | System Design Impact |
|----------|-------------|------------------------|
| **Load Balancing** | Distribute traffic across backend servers | Scalability, availability |
| **SSL Termination** | Handle TLS at proxy, forward plain HTTP to backends | Reduce backend CPU load |
| **Caching** | Cache static content and responses | Reduced origin load, faster responses |
| **Compression** | Compress responses (gzip, brotli) | Reduced bandwidth, faster transfers |
| **Request Routing** | Route based on URL, headers, cookies | Microservices architecture |
| **DDoS Protection** | Rate limiting, request filtering | Improved availability |
| **WAF** | Web Application Firewall integration | Security at edge |
| **Authentication** | Handle auth at proxy edge | Offload auth from backends |

**Implementation Examples:**
- Nginx (popular, high performance)
- HAProxy (load balancing focused)
- Apache HTTP Server (mod_proxy)
- Envoy (service mesh, modern)
- Traefik (cloud-native, auto-discovery)

**System Design Impact:**
- Single point of entry (need HA with multiple proxies)
- Can add latency (but often improves overall performance with caching)
- Simplifies backend architecture (backends don't need to handle TLS, compression)
- Centralized logging and monitoring
- Enables A/B testing, canary deployments

**Nginx Reverse Proxy Configuration:**
```nginx
server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://backend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

upstream backend_cluster {
    server 10.0.0.10:80;
    server 10.0.0.11:80;
    server 10.0.0.12:80;
}
```

**Configuration Explanation:**
```
proxy_pass: Forward requests to backend cluster
proxy_set_header Host: Pass original Host header to backend
proxy_set_header X-Real-IP: Pass client IP to backend
proxy_set_header X-Forwarded-For: Chain of proxies
proxy_set_header X-Forwarded-Proto: Original protocol (http/https)

upstream: Backend server pool for load balancing
```

### Proxy vs Load Balancer

**Why it matters for System Design:**
- Understanding the differences helps choose the right tool
- Proxies and load balancers often overlap in functionality
- Modern architectures use both together
- Clear terminology improves communication

| Aspect | Forward Proxy | Reverse Proxy | Load Balancer |
|--------|---------------|---------------|---------------|
| **Position** | Client side | Server side | Server side |
| **Protects** | Clients | Servers | Servers |
| **Primary Use** | Access control, caching | Security, SSL, caching | Traffic distribution |
| **Load Balancing** | No | Can be | Yes |
| **Visibility** | Clients know proxy exists | Clients think proxy is server | Clients see single endpoint |
| **Typical Location** | Corporate network, ISP | DMZ, cloud edge | DMZ, cloud edge |
| **Example** | Squid, Blue Coat | Nginx, HAProxy | AWS ALB, F5 BIG-IP |

**Key Insight:**
- Reverse proxies often include load balancing functionality
- Load balancers often include reverse proxy functionality
- In practice, the terms are often used interchangeably
- Focus on functionality, not terminology

---

## 10. Advanced Networking Concepts

### Connection Pooling

**Why it matters for System Design:**
- Connection pooling significantly improves performance
- Reduces latency and resource usage
- Critical for database and API clients
- Understanding pooling helps design high-throughput systems

**Connection Pool** maintains a cache of database connections to avoid connection overhead.

```
Without Pooling:
Request 1: Open connection → Query → Close connection (100ms)
Request 2: Open connection → Query → Close connection (100ms)
Request 3: Open connection → Query → Close connection (100ms)
Total: 300ms

With Pooling:
Request 1: Get connection → Query → Return connection (10ms)
Request 2: Get connection → Query → Return connection (10ms)
Request 3: Get connection → Query → Return connection (10ms)
Total: 30ms
```

**Benefits:**
- Reduced latency (no connection establishment overhead)
- Reduced resource usage (fewer connections)
- Better throughput (reuse connections)
- Limits maximum connections (prevents connection exhaustion)

**Configuration Parameters:**
```java
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(20);        // Max connections
config.setMinimumIdle(5);             // Min idle connections
config.setConnectionTimeout(30000);   // Max wait for connection (ms)
config.setIdleTimeout(600000);       // Max time connection can be idle (ms)
config.setMaxLifetime(1800000);      // Max lifetime of connection (ms)
```

**System Design Impact:**
- Pool size must be tuned based on workload
- Too small: Connection wait times increase
- Too large: Database server overwhelmed
- Monitor pool metrics (active, idle, waiting connections)
- Consider connection pooling for HTTP clients too (keep-alive)

**Pool Sizing Formula (PostgreSQL Wiki):**
```
connections = ((core_count * 2) + effective_spindle_count)

Example:
- 16 cores
- 1 disk (spindle_count = 1)
- connections = (16 * 2) + 1 = 33

This is per application instance. For multiple instances, adjust accordingly.
```

**Multiple Pools:**
- Separate pools for reads and writes
- Different pool sizes for different services

### Keep-Alive

**Why it matters for System Design:**
- Keep-Alive significantly reduces connection overhead
- Critical for high-performance web applications
- Understanding keep-alive helps optimize latency
- Proper configuration balances resource usage and performance

**HTTP Keep-Alive:**
```http
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

```
Without Keep-Alive:
Request 1: TCP handshake → HTTP request → HTTP response → TCP close
Request 2: TCP handshake → HTTP request → HTTP response → TCP close
Request 3: TCP handshake → HTTP request → HTTP response → TCP close
Overhead: 3 TCP handshakes + 3 TCP closes

With Keep-Alive:
TCP handshake → Request 1 → Response 1 → Request 2 → Response 2 → Request 3 → Response 3 → TCP close
Overhead: 1 TCP handshake + 1 TCP close
```

**Benefits:**
- Reduced latency (no repeated TCP handshakes)
- Reduced CPU usage (fewer TLS handshakes if HTTPS)
- Better throughput (reuse connections)
- Faster page loads

**Configuration:**
```
Server-side (nginx):
keepalive_timeout 65;
keepalive_requests 100;

Client-side:
Connection: keep-alive header
```

**System Design Impact:**
- Keep-alive connections consume server resources
- Must balance between performance and resource usage
- Timeout should be tuned based on traffic patterns
- Monitor connection counts and resource usage
- Consider keep-alive for database connections too

**TCP Keep-Alive:**
```
Description: Detects dead peers and prevents connection table exhaustion

Linux defaults:
- tcp_keepalive_time: 7200s (2 hours) - before sending keepalive probe
- tcp_keepalive_intvl: 75s - time between probes
- tcp_keepalive_probes: 9 - number of probes before giving up

System Design Impact:
- TCP keep-alive is different from HTTP keep-alive
- TCP keep-alive detects dead connections
- Useful for long-lived connections (WebSocket, database)
- Can be tuned for faster failure detection
```

**When to Use:**
```
- Long-running connections (WebSocket, gRPC)
- High-throughput scenarios
- Avoid connection overhead
- HTTP/1.1 (keep-alive is default)
- HTTP/2 (multiplexing requires persistent connections)
```

**When to Disable:**
```
- One-time requests
- Resource-constrained environments (many idle connections)
- When connection pooling is not used
- Legacy HTTP/1.0 clients
```

---

### Circuit Breakers

**Why it matters for System Design:**
- Circuit breakers prevent cascade failures in distributed systems
- Critical for microservice architectures
- Understanding circuit breakers helps design resilient systems
- Circuit breakers improve system availability during failures

**Circuit Breaker Pattern** prevents cascade failures in distributed systems.

**States:**
```
        ┌─────────────┐
        │   CLOSED    │ ←── Normal operation
        │  (requests  │
        │   allowed)  │
        └──────┬──────┘
               │ Failure threshold reached
               ▼
        ┌─────────────┐
        │    OPEN     │ ←── Fail fast
        │ (requests   │     (return error immediately)
        │  blocked)   │
        └──────┬──────┘
               │ Timeout expires
               ▼
        ┌─────────────┐
        │  HALF-OPEN  │ ←── Test if service recovered
        │ (allow one  │
        │  request)   │
        └──────┬──────┘
               │
               ├──── Success → CLOSED
               │
               └──── Failure → OPEN
```

**Configuration Parameters:**
```
- Failure Threshold: Number of failures before opening circuit
- Timeout: How long to stay in OPEN state
- Success Threshold: Number of successes in HALF-OPEN before closing
```

**Example:**
```
Configuration:
- Failure threshold: 5 failures
- Timeout: 60 seconds
- Success threshold: 2 successes

Scenario:
1. Service is working normally (CLOSED state)
2. 5 consecutive failures occur
3. Circuit opens (OPEN state) - requests fail fast
4. After 60 seconds, circuit moves to HALF-OPEN
5. Test request succeeds
6. Second test request succeeds
7. Circuit closes (CLOSED state) - normal operation resumes
```

**System Design Impact:**
- Prevents cascade failures
- Improves system availability
- Adds complexity (need to tune thresholds)
- Requires monitoring and alerting
- Can be implemented at client or service mesh level

**Configuration Example:**
```yaml
failureThreshold: 5        # Open after 5 failures
successThreshold: 3        # Close after 3 successes
timeout: 60s               # Time before HALF-OPEN
halfOpenMaxCalls: 1        # Test calls in HALF-OPEN
slowCallThreshold: 2s      # Consider slow as failure
```

**Implementations:**
- Hystrix (Netflix - maintenance mode)
- Resilience4j (Java)
- Polly (.NET)
- go-breaker (Go)
- Sentinel (Alibaba - Java/Go)
- Istio (service mesh)
- AWS App Mesh (service mesh)

---

### Service Mesh

**Why it matters for System Design:**
- Service meshes provide infrastructure layer for service-to-service communication
- Critical for microservice architectures
- Understanding service meshes helps design observable, secure, and resilient systems
- Service meshes add complexity but solve cross-cutting concerns

**Service Mesh** provides infrastructure layer for service-to-service communication.

**Architecture:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Service   │────▶│ Sidecar     │◀────│   Service   │
│     A       │     │ Proxy       │     │     B       │
└─────────────┘     └─────────────┘     └─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │ Control     │
                   │ Plane       │
                   │ (config,    │
                   │  metrics)   │
                   └─────────────┘
```

**Key Components:**
- **Data Plane:** Sidecar proxies (Envoy) handle all network traffic
- **Control Plane:** Manages and configures proxies (Istiod, Pilot)

**Features:**

| Feature | Description | System Design Impact |
|---------|-------------|------------------------|
| **Service Discovery** | Automatic service registration and discovery | No need for manual service registry |
| **Load Balancing** | Built-in load balancing algorithms | No need for external load balancer |
| **TLS/mTLS** | Automatic mutual TLS between services | Zero-trust security |
| **Circuit Breaking** | Built-in circuit breaker pattern | Resilience without code changes |
| **Retry/Timeout** | Automatic retry and timeout policies | Improved reliability |
| **Observability** | Metrics, logs, tracing out of the box | Deep visibility into service communication |
| **Traffic Shifting** | Canary deployments, A/B testing | Safer deployments |
| **Rate Limiting** | Per-service rate limiting | Protection from overload |

**Implementations:**
- **Istio:** Most popular, Kubernetes-native
- **Linkerd:** Lightweight, simpler
- **Consul Connect:** HashiCorp ecosystem
- **AWS App Mesh:** AWS-native
- **Linkerd:** Rust-based, lightweight

**System Design Impact:**
- **Pros:**
  - Solves cross-cutting concerns (security, observability, reliability)
  - Language-agnostic (works with any language)
  - No code changes required for many features
  - Consistent policies across all services

- **Cons:**
  - Adds complexity to infrastructure
  - Additional latency (sidecar proxy hop)
  - Operational overhead (managing control plane)
  - Learning curve
  - Resource overhead (sidecar per service)

**When to Use:**
- Large microservice architectures (10+ services)
- Polyglot environments (multiple languages)
- Strong security requirements (mTLS)
- Need deep observability
- Multiple teams managing different services

**When to Avoid:**
- Small number of services (monolith or few microservices)
- Single language (can use libraries instead)
- Resource-constrained environments
- Simple deployments

---

### API Gateway

**Why it matters for System Design:**
- API gateways are the single entry point for client requests
- Critical for microservice architectures
- Understanding API gateways helps design secure, observable APIs
- API gateways simplify client complexity

**API Gateway** is a single entry point for client requests to multiple backend services.

**Architecture:**
```
Clients (Web, Mobile, IoT)
         │
         ▼
┌─────────────────┐
│   API Gateway   │
│  (Kong, AWS API │
│   Gateway, NGINX)│
└────────┬────────┘
         │
    ┌────┴────┬─────────┬─────────┐
    ▼         ▼         ▼         ▼
 Service A  Service B  Service C  Service D
```

**Functions:**

| Function | Description | System Design Impact |
|----------|-------------|------------------------|
| **Request Routing** | Route requests to appropriate backend services | Clients don't need to know service locations |
| **Authentication/Authorization** | Handle auth at gateway (OAuth, JWT) | Centralized auth, no code in services |
| **Rate Limiting** | Limit request rate per client/API | Protection from abuse |
| **Request/Response Transformation** | Modify requests/responses (protocol translation) | Backend services can be simpler |
| **SSL Termination** | Handle TLS at gateway | Reduce backend CPU load |
| **Caching** | Cache responses | Reduced backend load, faster responses |
| **API Versioning** | Route based on API version | Smooth API evolution |
| **Logging/Monitoring** | Centralized logging and metrics | Observability across all services |
| **Request Validation** | Validate requests before reaching backend | Protect backends from invalid requests |

**API Gateway vs Service Mesh:**
```
API Gateway:
- Entry point for external clients
- North-South traffic (client to service)
- Edge of the system
- Focus: API management, client-facing features

Service Mesh:
- Service-to-service communication
- East-West traffic (service to service)
- Internal to the system
- Focus: Service communication, internal reliability

Both can be used together:
API Gateway at edge → Service Mesh internally
```

**Implementations:**
- **Kong:** Open source, plugin-rich
- **AWS API Gateway:** AWS-native, serverless
- **Azure API Management:** Azure-native
- **Apigee:** Google Cloud, enterprise
- **NGINX:** Lightweight, flexible
- **Traefik:** Cloud-native, auto-discovery

**System Design Impact:**
- **Pros:**
  - Single entry point simplifies client complexity
  - Centralized cross-cutting concerns (auth, rate limiting)
  - Backend services can be simpler
  - API versioning and evolution easier

- **Cons:**
  - Single point of failure (need HA)
  - Additional latency (extra hop)
  - Can become bottleneck (need scaling)
  - Adds operational complexity

**When to Use:**
- Microservice architectures
- Multiple client types (web, mobile, IoT)
- Need centralized auth and rate limiting
- API versioning requirements
- External API exposure

**When to Avoid:**
- Monolithic applications (can handle routing in app)
- Simple architecture with few services
- Performance-critical (minimal latency required)
- Direct service communication needed

---

**API Gateway vs Load Balancer:**
| Feature | API Gateway | Load Balancer |
|---------|-------------|---------------|
| **Level** | Application (L7) | L4 or L7 |
| **Routing** | Path-based, header-based | IP:Port-based |
| **Auth** | Built-in | Limited |
| **Rate Limit** | Advanced | Basic |
| **Transformation** | Yes | No |
| **Protocol Translation** | Yes | No |

---

## 11. Performance and Optimization

### Latency vs Bandwidth

**Why it matters for System Design:**
- Understanding latency vs bandwidth helps diagnose performance issues
- Different strategies are needed for latency vs bandwidth problems
- Critical for designing high-performance systems
- Network performance affects user experience

**Latency:** Time for single bit to travel (speed)
- Measured in milliseconds (ms)
- Limited by speed of light and distance
- Affects response time, user experience
- Cannot be reduced below physical limits

**Bandwidth:** Amount of data that can be transferred (capacity)
- Measured in bits per second (bps) or bytes per second (B/s)
- Limited by network infrastructure
- Affects throughput, data transfer speed
- Can be increased by adding more capacity

**Analogy:**
```
Latency = Speed limit on a highway (how fast you can drive)
Bandwidth = Number of lanes on a highway (how many cars can travel simultaneously)

High latency, high bandwidth: Wide highway with low speed limit
Low latency, low bandwidth: Narrow highway with high speed limit
Low latency, high bandwidth: Wide highway with high speed limit (ideal)
```

**Real-world Impact:**
```
Scenario 1: High Latency (e.g., satellite internet)
- Bandwidth: 100 Mbps (high)
- Latency: 500ms (high)
- Impact: Slow response times, but large files download quickly once started

Scenario 2: Low Bandwidth (e.g., 3G mobile)
- Bandwidth: 5 Mbps (low)
- Latency: 50ms (low)
- Impact: Fast response times, but large files download slowly

Scenario 3: High Latency + Low Bandwidth (worst case)
- Bandwidth: 5 Mbps (low)
- Latency: 500ms (high)
- Impact: Slow response times AND slow downloads
```

**System Design Impact:**
- **Latency-sensitive applications:** Gaming, video conferencing, trading systems
  - Focus on reducing latency (edge computing, CDN, protocol optimization)
  
- **Bandwidth-sensitive applications:** File transfer, video streaming, backup
  - Focus on increasing bandwidth (CDN, compression, parallel transfers)

- **Optimization strategies:**
  - Latency: Caching, CDN, edge computing, protocol optimization (HTTP/2, QUIC)
  - Bandwidth: Compression, data deduplication, parallel transfers, CDN

---

### Network Jitter

**Why it matters for System Design:**
- Jitter affects real-time applications (VoIP, video conferencing, gaming)
- Understanding jitter helps design smooth user experiences
- Jitter mitigation adds complexity to systems
- Critical for latency-sensitive applications

**Jitter** is variation in packet arrival times.

```
Example:
Packet 1 arrives at: 0ms
Packet 2 arrives at: 10ms (jitter: 10ms)
Packet 3 arrives at: 12ms (jitter: 2ms)
Packet 4 arrives at: 25ms (jitter: 13ms)

Average jitter: 8.33ms
```

**Impact:**
- VoIP/video quality degradation (choppy audio/video)
- Gaming lag spikes (inconsistent gameplay)
- Inconsistent user experience
- Buffering issues in streaming

**Mitigation:**
- **Jitter Buffers:** Temporarily store packets to smooth arrival
  - Trade-off: Increased latency for smoother playback
  - Adaptive buffers adjust based on jitter levels
  
- **QoS (Quality of Service):** Prioritize time-sensitive traffic
  - DSCP marking for priority queuing
  - Traffic shaping to prevent congestion
  
- **UDP for real-time:** Accept some loss over latency
  - TCP's retransmission causes more jitter
  - UDP with application-level recovery preferred

### Packet Loss Handling

**Why it matters for System Design:**
- Packet loss affects application performance and user experience
- Understanding packet loss helps design resilient systems
- Different protocols handle loss differently (TCP vs UDP)
- Loss mitigation strategies depend on application requirements

**Causes of Packet Loss:**
- Network congestion (router buffer overflow)
- Link errors (noise, interference)
- Faulty hardware (bad cables, failing NICs)
- Misconfigured QoS
- Software bugs

**Detection:**
- **TCP:** Missing ACKs trigger retransmission
- **UDP:** Application-level detection needed (sequence numbers, timestamps)

**Handling Strategies:**

**TCP (Automatic):**
```
- Fast retransmit on 3 duplicate ACKs
  - Detects loss quickly without waiting for timeout
  - Retransmits missing packet immediately
  
- Retransmission timeout (RTO)
  - Exponential backoff on consecutive timeouts
  - Adaptive based on measured RTT
  
- Selective Acknowledgment (SACK)
  - Receiver tells sender exactly which packets were received
  - More efficient than cumulative ACKs
```

**UDP (Application-Level):**
```
- Forward Error Correction (FEC): Send redundant data
  - Allows reconstruction without retransmission
  - Adds overhead but reduces latency
  
- Negative Acknowledgment (NACK): Request specific missing packets
  - Receiver explicitly requests missing packets
  - More efficient than ACKs for high loss rates
  
- Playout buffers: Wait for late packets
  - Trade latency for completeness
  - Used in streaming and VoIP
  
- Adaptive bitrate: Lower quality on loss
  - Reduce bandwidth requirements
  - Maintain smooth playback
```

**Recovery in Video Streaming:**
- I-frames, P-frames, B-frames hierarchy
- Concealment: Use previous frame
- Retransmit key frames only

### Latency Numbers

**Why it matters for System Design:**
- Understanding latency numbers helps estimate system performance
- Critical for designing low-latency applications
- Helps identify bottlenecks in system design
- Guides optimization priorities

**Jeff Dean's Latency Numbers Every Programmer Should Know (2024 approximate):**

```
L1 cache reference:                           ~1 ns
L2 cache reference:                          ~4 ns
L3 cache reference:                          ~12 ns
Main memory reference:                        ~100 ns
Compress 1KB with Zippy:                    ~3,000 ns (3 µs)
Send 1KB over 1 Gbps network:               ~10,000 ns (10 µs)
Read 1MB sequentially from memory:          ~250,000 ns (250 µs)
Read 1MB sequentially from SSD:          ~1,000,000 ns (1 ms)
Read 1MB sequentially from disk:        ~10,000,000 ns (10 ms)
Disk seek:                              ~10,000,000 ns (10 ms)
Read 1MB sequentially from network:    ~10,000,000 ns (10 ms)
Packet round trip in same datacenter:    ~500,000 ns (0.5 ms)
Packet round trip between continents:  ~100,000,000 ns (100 ms)
```

**System Design Implications:**
```
Example: Database query optimization

Scenario:
- Query returns 100 rows from disk
- Each row requires disk seek: 10ms
- Total time: 100 * 10ms = 1 second

Optimization:
- Add index: Data in memory or sequential read
- Total time: 100 * 100ns (memory) = 10µs
- Improvement: 100,000x faster

Key insight:
- Disk seeks are expensive (10ms)
- Memory access is cheap (100ns)
- Network round trips are expensive (0.5-100ms)
- Caching can provide massive performance improvements
```

**Latency Comparison Table:**

| Operation | Latency | Scaled (1s = 1ns) | System Design Impact |
|-----------|---------|-------------------|---------------------|
| L1 cache reference | 0.5 ns | 0.5 s | Cache hits are critical |
| Branch mispredict | 5 ns | 5 s | Code optimization matters |
| L2 cache reference | 7 ns | 7 s | Cache hierarchy design |
| Mutex lock/unlock | 25 ns | 25 s | Lock contention hurts performance |
| Main memory reference | 100 ns | 1.7 min | Memory bandwidth is key |
| Send 2K bytes over 1 Gbps network | 20 μs | 5.5 hours | Network latency is significant |
| Read 1 MB sequentially from memory | 250 μs | 2.9 days | Sequential reads are faster |
| Round trip within datacenter | 500 μs | 5.8 days | Same DC RTT is non-trivial |
| Disk seek | 10 ms | 11.6 days | Disk seeks are very expensive |
| Read 1 MB sequentially from disk | 20 ms | 23 days | Disk I/O is a bottleneck |
| Packet round trip between continents | 100 ms | 115 days | Geographic distance matters |

**Key Takeaways:**
```
1. Memory is 100,000x faster than disk
   - Cache everything possible
   - Avoid disk seeks
   
2. Network RTT is significant
   - Minimize round trips
   - Batch requests
   - Use connection pooling
   
3. Disk I/O is the bottleneck
   - Use SSDs when possible
   - Optimize for sequential access
   - Cache aggressively
   
4. CPU operations are fast
   - Don't prematurely optimize code
   - Focus on I/O, not CPU
```

---

## Additional Resources

### Books
- "Computer Networking: A Top-Down Approach" by Kurose & Ross
- "TCP/IP Illustrated, Volume 1" by W. Richard Stevens
- "High Performance Browser Networking" by Ilya Grigorik
- "Designing Data-Intensive Applications" by Martin Kleppmann

### Online Resources
- [Cloudflare Learning Center](https://www.cloudflare.com/learning/)
- [MDN Web Docs - HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [Google SRE Book - Networking](https://sre.google/sre-book/table-of-contents/)

### Tools for Network Debugging
- `ping` - Basic connectivity
- `traceroute` / `tracert` - Route tracking
- `netstat` / `ss` - Socket statistics
- `tcpdump` / `wireshark` - Packet capture
- `curl` / `httpie` - HTTP debugging
- `nmap` - Network scanning
- `iperf` - Bandwidth measurement
- `dig` / `nslookup` - DNS debugging
- `openssl s_client` - TLS debugging

---

## Summary Checklist for System Design

When designing systems, ensure you consider:

- [ ] **Protocol Selection:** TCP vs UDP vs HTTP/2 vs gRPC
- [ ] **Connection Management:** Pooling, keep-alive, timeouts
- [ ] **Caching Strategy:** Multiple layers (browser, CDN, application, database)
- [ ] **Load Balancing:** Algorithm choice, health checks, session handling
- [ ] **Security:** TLS version, cipher suites, certificate management
- [ ] **DNS:** TTL settings, failover, geographic routing
- [ ] **Resilience:** Circuit breakers, retries, exponential backoff
- [ ] **Observability:** Metrics, logging, distributed tracing
- [ ] **Scalability:** Horizontal scaling, statelessness, shared nothing
- [ ] **Latency Optimization:** CDNs, connection reuse, payload size

---

*This guide covers networking fundamentals essential for system design interviews and building scalable distributed systems. For deeper understanding, experiment with actual implementations and analyze real-world system architectures.*
