# Complete End-to-End Guest Network Topology
## Full Traffic Path: Guest Device → Internet Access

---

## EXECUTIVE OVERVIEW

This document details the **complete path** of a guest user connecting from a remote site, authenticating through Cisco ISE, and accessing the internet. Every device, firewall rule, and security boundary is documented.

**Total Devices in Traffic Path: 19**
- **Remote Site:** 4 devices
- **WAN/SDWAN:** 2 devices  
- **Data Center - Perimeter:** 1 device
- **Data Center - DMZ:** 4 devices
- **Data Center - Trusted:** 5 devices
- **Internet:** 3 zones

---

## SECTION 1: REMOTE SITE (Step 1-3)

### 1.1 Guest Device
**Device:** User's Laptop, Phone, or Tablet  
**Location:** Remote office, conference room, or visitor area  
**Function:** Initiates network connection  
**Specifications:**
- No pre-existing credentials
- Connects via WiFi (802.11ac/ax)
- Initial VLAN assignment: **Unauthenticated / Guest VLAN 10**
- Assigned IP: DHCP from unauthenticated pool (e.g., 10.0.1.100)
- No access to corporate resources

**Traffic Originating from Device:**
```
Source MAC: Device MAC address
Source IP: Initially 0.0.0.0 (DHCP request)
Destination: Cisco AP broadcast
Protocol: 802.11 probe request
```

---

### 1.2 Cisco Access Point (AP)
**Device:** Cisco Aironet AP (Model: 2800/3800/4800 series)  
**Location:** Mounted in ceiling/wall at remote site  
**Function:** Wireless transmission bridge  
**Specifications:**
- Operates in **bridge mode** (not autonomous)
- All policy enforcement delegated to WLC
- Sends frames to Foreign WLC over wired connection
- Supports WPA2/WPA3 encryption
- 802.11ac/ax standards (5 GHz + 2.4 GHz)

**Traffic Role:**
```
Receives: WiFi probe requests from device
Encapsulates: Frame into 802.11 over LWAPP (lightweight)
Forwards to: Foreign WLC IP on wired network (vlan-based)
Encryption: DTLS tunnel to WLC (default)
```

---

### 1.3 Foreign Wireless LAN Controller (WLC)
**Device:** Cisco Catalyst 9800 WLC (Remote Site)  
**Model:** WLC-9800-CL or WLC-9800-40 appliance  
**Location:** Network closet or data room at remote site  
**Primary Function:** **Guest Policy Server (Remote)**  
**Specifications:**
- Manages all guest authentication requests from the AP
- Handles initial MAB (MAC-based authentication)
- Redirects guest browser to portal
- Issues RADIUS Change-of-Authorization (CoA) for VLAN assignment
- **Does NOT store credentials** — forwards to DMZ ISE for validation

**Key Ports/Protocols:**
- RADIUS Authentication: UDP 1812 (to DMZ ISE)
- RADIUS Accounting: UDP 1813 (to DMZ ISE)
- SSH Management: TCP 22 (admin only)
- HTTPS UI: TCP 443 (admin UI)
- LWAPP/CAPWAP: UDP 5246, 5247 (AP communication)

**Traffic Flow - Foreign WLC Processes:**
```
Step 1: Receives WiFi association from Device via AP
   └─ 802.11 probe → AP encapsulates → Forwarded to WLC

Step 2: Sends MAB Authentication Request to DMZ ISE
   └─ Source: Foreign WLC IP (10.10.10.50)
   └─ Destination: ISE-DMZ-01 (10.20.20.100) Port 1812
   └─ Protocol: RADIUS Request (MAB - MAC only)
   └─ Content: Device MAC, AP name, switch port

Step 3: Receives HTTP Response from ISE
   └─ RADIUS Access-Challenge with portal URL
   └─ WLC generates HTTP 301 redirect
   └─ Sends to device: "Go to https://guest-portal.company.com"

Step 4: Device Opens Browser
   └─ Device gets redirected to guest portal
   └─ Portal hosted at ISE-DMZ nodes

Step 5: After Credentials Valid (from ISE)
   └─ ISE sends RADIUS Access-Accept + CoA
   └─ Includes VLAN change instruction
   └─ Foreign WLC moves device to Guest VLAN 
   └─ Sends RADIUS Accounting Start/Stop messages
```

