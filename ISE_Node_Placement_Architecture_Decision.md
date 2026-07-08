# ISE Node Placement Architecture
## Why Put ISE Guest Nodes in DMZ vs Trusted Zone with Open Firewall

---

## QUICK ANSWER

**Best Practice (Cisco Recommended):** **Split Deployment**
- **DMZ:** ISE-DMZ-01 and ISE-DMZ-02 (Policy Service Nodes for guest portal + auth)
- **Trusted:** Primary ISE (Policy Admin Node for configuration + backend auth)

**Why NOT open firewall holes to Trusted ISE:**
- ❌ Requires guest traffic to cross trust boundary
- ❌ Creates firewall rule exceptions (security gaps)
- ❌ If compromised, attacker has direct path to core policies
- ❌ Harder to isolate/revoke guest access in emergency
- ❌ Mixing guest and corporate policy logic

---

## ARCHITECTURE OPTION 1: DMZ SPLIT DEPLOYMENT (RECOMMENDED)

```
REMOTE SITE                    DATA CENTER
┌──────────────┐              ┌─────────────────────────────────┐
│ Foreign WLC  │─SDWAN────→   │  ┌──────────────────────────┐   │
│              │  Tunnel      │  │     PERIMETER FW         │   │
│ IP: 10.10..  │              │  └──────────────────────────┘   │
└──────────────┘              │           ↓                      │
                              │  ┌──────────────────────────┐   │
                              │  │   DMZ Firewall #1        │   │
                              │  └──────────────────────────┘   │
                              │           ↓                      │
                    ┌─────────┴─────────────────────────────┐   │
                    │          D M Z   Z O N E              │   │
                    │  ┌──────────────────────────────────┐ │   │
                    │  │  ISE-DMZ-01 (Policy Service)     │ │   │
                    │  │  IP: 10.20.20.100                │ │   │
                    │  │  Persona: Guest Portal + Auth    │ │   │
                    │  │  Ports: 80, 8443, 1812, 1813     │ │   │
                    │  └──────────────────────────────────┘ │   │
                    │  ┌──────────────────────────────────┐ │   │
                    │  │  ISE-DMZ-02 (Policy Service HA)  │ │   │
                    │  │  IP: 10.20.20.101                │ │   │
                    │  │  Persona: Failover Portal + Auth │ │   │
                    │  │  Ports: 80, 8443, 1812, 1813     │ │   │
                    │  └──────────────────────────────────┘ │   │
                    │                                        │   │
                    │  Guest Portal Functions:               │   │
                    │  ✓ Hosts guest login page              │   │
                    │  ✓ Validates guest credentials         │   │
                    │  ✓ Checks device posture               │   │
                    │  ✓ Sends RADIUS CoA to WLC            │   │
                    │  ✓ Runs posture compliance checks      │   │
                    │  ✓ No access to corporate policies     │   │
                    │                                        │   │
                    └────────────────────────────────────────┘   │
                              ↓                                   │
                    ┌─────────────────────────────────────┐       │
                    │   TRUST BOUNDARY FIREWALL           │       │
                    │                                     │       │
                    │   Rules:                            │       │
                    │   • Block: All guest inbound        │       │
                    │   • Allow: ISE-DMZ to Primary ISE  │       │
                    │     (admin sync only: port 8443)   │       │
                    │   • Allow: RADIUS to Primary ISE    │       │
                    │     (replication: port 1812/1813)  │       │
                    │                                     │       │
                    └─────────────────────────────────────┘       │
                              ↓                                   │
                    ┌─────────────────────────────────────┐       │
                    │  T R U S T E D   Z O N E            │       │
                    │  ┌────────────────────────────────┐ │       │
                    │  │  Primary ISE (PAN)             │ │       │
                    │  │  IP: 10.30.30.50               │ │       │
                    │  │  Personas:                      │ │       │
                    │  │  • Administration (primary)     │ │       │
                    │  │  • Policy Service (backup)      │ │       │
                    │  │  • Monitoring                   │ │       │
                    │  │                                 │ │       │
                    │  │  Functions:                     │ │       │
                    │  │  ✓ Master policy authority      │ │       │
                    │  │  ✓ Configuration management     │ │       │
                    │  │  ✓ Backend auth (AD/LDAP)       │ │       │
                    │  │  ✓ Employee identity validation │ │       │
                    │  │  ✓ Session logging              │ │       │
                    │  │  ✓ Corporate policy enforcement │ │       │
                    │  │  ✓ Replicates to ISE-DMZ nodes  │ │       │
                    │  │                                 │ │       │
                    │  │  Admin Access: Management Only   │ │       │
                    │  │  (No guest traffic ever reaches) │ │       │
                    │  └────────────────────────────────┘ │       │
                    └─────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘

TRAFFIC PATHS:

1. Guest Authentication (DMZ Only)
   ├─ Foreign WLC → (SDWAN tunnel) → ISE-DMZ-01
   ├─ ISE-DMZ-01 validates guest credentials
   ├─ ISE-DMZ-01 → (RADIUS CoA) → Anchor WLC
   └─ Guest gets internet access (never crosses trust boundary)

2. ISE Administration (Trusted Only)
   ├─ Admin → Primary ISE (10.30.30.50)
   ├─ Creates guest policies
   ├─ Updates authorization rules
   └─ Changes replicate down to ISE-DMZ-01/02 (unidirectional)

3. ISE Sync (Across Trust Boundary - CONTROLLED)
   ├─ Primary ISE → (HTTPS 8443 secure tunnel) → ISE-DMZ-01/02
   ├─ Replicates: Guest policies, user database snapshots
   ├─ Direction: Trusted → DMZ (one-way push)
   └─ Firewall allows only admin/replication (not guest traffic)
```