---

### 1.4 SDWAN Network Appliance (Remote Site)
**Device:** Orange Equipment SDWAN NAS  
**Location:** Same facility as Foreign WLC  
**Function:** **Encryption tunnel to data center**  
**Specifications:**
- Encrypts all traffic from remote site to DC
- Provides redundant paths (if ISP-1 fails, auto-failover to ISP-2)
- Performs IPSec encryption (AES-256 typical)
- Zero-trust tunnel — no cleartext traffic on WAN

**Traffic Tunnel Characteristics:**
```
Encrypted Packets:
   Source: Remote Site SDWAN IP (ISP IP)
   Destination: DC SDWAN IP (ISP IP)
   Encryption: IPSec / AES-256
   Protocol: IKEv2 (encryption negotiation)
   
All guest traffic flows through this tunnel:
   - RADIUS auth traffic (port 1812/1813)
   - HTTP/HTTPS from guest device (ports 80/443)
   - Return traffic from internet
```

---

## SECTION 2: WAN / SDWAN CONNECTION (Step 4)

### 2.1 SDWAN Tunnel
**Type:** Encrypted site-to-site VPN  
**Encryption:** IPSec with AES-256  
**Path:** Remote Site SDWAN NAS ↔ Data Center SDWAN NAS  
**Redundancy:** Multi-ISP failover (if primary ISP down, secondary activates)  
**Latency Requirement:** < 200ms (Cisco recommendation for ISE replication)

**What Travels Through Tunnel:**
```
✓ RADIUS authentication requests (UDP 1812)
✓ RADIUS accounting (UDP 1813)
✓ Guest HTTP/HTTPS traffic (TCP 80, 443)
✓ DNS queries from guest device (UDP 53)
✓ Return packets from internet (all protocols)

✗ Blocked: Corporate traffic (separate tunnel)
✗ Blocked: Management traffic (separate tunnel if needed)
```

---

### 2.2 Data Center SDWAN Appliance
**Device:** Orange Equipment SDWAN NAS (DC side)  
**Location:** Data center entrance, DMZ  
**Function:** Decrypts incoming SDWAN tunnel, routes traffic  
**Specifications:**
- Decrypts IPSec from remote site
- Routes authenticated packets to Perimeter Firewall
- Handles failover if remote WLC goes down
- Integrates with DC routing

---

## SECTION 3: DATA CENTER — PERIMETER SECURITY (Step 5)

### 3.1 Internet (Outside, Untrusted Zone)
**Zone:** Public internet  
**Devices:** ISP gateways, DNS servers (8.8.8.8), public web servers  
**Function:** Guest's destination for browsing

**Firewall Rule (Inbound):**
```
Rule: DEFAULT DENY
Only exceptions:
   ✓ Response packets to established sessions
   ✗ All unsolicited inbound traffic blocked
   ✗ DDoS mitigation: Rate limiting, GeoIP blocks
   ✗ Threat filtering: Malware signatures updated hourly
```

---

### 3.2 ISP Uplink Connection
**Device:** ISP Gateway (Comcast, AT&T, Level3, etc.)  
**Function:** Internet entry point  
**Specifications:**
- **Redundant connections:** Primary + backup ISP
- **BGP Advertised Routes:** Corporate public IP ranges
- **Bandwidth:** 1 Gbps / 10 Gbps (depends on contract)
- **NAT/PAT Handled By:** Corporate edge router (inside trusted zone)

**Traffic Direction:**
```
OUTBOUND (Guest to Internet):
   Source IP: Corporate Public IP (NAT'd)
   Destination: Internet servers
   Port: 80 (HTTP), 443 (HTTPS)
   
INBOUND (Internet Response):
   Source IP: Internet server
   Destination: Corporate Public IP
   Port: > 1024 (ephemeral)
```

---

### 3.3 Perimeter Firewall (Outside)
**Device:** Cisco ASA 5520 or Catalyst ISR 4000  
**Location:** Data center edge, internet-facing  
**Function:** **First line of defense against internet threats**

**Critical Specifications:**
- **Default Policy:** DENY all inbound
- **Threat Protection:** DDoS, malware scanning, botnet detection
- **Stateful Inspection:** Tracks all outbound connections
- **Ports Protected:** DNS (53), HTTPS (443), HTTP (80)

**Firewall Rules (Simplified):**
```
1. Rule: Block all inbound traffic
   └─ Except: Response packets to outbound connections
   └─ Except: Configured public services (web, mail, VPN)

2. Rule: DDoS Protection
   └─ Rate limit: 10,000 packets/sec per source
   └─ Geo-blocking: Block traffic from suspicious countries
   └─ SYN flood protection: Enable

3. Rule: Threat Inspection
   └─ IDS/IPS: Active scanning for CVEs
   └─ Malware Signatures: Updated from Cisco Threat Feed
   └─ File Blocking: Executable files from internet blocked

4. Rule: NAT Configuration
   └─ Outside Interface: ISP IP (public)
   └─ Inside Interface: Corporate network (private)
   └─ Translation: Inside IP 10.x.x.x → Outside IP (public)
```

**Traffic Through Perimeter FW (Inbound):**
```
Source: ISP Uplink
Destination: This firewall's interface
Inspection: DDoS patterns, threat signatures
Action: Allow response traffic only
        Drop unsolicited packets
```

**Traffic Through Perimeter FW (Outbound):**
```
Source: Corporate network (any subnet)
Destination: ISP Uplink
Inspection: Stateful tracking
Action: Allow (response expected)
Log: All connections for audit trail
```

---

## SECTION 4: DATA CENTER — DMZ (Guest Zone) (Step 6)

### 4.1 DMZ Firewall #1 (Outer Boundary)
**Device:** Cisco ASA 5520 or Catalyst ISR 4000  
**Location:** Between Perimeter FW and DMZ systems  
**Function:** **Validate guest traffic, isolate from corporate**

**DMZ Firewall Configuration:**
```
INBOUND Rules (from WAN/SDWAN):
1. Allow: SDWAN tunnel from remote site (IP-based)
   └─ Source: Remote SDWAN IP
   └─ Destination: DC SDWAN IP
   └─ Protocol: IPSec (IKEv2, UDP 500/4500)
   └─ Action: ALLOW

2. Allow: RADIUS from Foreign WLC
   └─ Source: Foreign WLC IP (10.10.10.50)
   └─ Destination: ISE-DMZ-01/02 (10.20.20.100-101)
   └─ Port: 1812, 1813 (RADIUS)
   └─ Action: ALLOW

3. Allow: HTTP/HTTPS from guest devices
   └─ Source: Guest VLAN (10.0.x.x)
   └─ Destination: ISE Portal (10.20.20.100:80, :8443)
   └─ Port: 80 (HTTP), 8443 (HTTPS)
   └─ Action: ALLOW
   └─ Duration: Until credentials valid (CoA issued)

4. Block: Guest to Trusted Network
   └─ Source: Guest VLAN (10.0.x.x)
   └─ Destination: Trusted Zone (10.30.x.x)
   └─ Action: DENY
   └─ Log: All denied traffic (policy violation)

OUTBOUND Rules (from DMZ to Internet/Trusted):
1. Allow: ISE to Trusted ISE (admin only)
   └─ Source: ISE-DMZ nodes
   └─ Destination: Primary ISE (10.30.30.50)
   └─ Port: 8443 (admin), 3000 (pxGrid)
   └─ Action: ALLOW

2. Block: Guest devices to corporate resources
   └─ Source: Guest VLAN
   └─ Destination: Trusted networks, databases
   └─ Action: DENY (absolute barrier)

3. Allow: DNS queries (limited)
   └─ Source: Guest VLAN
   └─ Destination: External DNS (8.8.8.8)
   └─ Port: 53 (DNS)
   └─ Action: ALLOW

4. Redirect: HTTP traffic to portal
   └─ If unauthenticated: Intercept port 80
   └─ Redirect: To ISE guest portal
   └─ Action: HTTP 301 Moved Permanently
```

---

### 4.2 SDWAN Network Appliance (DC Side)
**Device:** Orange Equipment SDWAN NAS  
**Function:** Decrypt tunnel, route to load balancer  
**Specifications:**
- Decrypts IPSec packets from remote site
- Extracts guest traffic and forwards to Load Balancer
- Maintains symmetric routing (return packets same path)

---

### 4.3 Load Balancer
**Device:** Citrix NetScaler or F5 BIG-IP  
**Location:** In front of PSN pool  
**Function:** **Distribute guest authentication requests across ISE nodes**