**Why This Works:**

✅ **Guest traffic never crosses trust boundary** - Reduces attack surface
✅ **DMZ ISE nodes are "read-only" for guest data** - Limited damage if compromised
✅ **ISE-DMZ-01/02 cannot modify corporate policies** - No admin access
✅ **Clear separation of duties** - Guest portal separate from policy authority
✅ **Firewall rules minimal** - Only allow ISE-DMZ to communicate with Primary ISE for sync
✅ **Easy to revoke guest access** - Shut down DMZ firewall #1, guests cut off immediately
✅ **RADIUS can be offline** - ISE-DMZ caches guest policies, continues serving for ~1 hour

---

## ARCHITECTURE OPTION 2: TRUSTED ISE WITH FIREWALL HOLES (NOT RECOMMENDED)

```
REMOTE SITE                    DATA CENTER
┌──────────────┐              ┌─────────────────────────────────┐
│ Foreign WLC  │─SDWAN────→   │  ┌──────────────────────────┐   │
│              │  Tunnel      │  │     PERIMETER FW         │   │
│ IP: 10.10..  │              │  └──────────────────────────┘   │
└──────────────┘              │           ↓                      │
                              │  ┌──────────────────────────┐   │
                              │  │   DMZ Firewall #1        │   │
                              │  │                          │   │
                              │  │  FIREWALL HOLES REQUIRED │   │
                              │  │  • HTTPS 8443 to ISE     │   │
                              │  │  • HTTP 80 to ISE portal │   │
                              │  │  • RADIUS 1812/1813      │   │
                              │  │                          │   │
                              │  └──────────────────────────┘   │
                              │           ↓                      │
                    ┌─────────────────────────────────────┐     │
                    │     D M Z   Z O N E                 │     │
                    │  (Empty - no ISE nodes)             │     │
                    │                                     │     │
                    │  BUT: Must allow guest traffic      │     │
                    │  to reach Trusted zone!             │     │
                    │  (Security gap)                     │     │
                    └──────────────┬──────────────────────┘     │
                                   │                             │
                    ┌──────────────────────────────────────┐    │
                    │   TRUST BOUNDARY FIREWALL            │    │
                    │                                      │    │
                    │   SECURITY PROBLEM:                  │    │
                    │   • Allow: Guest HTTPS 443 to ISE    │    │
                    │   • Allow: Guest HTTP 80 to ISE      │    │
                    │   • Allow: Guest RADIUS 1812 to ISE  │    │
                    │                                      │    │
                    │   Result: Guest traffic crosses      │    │
                    │   trust boundary 3 times!            │    │
                    │                                      │    │
                    │   If guest network compromised:      │    │
                    │   Attacker can attack ISE directly   │    │
                    │                                      │    │
                    └──────────────┬───────────────────────┘    │
                                   │                             │
                    ┌──────────────────────────────────────┐    │
                    │  T R U S T E D   Z O N E             │    │
                    │  ┌────────────────────────────────┐  │    │
                    │  │  Primary ISE (PAN/PSN combined)│  │    │
                    │  │  IP: 10.30.30.50               │  │    │
                    │  │                                 │  │    │
                    │  │  Personas:                      │  │    │
                    │  │  • Administration (primary)     │  │    │
                    │  │  • Policy Service (guest)       │  │    │
                    │  │  • Monitoring                   │  │    │
                    │  │                                 │  │    │
                    │  │  Mixed Functions:               │  │    │
                    │  │  ✓ Guest portal (exposed)       │  │    │
                    │  │  ✓ Corporate auth (protected)   │  │    │
                    │  │  ✓ Configuration management     │  │    │
                    │  │  ✓ Policy enforcement           │  │    │
                    │  │                                 │  │    │
                    │  │  RISK: If ISE compromised,      │  │    │
                    │  │  attacker has access to:        │  │    │
                    │  │  • Employee credentials         │  │    │
                    │  │  • Corporate policies           │  │    │
                    │  │  • Authorization rules          │  │    │
                    │  │  • Backend AD/LDAP             │  │    │
                    │  └────────────────────────────────┘  │    │
                    └──────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

TRAFFIC PATHS (With Security Issues):

1. Guest Authentication (Crosses Trust Boundary - BAD)
   ├─ Foreign WLC → (SDWAN) → Guest VLAN (10.0.x.x)
   ├─ Guest device opens browser
   ├─ HTTP request hits DMZ firewall
   ├─ DMZ FW rule allows: TCP 80 to 10.30.30.50 (ISE)
   ├─ Guest traffic CROSSES TRUST BOUNDARY ❌
   ├─ Reaches ISE portal in Trusted zone
   ├─ Guest logs in
   ├─ ISE issues RADIUS response
   ├─ Response CROSSES TRUST BOUNDARY again ❌
   └─ Guest gets access

2. RADIUS Authentication
   ├─ Anchor WLC sends RADIUS to ISE (10.30.30.50:1812)
   ├─ Goes through Trust Boundary Firewall ❌
   ├─ ISE responds with RADIUS Access-Accept
   └─ Response comes back

3. Posture Compliance Checks
   ├─ Guest device sends health check to ISE (10.30.30.50:8443)
   ├─ Goes through Trust Boundary Firewall ❌
   └─ ISE validates device status

TOTAL SECURITY ISSUES:

❌ Guest traffic crosses trust boundary 4+ times
❌ Attacker on guest network can attack firewall holes directly
❌ If ISE compromised, attacker gets corporate database + policies
❌ Hard to revoke guest access (must shut down ISE or close firewall)
❌ All guest sessions logged on primary ISE with corporate data
❌ No isolation between guest functions and admin functions
❌ Firewall rules are complex and hard to audit
```