**Load Balancer Configuration:**
```
Pool Members:
   - ISE-DMZ-01 (10.20.20.100)
   - ISE-DMZ-02 (10.20.20.101)

Algorithm: Round-robin with session stickiness

Stickiness: Source MAC address (Calling Station ID)
   └─ Ensures same device always goes to same ISE
   └─ Prevents session loss on failover

Health Checks:
   - Every 5 seconds
   - Port: 8443 (HTTPS)
   - If ISE-DMZ-01 down: Route to ISE-DMZ-02
   - If both down: Return 503 (Service Unavailable)

Ports Published:
   - Port 80 (HTTP) → ISE-DMZ node port 80
   - Port 8443 (HTTPS) → ISE-DMZ node port 8443
   - Port 3000 (pxGrid) → ISE admin port
```

---

### 4.4 Anchor Wireless LAN Controller
**Device:** Cisco Catalyst 9800 WLC (Data Center)  
**Models:** WLC-9800-40 (primary + secondary for HA)  
**Location:** Data center, DMZ segment  
**Function:** **Policy Server and enforcement point for guest access**

**Anchor WLC Specifications:**
```
Deployment: Clone Zone HA (Active/Active)
   - WLC-Anchor-01 (10.20.20.10)
   - WLC-Anchor-02 (10.20.20.11)
   - Shared virtual IP: 10.20.20.12

Database Replication: Real-time sync
   - Guest user sessions
   - Authorization policies
   - ACL definitions
   - Failover: < 1 second

RADIUS Server Configuration:
   - Primary: ISE-DMZ-01 (10.20.20.100:1812)
   - Secondary: ISE-DMZ-02 (10.20.20.101:1812)
   - Shared Secret: [encrypted]
   - Timeout: 3 seconds, 2 retries

Guest Policies Configured:
   1. Guest authentication method: MAB (MAC)
   2. Guest VLAN: 10 (unauthenticated)
   3. Authorized VLAN: 11 (after ISE validation)
   4. Guest ACL: GuestACL_Internet_Only
      └─ Allow: TCP/UDP ports 80, 443 (HTTP/HTTPS)
      └─ Allow: UDP port 53 (DNS)
      └─ Block: All other protocols
      └─ Block: Corporate networks (10.30.0.0/16)
```

**Anchor WLC Roles:**
```
Role 1: RADIUS Server for ISE
   └─ Receives authentication requests from Foreign WLC
   └─ Forwards to ISE-DMZ nodes via RADIUS
   └─ Receives responses and sends CoA

Role 2: Policy Enforcement
   └─ Applies guest ACL to authenticated devices
   └─ Enforces bandwidth limits (if configured)
   └─ Terminates sessions on timeout

Role 3: Redundancy Point
   └─ If Foreign WLC loses ISE connection
   └─ Anchor WLC can cache credentials
   └─ Supports local authentication fallback
```

---

### 4.5 ISE Nodes (DMZ) — Guest Authentication Engines
**Device:** Cisco Identity Services Engine (ISE 3.4)  
**Models:** ISE-3795 appliances  
**Nodes:** ISE-DMZ-01 (10.20.20.100), ISE-DMZ-02 (10.20.20.101)  
**Location:** Data center, DMZ segment  
**Function:** **Guest credential validation and portal hosting**

**ISE-DMZ Node Specifications:**
```
Personas Enabled:
   ✓ Policy Service (guest portal, profiling)
   ✓ Not Administration (no config changes)
   ✓ Not Monitoring (logging separate)

Deployment Type: Split (HA pair with load balancer)
Synchronization: Real-time replication of guest sessions
Database: Guest user accounts, authorization rules

Ports Exposed:
   - Port 80 (HTTP): Guest portal redirect
   - Port 8443 (HTTPS): Secure portal, admin API
   - Port 1812 (RADIUS): Authentication requests
   - Port 1813 (RADIUS): Accounting packets
   - Port 3000 (pxGrid): Policy exchange
   - Port 3270 (REST API): Automation

Storage Capacity: 10 million guest accounts
Session Timeout: 24 hours (configurable)
```

**Guest Portal Hosted By ISE-DMZ:**
```
Portal Type: Self-Registration
URL: https://guest-portal.company.com (hosted at load balancer)

Portal Flow:
1. User clicks "Connect to Guest WiFi"
   └─ Device navigates to captive portal
   └─ Browser captured, redirected to login page

2. Portal loads (from ISE-DMZ node)
   └─ Displays company logo, terms
   └─ Provides username/password entry fields

3. User enters credentials
   └─ Optional: Email for sending access code
   └─ Optional: Terms of Use acceptance

4. ISE validates credentials
   └─ Checks against guest database
   └─ Verifies device MAC against blacklist
   └─ Checks device posture (if required)

5. Sends RADIUS Access-Accept to Anchor WLC
   └─ Includes CoA (Change of Authorization)
   └─ Instructs: Move device to Guest VLAN 11
   └─ Sets session timeout: 24 hours

6. Anchor WLC sends CoA to Foreign WLC
   └─ Signal: Re-authenticate this device
   └─ New VLAN: 11 (authorized guest)
   └─ ACL: GuestACL_Internet_Only

7. Guest device re-authenticates
   └─ Gets new IP from DHCP (10.0.2.x now authorized)
   └─ Internet connectivity activated
   └─ Portal closes automatically
```

**ISE-DMZ Session Logging:**
```
Every guest session creates an audit log:
   - Device MAC: AA:BB:CC:DD:EE:FF
   - Username: john.doe@company.com
   - Login time: 2026-07-08 14:35:22 UTC
   - VLAN assigned: 11 (Guest)
   - IP Address: 10.0.2.105
   - Session timeout: 24 hours
   - Posture status: COMPLIANT
   - Authorization profile: Guest_Internet
   - Bandwidth limit: Unlimited (or 10 Mbps)

Logs stored:
   - ISE-DMZ local database (7 days)
   - Syslog Server (90 days retention)
   - SIEM integration (Splunk/ELK)
```

---

## SECTION 5: DATA CENTER — TRUST BOUNDARY (Step 7)

### 5.1 Internal Firewall (Trust Boundary)
**Device:** Cisco ASA 5520 or Catalyst ISR 4000  
**Location:** Between DMZ and Trusted Zone  
**Function:** **Absolute barrier between guest and corporate network**

**Trust Boundary Firewall Rules:**
```
INBOUND from DMZ:
1. Rule: Block all guest traffic to Trusted
   └─ Source: Guest VLAN (10.0.x.x)
   └─ Destination: Any Trusted address (10.30.x.x)
   └─ Action: DENY
   └─ Log: CRITICAL (all attempts logged)

2. Rule: Block guest access to databases
   └─ Source: Guest VLAN
   └─ Destination: Database servers (10.30.200.x)
   └─ Port: 1433, 5432, 3306 (SQL ports)
   └─ Action: DENY

3. Rule: Block guest access to file servers
   └─ Source: Guest VLAN
   └─ Destination: File servers (10.30.100.x)
   └─ Port: 445, 139 (SMB/CIFS)
   └─ Action: DENY

OUTBOUND from Trusted:
1. Rule: Allow ISE-DMZ to Primary ISE (admin only)
   └─ Source: ISE-DMZ nodes
   └─ Destination: Primary ISE (10.30.30.50)
   └─ Port: 8443, 3000
   └─ Action: ALLOW

2. Rule: Allow Anchor WLC to RADIUS
   └─ Source: Anchor WLC
   └─ Destination: Primary ISE
   └─ Port: 1812, 1813
   └─ Action: ALLOW

3. Rule: Block peer-to-peer
   └─ Source/Destination: Guest VLAN
   └─ Action: DENY (no device-to-device)

4. Rule: Allow corporate devices to internet
   └─ Source: Employee VLAN (10.30.10.x)
   └─ Destination: Internet (any)
   └─ Port: 80, 443
   └─ Action: ALLOW
   └─ Note: Guest traffic routed differently (via internet gateway)
```

---

### 5.2 Primary ISE (Trusted Zone)
**Device:** Cisco Identity Services Engine (ISE 3.4)  
**Model:** ISE-3795  
**Location:** Data center, Trusted network  
**IP Address:** 10.30.30.50  
**Function:** **Core authentication authority, policy repository**