**Problems with This Approach:**

❌ **Security gaps** - Multiple firewall holes = multiple attack vectors
❌ **Mixed trust zones** - Guest traffic in trusted ISE database
❌ **Single point of failure** - One ISE for both guest and corporate
❌ **Compliance risk** - Guest activity logged on primary ISE (audit nightmare)
❌ **Attack escalation** - If guest network compromised, direct path to corporate policies
❌ **Harder to revoke** - Revoking guest access requires more than just closing firewall
❌ **Lateral movement** - Compromised guest ISE can pivot to trusted services

---

## DETAILED COMPARISON TABLE

| Aspect | DMZ Deployment (Recommended) | Trusted with Firewall Holes |
|--------|------------------------------|---------------------------|
| **ISE-DMZ Node Location** | DMZ | N/A (not deployed) |
| **ISE Primary Location** | Trusted | Trusted |
| **Guest Portal Location** | DMZ (direct access) | Trusted (behind FW hole) |
| **Trust Boundary Crossings** | 0 (guest stays in DMZ) | 4+ (auth, portal, posture) |
| **Firewall Holes Needed** | Only ISE-DMZ to Primary ISE | Guest to ISE (3 ports) |
| **Compromise Impact** | Limited to guest functions | Access to entire ISE system |
| **Revoke Guest Access** | Close DMZ FW #1 (instant) | Complex, must edit rules |
| **Compliance Audit** | Guest logs separate from corporate | Guest logs mixed with corporate |
| **Admin Complexity** | Higher (two ISE nodes) | Lower (one ISE) |
| **Security Posture** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐ Poor |
| **Best for** | Enterprise security-conscious | Small businesses, cost-cutting |