**Primary ISE Specifications:**
```
Personas Enabled:
   ✓ Administration (primary configuration)
   ✓ Policy Service (policy evaluation)
   ✓ Monitoring (log collection)

Role: Primary PAN (Policy Administration Node)
   └─ Manages all ISE configurations
   └─ Replicates to ISE-DMZ (downstream)
   └─ Maintains master policy database

Database: Corporate authentication database
   └─ Employee identities
   └─ Guest user accounts
   └─ Device identity groups
   └─ Authorization policies

Not directly used for guest authentication:
   └─ Guest portal requests go to ISE-DMZ nodes
   └─ Primary ISE validates ISE-DMZ integrity
   └─ Acts as authority for policy changes
```

---

### 5.3 Backend Authentication Services
**Services:** Active Directory / LDAP, RADIUS, Syslog  
**Location:** Trusted zone, protected  
**Function:** Provide identity and logging

#### 5.3a AD/LDAP Server
**Purpose:** Employee directory (not used for guests)  
**Specifications:**
- Windows AD or Linux LDAP
- Corporate user accounts
- Group memberships
- Guest accounts separate (stored in ISE guest DB)

#### 5.3b RADIUS Server
**Purpose:** Legacy authentication protocol support  
**Port:** 1812/1813  
**Function:** If ISE not available, RADIUS fallback

#### 5.3c Syslog Server
**Purpose:** Centralized logging  
**Specifications:**
- Collects logs from all ISE nodes
- Retention: 90 days
- Supported systems: Splunk, ELK, Syslog-ng
- Storage: All guest authentication events

---

### 5.4 Edge Router / NAT Engine
**Device:** Cisco ASR 1000 or Catalyst 8500 router  
**Location:** Data center, Trusted zone  
**Function:** **Network Address Translation (NAT) for guest traffic**

**Edge Router NAT Configuration:**
```
Outside Interface: ISP IP address (e.g., 203.0.113.50)
Inside Interface: Trusted network (10.x.x.x)

NAT Rules:
   Source: Guest VLAN (10.0.x.x)
   Translate To: ISP IP address (203.0.113.50)
   Return traffic: Translated back to guest IP

SNAT (Source NAT):
   Guest device 10.0.2.105 : 54321
   ↓ NAT translation ↓
   203.0.113.50 : 54321 (to internet)
   
SNAT Return:
   Internet response to 203.0.113.50:54321
   ↓ NAT de-translation ↓
   10.0.2.105 : 54321 (to guest device)

BGP Advertisement:
   ASN: 65000 (corporate AS)
   Advertised Routes:
      └─ 203.0.113.0/24 (public IPs)
   ISP Routes BGP back via default gateway
```

**Why NAT for Guest Traffic?**
- Guests see corporate public IP as source
- Prevents guest traffic from revealing corporate private IPs
- Enables bandwidth metering per guest
- Allows guest traffic filtering/monitoring

---

## SECTION 6: INTERNET ACCESS (Step 8-10)

### 6.1 Return Internet Traffic Path
**Flow:** Internet Server → ISP → Perimeter FW → Edge Router (NAT) → Guest Device

**Example HTTP Request/Response:**
```
GUEST INITIATING REQUEST:
   Device: 10.0.2.105 : 54321 (src)
   Server: 8.8.8.8 : 443 (dst)
   
   ↓ Through Edge Router NAT ↓
   
   ISP IP: 203.0.113.50 : 54321 (src)
   Server: 8.8.8.8 : 443 (dst)
   
   ↓ Through Perimeter Firewall ↓
   Firewall state table: 203.0.113.50:54321 ↔ 8.8.8.8:443
   
   ↓ To ISP Uplink ↓
   Packet exits data center to internet

INTERNET RESPONDING:
   Server: 8.8.8.8 : 443 (src)
   ISP IP: 203.0.113.50 : 54321 (dst)
   
   ↓ Through Perimeter Firewall ↓
   Firewall checks state table
   Entry exists: ✓ Allow
   
   ↓ Through Edge Router NAT (reverse translation) ↓
   Server: 8.8.8.8 : 443 (src)
   Guest IP: 10.0.2.105 : 54321 (dst)
   
   ↓ Routed to Guest VLAN ↓
   Device receives response packet
```

---

## COMPLETE DEVICE INVENTORY

### Remote Site (4 devices)
| # | Device | Model | IP Address | Function |
|---|--------|-------|------------|----------|
| 1 | Guest Device | Laptop/Phone | 10.0.1.x (DHCP) | End user |
| 2 | Cisco AP | Aironet 2800+ | 10.10.10.20 | WiFi transmission |
| 3 | Foreign WLC | 9800-CL | 10.10.10.50 | Guest auth server |
| 4 | SDWAN NAS | Orange Equipment | ISP IP | Encryption tunnel |

### WAN (2 devices)
| # | Device | Function | Encryption |
|---|--------|----------|-----------|
| 5 | SDWAN Tunnel | Encrypted path | IPSec/AES-256 |
| 6 | SDWAN NAS (DC) | Decryption endpoint | AES-256 decrypt |

### Data Center - Perimeter (1 device)
| # | Device | Model | Function |
|---|--------|-------|----------|
| 7 | Perimeter FW | ASA 5520 | Internet boundary |

### Data Center - DMZ (4 devices)
| # | Device | Model | IP Address | Function |
|---|--------|-------|-----------|----------|
| 8 | DMZ FW #1 | ASA 5520 | 10.20.20.5 | Guest validation |
| 9 | Load Balancer | Citrix/F5 | 10.20.20.8 | PSN distribution |
| 10 | Anchor WLC | 9800-40 | 10.20.20.10/11 | Policy enforcer |
| 11 | ISE-DMZ-01 | ISE-3795 | 10.20.20.100 | Portal + auth |
| 12 | ISE-DMZ-02 | ISE-3795 | 10.20.20.101 | Failover auth |

### Data Center - Trusted (5 devices)
| # | Device | Model | IP Address | Function |
|---|--------|-------|-----------|----------|
| 13 | Internal FW | ASA 5520 | 10.30.30.5 | Trust barrier |
| 14 | Primary ISE | ISE-3795 | 10.30.30.50 | Policy authority |
| 15 | AD/LDAP | Windows/Linux | 10.30.30.20 | Directory service |
| 16 | RADIUS Server | ISE | 10.30.30.21 | Legacy auth |
| 17 | Edge Router | ASR 1000 | 10.30.30.1 | NAT + internet |

### Internet (2 devices)
| # | Device | Function |
|---|--------|----------|
| 18 | ISP Uplink | Internet gateway |
| 19 | DNS / Web Servers | Internet resources |

---

## TRAFFIC FLOW SUMMARY TABLE

| Step | Device A | Device B | Protocol | Port | Direction | Data |
|------|----------|----------|----------|------|-----------|------|
| 1 | Guest Device | Cisco AP | 802.11 | 2.4/5 GHz | Wireless | Association request |
| 2 | Cisco AP | Foreign WLC | LWAPP | 5246/5247 | Encapsulated | Frame encapsulation |
| 3 | Foreign WLC | ISE-DMZ | RADIUS | 1812 | UDP | MAB request |
| 4 | ISE-DMZ | Foreign WLC | RADIUS | 1812 | UDP | Portal URL redirect |
| 5 | Guest Device | Load Balancer | HTTP | 80 | TCP | Portal login |
| 6 | Load Balancer | ISE-DMZ | HTTP | 80 | TCP | Forward to ISE |
| 7 | Guest Device | ISE-DMZ | HTTPS | 8443 | TCP | Secure credentials |
| 8 | ISE-DMZ | Anchor WLC | RADIUS | 1812 | UDP | Access-Accept + CoA |
| 9 | Anchor WLC | Foreign WLC | RADIUS CoA | 3799 | UDP | VLAN change signal |
| 10 | Foreign WLC | Guest Device | 802.11 | WiFi | OTA | Re-authentication (VLAN 11) |
| 11 | Guest Device | Load Balancer | HTTPS | 443 | TCP | Internet traffic |
| 12 | Load Balancer | Edge Router | HTTPS | 443 | TCP | Forwarded to internet |
| 13 | Edge Router | ISP Uplink | HTTPS (NAT) | 443 | TCP | Translated to public IP |
| 14 | ISP Uplink | Internet | HTTPS | 443 | TCP | Guest browses web |
| 15 | Internet | ISP Uplink | HTTPS | 443 | TCP | Response data |
| 16 | ISP Uplink | Edge Router | HTTPS (NAT) | 443 | TCP | Reverse NAT translation |
| 17 | Edge Router | Guest Device | HTTPS | 443 | TCP | Response delivered |