---

## WHY CISCO RECOMMENDS DMZ DEPLOYMENT

### 1. Defense in Depth
```
DMZ Deployment:
Guest Network → DMZ Firewall → ISE-DMZ → Trust Boundary FW → Primary ISE

Multiple layers between guest and corporate:
  Layer 1: DMZ Firewall #1 (guest validation)
  Layer 2: ISE-DMZ nodes (limited functions)
  Layer 3: Trust Boundary Firewall (absolute barrier)
  Layer 4: Primary ISE (core policies)
```

### 2. Least Privilege Access
```
DMZ ISE Nodes Can Do:
✓ Host guest portal
✓ Validate guest credentials
✓ Send RADIUS responses
✓ Run device posture checks
✗ Create new policies
✗ Modify authorization rules
✗ Access employee data
✗ Change system configuration
```

### 3. Isolation of Duties
```
DMZ ISE-01/02:
- Handles GUEST traffic only
- No access to corporate policies
- Stateless posture checks
- Limited session data

Primary ISE:
- Handles CORPORATE traffic only
- Master policy authority
- Employee identity validation
- Full audit logging
```

### 4. Rapid Revocation
```
Emergency Guest Access Shutdown:

DMZ Model:
  1. Close DMZ Firewall #1 port 80/443
  2. All guest portals offline (instant)
  3. No ISE-DMZ nodes needed
  4. Corporate network unaffected
  ⏱ Time: 2 minutes

Trusted Model:
  1. Edit ISE firewall rules
  2. Restart ISE (or selectively disable)
  3. Affects corporate auth too
  4. Risky to both guest and corporate
  ⏱ Time: 15+ minutes + risk
```

---

## THE ISE NODES SPECIFICALLY

### ISE-DMZ-01 (Guest Delivery Node)
**Physical Location:** Data Center DMZ  
**IP Address:** 10.20.20.100  
**Hardware:** ISE-3795 or similar  
**Personas Enabled:**
```
✓ Policy Service
  └─ Guest portal (port 80, 8443)
  └─ Guest authentication (RADIUS: 1812, 1813)
  └─ Device profiling (guest devices only)
  └─ Posture compliance checks
✗ NOT Administration (cannot change config)
✗ NOT Monitoring (no logging responsibility)
```

**What ISE-DMZ-01 Hosts:**
```
1. Guest Portal Files
   - Login page HTML/CSS
   - Credential capture form
   - BYOD enrollment flow
   - Portal customization

2. Guest Database (Read-Only Copy)
   - Sync from Primary ISE every 5 minutes
   - Contains: Guest users, credentials, session state
   - Does NOT contain: Employee data, corporate policies

3. Guest Authorization Rules
   - "If guest, then allow internet only"
   - "If guest + posture fails, deny"
   - "Guest session timeout: 24 hours"
   - Does NOT contain: Corporate policies, VPN rules

4. Guest Posture Engine
   - Checks: Windows Update status, antivirus
   - Applies to: Guest devices only
   - Does NOT check: Corporate compliance

5. RADIUS Server (Guest Auth)
   - Listens on port 1812/1813
   - Processes requests from Foreign WLC
   - Sends responses with guest VLAN + ACL
   - Does NOT process: Corporate authentication
```

**Network Connections (ISE-DMZ-01):**
```
INBOUND:
  • From Guest Device: HTTP/HTTPS (port 80, 8443)
    └─ Guest opens browser, logs in to portal
  • From Foreign WLC: RADIUS (port 1812)
    └─ WLC sends authentication requests
  • From Load Balancer: Health checks (port 8443)
    └─ LB verifies ISE-DMZ-01 is alive

OUTBOUND:
  • To Anchor WLC: RADIUS (port 1812)
    └─ Sends RADIUS responses
  • To Primary ISE: HTTPS (port 8443)
    └─ Syncs guest user data, policies
  • To Syslog Server: Syslog (port 514)
    └─ Sends guest session logs

BLOCKED:
  • No connection to AD/LDAP (in Trusted zone)
  • No connection to employee databases
  • No connection to corporate VPN gateways
  • No connection to file servers
```

### ISE-DMZ-02 (Guest Delivery Node - HA)
**Physical Location:** Data Center DMZ  
**IP Address:** 10.20.20.101  
**Hardware:** ISE-3795 or similar  
**Identical to ISE-DMZ-01** but as failover

**Load Balancer Configuration:**
```
VIP: 10.20.20.99 (virtual IP for guests)

Members:
  Primary: ISE-DMZ-01 (10.20.20.100)
  Secondary: ISE-DMZ-02 (10.20.20.101)

Health Checks:
  - Every 5 seconds
  - Port 8443 (HTTPS)
  - If ISE-DMZ-01 fails: Route to ISE-DMZ-02
  - If both fail: Return 503 (Service Unavailable)

Session Stickiness:
  - MAC address (Calling Station ID)
  - Ensures same device always hits same ISE-DMZ
```

### Primary ISE (Policy Admin Node)
**Physical Location:** Data Center Trusted Zone  
**IP Address:** 10.30.30.50  
**Hardware:** ISE-3795 or similar  

**Personas Enabled:**
```
✓ Administration (PRIMARY)
  └─ Configuration management
  └─ Policy creation
  └─ User management
✓ Policy Service (SECONDARY - backup only)
  └─ Handles corporate auth if needed
  └─ Employee identity validation
✓ Monitoring (PRIMARY)
  └─ Log collection
  └─ Session analytics
  └─ Audit trail
```

**What Primary ISE Manages:**
```
1. Guest Policies (Master Copy)
   ├─ Replicates DOWN to ISE-DMZ-01/02
   ├─ Administrators edit here
   ├─ Changes sync within 5 minutes
   └─ Includes: Guest types, portal settings, timeouts

2. Corporate Policies (Protected)
   ├─ Employee authentication rules
   ├─ VPN access policies
   ├─ Device compliance rules
   ├─ 802.1X authorization
   └─ NOT replicated to DMZ

3. Employee Identity Store
   ├─ Connection to AD/LDAP
   ├─ Employee database
   ├─ Group memberships
   └─ NOT accessible from DMZ

4. Backend Authentication Services
   ├─ RADIUS for legacy devices
   ├─ TACACS+ for network devices
   ├─ Kerberos integration
   └─ All protected in Trusted zone

5. Comprehensive Logging
   ├─ All guest sessions
   ├─ Corporate authentication events
   ├─ Policy violations
   ├─ Admin actions
   └─ 90-day retention (Syslog Server)
```

**Network Connections (Primary ISE):**
```
INBOUND:
  • From ISE-DMZ-01/02: HTTPS (port 8443, 3000)
    └─ Guest policy sync requests
  • From Admin: HTTPS (port 443)
    └─ Web UI administration
  • From Anchor WLC: RADIUS (port 1812)
    └─ Corporate authentication
  • From Syslog Server: Syslog (port 514)
    └─ Logs from all nodes

OUTBOUND:
  • To ISE-DMZ-01/02: HTTPS (port 8443)
    └─ Push policy updates
  • To AD/LDAP: LDAP (port 389/636)
    └─ Employee lookup
  • To RADIUS Clients: RADIUS (port 1812)
    └─ Authentication responses
  • To Syslog Server: Syslog (port 514)
    └─ Send logs
```

---

## FIREWALL RULES FOR SPLIT DEPLOYMENT

### DMZ Firewall #1 (Guest Zone)

```
RULE 1: Allow guest portal access
  Source: Guest VLAN (10.0.x.x)
  Destination: Load Balancer (10.20.20.99)
  Port: 80 (HTTP), 8443 (HTTPS)
  Action: ALLOW
  Log: All connections

RULE 2: Allow RADIUS from Foreign WLC
  Source: Foreign WLC (10.10.10.50)
  Destination: Load Balancer (10.20.20.99)
  Port: 1812, 1813 (RADIUS)
  Action: ALLOW
  Log: All connections

RULE 3: Block guest to corporate
  Source: Guest VLAN (10.0.x.x)
  Destination: Trusted zone (10.30.x.x)
  Port: All
  Action: DENY
  Log: CRITICAL - policy violation

RULE 4: Block guest peer-to-peer
  Source: Guest VLAN
  Destination: Guest VLAN (internal)
  Port: All
  Action: DENY
  Log: CRITICAL - lateral movement attempt

DEFAULT: Deny all other traffic
```