---

## FIREWALL RULES REFERENCE

### Perimeter Firewall (Outside)
```
DEFAULT: DENY ALL INBOUND
ALLOW: Response traffic only
BLOCK: All unsolicited connections
```

### DMZ Firewall #1 (Guest Validation)
```
ALLOW: SDWAN from remote site
ALLOW: RADIUS from Foreign WLC
ALLOW: HTTP/HTTPS to guest portal
BLOCK: Guest to corporate networks
BLOCK: Peer-to-peer guest traffic
```

### Internal Firewall (Trust Boundary)
```
BLOCK: All inbound from DMZ to Trusted
BLOCK: Guest access to databases
BLOCK: Guest access to file servers
ALLOW: ISE-DMZ to Primary ISE (admin only)
ALLOW: Anchor WLC to Primary ISE (RADIUS)
```

---

## AUTHENTICATION SEQUENCE (Detailed)

```
Timeline: Guest onboarding from first AP connection to internet access

T+0s: Device powers on, scans for WiFi
└─ Cisco AP beacon detected
└─ Device associates with SSID "CompanyGuest"

T+1s: 802.11 Association Complete
└─ WLC assigns to unauthenticated guest VLAN 10
└─ Guest device obtains IP 10.0.1.105 (DHCP)
└─ Can only reach portal gateway

T+2s: Device opens browser
└─ All HTTP requests intercepted
└─ Redirected to guest portal URL (HTTP 301)
└─ Portal loading from ISE-DMZ node

T+3s: Portal fully loaded
└─ User sees login form
└─ Company logo, terms of service

T+5s: User enters credentials
└─ Username: john.doe@company.com
└─ Password: ••••••••
└─ Accepts terms

T+6s: ISE-DMZ processes login
└─ Validates username/password against guest DB
└─ Runs device posture checks
└─ Generates guest access token

T+7s: ISE sends RADIUS Access-Accept
└─ Message to Anchor WLC
└─ Includes CoA (Change of Authorization)
└─ VLAN assignment: 11
└─ Session timeout: 24 hours
└─ ACL: GuestACL_Internet_Only

T+8s: Anchor WLC sends CoA to Foreign WLC
└─ Signal: Re-authenticate device AA:BB:CC:DD:EE:FF
└─ New VLAN: 11 (authorized)
└─ ACL: GuestACL_Internet_Only applied

T+9s: Foreign WLC triggers re-authentication
└─ Device receives re-auth signal
└─ Disconnects from VLAN 10
└─ Associates with VLAN 11

T+10s: Guest device in VLAN 11
└─ Obtains new IP 10.0.2.105 (DHCP)
└─ Guest ACL applied (internet only)
└─ Portal closes automatically

T+11s: User opens browser, navigates to www.google.com
└─ DNS query: google.com
└─ Routed to external DNS (8.8.8.8)
└─ Response: 142.251.33.14

T+12s: HTTP connection to google.com
└─ Source: 10.0.2.105:54321
└─ Destination: 142.251.33.14:443
└─ Traffic through Edge Router (NAT)
└─ Source NAT: 203.0.113.50:54321

T+13s: Packet exits data center
└─ ISP routes to Google servers
└─ Encrypted HTTPS connection

T+14s: Google responds
└─ Page loads in browser
└─ GUEST HAS INTERNET ACCESS

Session Active: 24 hours (until timeout or explicit logout)
Logging: All events logged to Syslog server for audit
```

---

## CONCLUSION

A guest user's journey from device connection to internet access involves **19 distinct network devices** across **3 geographic/logical zones** (remote site, WAN, data center). Every device plays a critical role in authentication, authorization, and security.

**Key Takeaways:**
- **Remote Site:** Foreign WLC authenticates guest and initiates authentication
- **WAN:** Encrypted SDWAN tunnel protects all traffic in transit
- **Data Center - Perimeter:** Multiple firewalls defend against threats
- **Data Center - DMZ:** ISE nodes validate credentials, Anchor WLC enforces policy
- **Data Center - Trusted:** Protected corporate network stays isolated from guest
- **Internet:** Guest can browse freely while corporate network remains secure

---

**Document Version:** 3.4 Complete Topology  
**Last Updated:** July 2026  
**Audience:** Network Architects, Security Teams, ISE Administrators