### Trust Boundary Firewall (Internal)

```
RULE 1: Block all inbound from DMZ
  Source: DMZ (10.20.x.x)
  Destination: Trusted zone (10.30.x.x)
  Action: DENY
  Exception: ISE-DMZ to Primary ISE ONLY
  Log: All denied connections

RULE 2: Allow ISE-DMZ policy sync
  Source: ISE-DMZ-01/02 (10.20.20.100-101)
  Destination: Primary ISE (10.30.30.50)
  Port: 8443 (HTTPS), 3000 (pxGrid)
  Action: ALLOW
  Note: One-way push only (DMZ cannot pull corporate policies)

RULE 3: Allow Anchor WLC to Primary ISE
  Source: Anchor WLC (10.20.20.10)
  Destination: Primary ISE (10.30.30.50)
  Port: 1812, 1813 (RADIUS)
  Action: ALLOW
  Note: Corporate authentication requests only

RULE 4: Block guest database lookups
  Source: ISE-DMZ-01/02
  Destination: AD/LDAP (10.30.30.20)
  Action: DENY
  Note: Guest ISE cannot query employee database

DEFAULT: Deny all inbound from DMZ
```

---

## SCENARIO: WHAT IF GUEST NETWORK IS COMPROMISED?

### With DMZ Deployment (Safe)
```
Attacker on Guest Network:
  1. Compromises guest device
  2. Tries to attack ISE-DMZ-01 (10.20.20.100)
     └─ Attacks guest portal (low value, limited access)
     └─ Gets guest database (only guest users)
  3. Tries to reach Primary ISE (10.30.30.50)
     └─ DMZ Firewall blocks all inbound (DENIED)
     └─ Cannot reach Primary ISE
  4. Tries to reach corporate resources
     └─ DMZ Firewall blocks all corporate traffic (DENIED)
  5. Damage: Limited to guest portal compromise
     └─ Can revoke guest accounts
     └─ Bounce ISE-DMZ-01, restore from backup
     └─ Primary ISE and corporate network untouched

Recovery Time: 30 minutes to restore guest portal
```

### With Trusted Deployment (Dangerous)
```
Attacker on Guest Network:
  1. Compromises guest device
  2. Attacks firewall holes (port 80, 443, 1812)
     └─ Exploits ISE vulnerability
  3. Gets shell access on Primary ISE (10.30.30.50)
  4. Now has access to:
     ├─ Employee credentials (from AD)
     ├─ Corporate policies
     ├─ VPN configuration
     ├─ Device compliance rules
     ├─ Authorization rules
     ├─ All session logs
     └─ Potential lateral movement to network
  5. Damage: COMPLETE compromise of ISE and policies
     └─ Attacker can create backdoor admin account
     └─ Attacker can modify authorization rules
     └─ Attacker can log all corporate connections
     └─ Corporate network at risk

Recovery Time: 4+ hours (full ISE rebuild, policy reset, credential change)
Recovery Cost: Incident response, forensics, breach notification
```

---

## BEST PRACTICE RECOMMENDATION

**Deploy ISE in Split Topology:**

```
Guest Portal Layer:        ISE-DMZ-01 (10.20.20.100)
                          ISE-DMZ-02 (10.20.20.101)
                          Load Balancer (10.20.20.99)
                          Location: DMZ
                          Firewall: DMZ FW #1

                          ↓ (Policy sync via HTTPS 8443)

Policy Authority Layer:    Primary ISE (10.30.30.50)
                          Location: Trusted Zone
                          Firewall: Trust Boundary FW

                          ↓ (Corporate auth)

Backend Services:          AD/LDAP (10.30.30.20)
                          RADIUS (10.30.30.21)
                          Syslog (10.30.30.30)
```

**This Design:**
✅ Isolates guest from corporate
✅ Limits damage if guest portal compromised
✅ Allows rapid revocation of guest access
✅ Protects employee identity data
✅ Maintains audit separation
✅ Meets security compliance requirements
✅ Follows Cisco prescriptive guidance

---

**Document Version:** ISE Architecture Decision  
**Recommendation:** Use Split Deployment (DMZ + Trusted)  
**Security Rating:** ⭐⭐⭐⭐⭐ (Excellent)
