# ISE Clustering Architecture: DMZ vs Trusted Nodes
## Should ISE-DMZ Nodes Be in Same Cluster as Primary ISE?

---

## QUICK ANSWER

**Most Common (Recommended for Most Organizations):**
- **YES** - DMZ and Trusted ISE nodes share the same **ISE cluster**
- **BUT** DMZ nodes are configured as separate **Node Group** within that cluster
- Single Admin Console, single policy database, unified management
- All nodes replicate policies, but DMZ nodes are demoted to Policy Service only (no admin)

**Alternative (Maximum Resilience):**
- **NO** - Separate independent ISE clusters
- DMZ cluster operates independently from Trusted cluster
- Guest auth can fail without affecting corporate auth
- Complex, requires two separate ISE deployments
- Only for critical infrastructure with extreme availability requirements

---

## OPTION 1: UNIFIED CLUSTER (RECOMMENDED)

### Architecture

```
ISE CLUSTER (Single Deployment)
┌─────────────────────────────────────────────────┐
│                ISE CLUSTER                      │
│                                                 │
│  Primary Admin Node (PAN) - Trusted Zone       │
│  ├─ Persona: Administration (Primary)          │
│  ├─ Persona: Policy Service                    │
│  ├─ Persona: Monitoring                        │
│  ├─ IP: 10.30.30.50                            │
│  └─ Role: Master, Config Authority             │
│                                                 │
│  ↓ Replicates (HTTPS 8443)                     │
│                                                 │
│  Policy Service Nodes - DMZ Zone                │
│  ├─ ISE-DMZ-01 (10.20.20.100)                  │
│  │  └─ Persona: Policy Service ONLY            │
│  │     └─ Guest portal, RADIUS, profiling      │
│  │     └─ NO Administration access             │
│  │     └─ NO policy creation ability           │
│  │                                              │
│  └─ ISE-DMZ-02 (10.20.20.101)                  │
│     └─ Persona: Policy Service ONLY            │
│        └─ Guest portal HA, RADIUS failover     │
│        └─ NO Administration access             │
│        └─ NO policy creation ability           │
│                                                 │
│  Node Groups (Load Balanced):                   │
│  ├─ Guest Services Group                       │
│  │  ├─ ISE-DMZ-01 (member)                    │
│  │  ├─ ISE-DMZ-02 (member)                    │
│  │  └─ Load Balancer VIP: 10.20.20.99         │
│  │                                              │
│  └─ Replication Group                          │
│     └─ For sync between trusted and DMZ        │
│                                                 │
└─────────────────────────────────────────────────┘

Single Database (Shared Across Cluster):
├─ Guest policies (master copy on PAN)
├─ Guest user database
├─ Corporate policies (PRIMARY ISE only)
├─ Employee identity (PRIMARY ISE only)
├─ Session logs (all nodes contribute)
└─ Audit trail (all nodes replicated)
```

### How It Works

**Scenario 1: Administrator Creates Guest Policy**
```
Admin logs into: Primary ISE GUI (10.30.30.50:443)
Action: Creates new guest authorization rule
  └─ "If guest, then allow internet only"
  └─ Set session timeout: 24 hours

Database Update:
  └─ Stored on Primary ISE database

Replication (Automatic):
  └─ Primary ISE → (HTTPS 8443) → ISE-DMZ-01
  └─ Primary ISE → (HTTPS 8443) → ISE-DMZ-02
  └─ Replication Time: <5 minutes
  └─ Direction: Trusted → DMZ (one-way push)

ISE-DMZ-01 Behavior:
  ✓ Receives updated policy
  ✓ Caches locally (read-only copy)
  ✓ Applies to new guest authentications
  ✓ Cannot modify policy (no admin access)
  ✓ Cannot push changes back to Primary ISE

Result: All nodes have consistent policies
```

**Scenario 2: Guest Authenticates**
```
Guest Device → Foreign WLC (10.10.10.50)
    └─ RADIUS request: MAC authentication

Foreign WLC → Load Balancer (10.20.20.99:1812)
    └─ RADIUS Access-Request

Load Balancer → ISE-DMZ-01 (10.20.20.100:1812)
    └─ Request routed to ISE-DMZ-01 (round-robin)

ISE-DMZ-01 Processing:
  ✓ Looks up guest in local database (cached from Primary ISE)
  ✓ Validates credentials against local copy
  ✓ Evaluates guest policies (cached from Primary ISE)
  ✓ Generates RADIUS response
  ✓ Includes VLAN assignment + ACL

ISE-DMZ-01 → Foreign WLC
    └─ RADIUS Access-Accept

Note: ISE-DMZ-01 can operate independently if Primary ISE goes down
    └─ Uses cached policies (usually fresh within 5 min)
    └─ Can serve guests for ~1-2 hours without Primary ISE
```

**Scenario 3: Replication Failure**

```
If Primary ISE goes down:
  └─ ISE-DMZ-01/02 continue serving guests
  └─ Using last cached policy snapshot
  └─ No new policies pushed until Primary ISE recovers
  
If ISE-DMZ-01 goes down:
  └─ Load Balancer detects failure (health check fails)
  └─ Automatically routes to ISE-DMZ-02
  └─ No guest authentication downtime
  └─ Primary ISE unaffected

If both ISE-DMZ nodes go down:
  └─ Load Balancer returns 503 (Service Unavailable)
  └─ Guests cannot authenticate
  └─ But guest devices stay connected (no session revocation)
  └─ Primary ISE still handles corporate auth (unaffected)
```

### Advantages of Unified Cluster

✅ **Single Management Console**
   - One Primary ISE to administer
   - One policy database to maintain
   - One audit trail across all nodes

✅ **Consistent Policies**
   - All guests see same rules
   - No policy inconsistencies
   - Easy to enforce compliance

✅ **Simple Deployment**
   - Standard Cisco ISE deployment
   - No special cluster setup
   - Easier training/handoff

✅ **Partial Resilience**
   - If DMZ nodes fail: Guests can't auth (expected)
   - If Primary ISE fails: Guests can auth from cache (~1-2 hours)
   - Corporate auth always protected (separate Trusted network)

❌ **Shared Fate Risk**
   - Database failure affects both guest and corporate
   - Cluster-wide issues impact everything
   - No independent resilience

---

## OPTION 2: SEPARATE INDEPENDENT CLUSTERS

### Architecture

```
CLUSTER 1: DMZ Guest Cluster          CLUSTER 2: Trusted Corporate Cluster
┌─────────────────────────────────┐   ┌────────────────────────────────┐
│ DMZ ISE Cluster (Independent)   │   │ Trusted ISE Cluster (Corp)     │
│                                 │   │                                │
│ Primary Admin (PAN):            │   │ Primary Admin (PAN):           │
│ ├─ ISE-DMZ-Admin               │   │ ├─ Primary ISE (10.30.30.50)  │
│ ├─ IP: 10.20.20.50             │   │ ├─ Personas: Admin + Policy    │
│ └─ Manages guest only           │   │ └─ Manages corporate only      │
│                                 │   │                                │
│ Policy Service Nodes:           │   │ Policy Service Nodes:          │
│ ├─ ISE-DMZ-01 (10.20.20.100)   │   │ ├─ PSN-Corp-01 (10.30.30.60) │
│ ├─ ISE-DMZ-02 (10.20.20.101)   │   │ └─ PSN-Corp-02 (10.30.30.61) │
│ └─ Personas: Policy Service     │   │                                │
│                                 │   │ Node Group:                    │
│ Database: Guest Only            │   │ └─ Corporate Services          │
│ ├─ Guest users                  │   │                                │
│ ├─ Guest policies               │   │ Database: Corporate Only       │
│ └─ Guest sessions               │   │ ├─ Employee users             │
│                                 │   │ ├─ Corporate policies         │
│ Admin UI: 10.20.20.50:443       │   │ ├─ VPN rules                  │
│ (DMZ Network)                   │   │ └─ 802.1X rules               │
│                                 │   │                                │
└─────────────────────────────────┘   │ Admin UI: 10.30.30.50:443     │
                                       │ (Trusted Network)             │
                                       │                                │
                                       └────────────────────────────────┘

NO REPLICATION BETWEEN CLUSTERS:
├─ Completely independent
├─ No data sync
├─ No policy sharing
└─ Manual sync if changes needed
```

### How It Works

**Scenario 1: Administrator Creates Guest Policy**
```
Admin logs into: DMZ ISE PAN GUI (10.20.20.50:443)
  └─ Separate from Corporate ISE (10.30.30.50:443)
  
Action: Creates guest authorization rule

Database Update:
  └─ Stored ONLY in DMZ cluster
  └─ Corporate cluster unaware
  └─ Replication: Within DMZ cluster only (ISE-DMZ-01/02)

Result: Policy exists in guest cluster, not corporate
```

**Scenario 2: Guest Authenticates**
```
Same as unified cluster, but:
└─ ISE-DMZ nodes are independent
└─ Can run indefinitely without Primary ISE (not shared)
└─ Corporate cluster completely unaware
```

**Scenario 3: Cluster Failures**

```
If DMZ Cluster fails (both PAN + PSN):
  └─ ALL guest authentication offline
  └─ Primary ISE (corporate) running normally ✓
  └─ Corporate authentication unaffected ✓
  └─ No shared fate with corporate

If Corporate Cluster fails:
  └─ ALL corporate authentication offline
  └─ DMZ ISE cluster running normally ✓
  └─ Guest authentication unaffected ✓
  └─ No shared fate with guest
```

### Advantages of Separate Clusters

✅ **Complete Isolation**
   - Guest failure ≠ corporate failure
   - Separate databases
   - Separate admin consoles

✅ **Maximum Resilience**
   - Each cluster can be sized independently
   - Each cluster has its own HA/backup
   - Failure is truly isolated

✅ **Independent Scaling**
   - Guest cluster can be small (2 nodes)
   - Corporate cluster can be large (4-6 nodes)
   - No over-provisioning

❌ **Operational Complexity**
   - TWO separate ISE deployments
   - TWO admin consoles to manage
   - TWO separate databases to sync
   - TWO licensing tracks

❌ **High Maintenance**
   - Configuration changes in two places
   - Policy updates duplicated
   - Twice the management overhead
   - Higher cost (double hardware)

❌ **No Shared Policy**
   - Cannot share rules between guest and corporate
   - Manual updates to both clusters
   - Risk of inconsistency

---

## COMPARISON TABLE

| Aspect | Unified Cluster | Separate Clusters |
|--------|-----------------|-------------------|
| **Same Cluster?** | YES (different personas) | NO (independent) |
| **Database Sharing** | Shared, replicated | Independent |
| **Admin Console** | 1 Primary ISE | 2 separate ISE PANs |
| **Policy Replication** | Automatic (PAN → PSN) | Manual/none |
| **If DMZ fails** | Guests down, corporate OK | Guests down, corporate OK |
| **If Corporate fails** | Corporate down, guests OK | Corporate down, guests OK |
| **If Cluster fails** | Both affected | Only that cluster affected |
| **Complexity** | Simple (standard) | Very complex |
| **Cost** | Lower (1 cluster) | Higher (2 clusters) |
| **Best for** | Most organizations | Critical/zero-downtime |
| **Cisco Recommendation** | ✓ Standard | ✗ Only if mandatory |

---

## CISCO'S OFFICIAL GUIDANCE

From Cisco ISE 3.4 documentation:

### Split Deployment (Unified Cluster - RECOMMENDED)

```
Quote: "A split deployment separates key personas across different nodes.
        The Administration and Monitoring personas can run on dedicated nodes
        in the Trusted zone, while the Policy Service persona runs on nodes
        in the DMZ or remote locations."

Implementation:
  1. Primary ISE (PAN) in Trusted zone
     └─ Personas: Admin + Policy Service + Monitoring
  
  2. ISE-DMZ nodes in DMZ
     └─ Personas: Policy Service ONLY
     └─ NO Administration persona
     └─ NO Monitoring persona
  
  3. Shared cluster database
     └─ PAN replicates to PSN nodes
     └─ All nodes part of same cluster
     └─ One management point
     └─ One policy database

Benefits:
  ✓ Separates admin/policy management from guest portal
  ✓ Better load distribution
  ✓ Simple management (single cluster)
  ✓ PSN can be offline or failed without affecting PAN
  ✓ Industry standard architecture
```

### Distributed Deployment (Separate Clusters - NOT COMMON)

```
Quote: "For deployments requiring complete separation of guest and
        corporate authentication, you may deploy independent ISE clusters."

But: "This is not recommended due to operational complexity"

Only Consider If:
  • Regulatory requirement for complete isolation
  • Guest and corporate must survive independently
  • Separate funding/teams managing each
  • Complex multi-tenant environment
```

---

## RECOMMENDATION FOR YOUR ARCHITECTURE

### Best Option: **Unified Cluster with Node Groups**

```
PRIMARY ISE (Trusted Zone)
├─ IP: 10.30.30.50
├─ Personas: Administration (primary), Policy Service, Monitoring
├─ Role: Master/Authority
└─ Database: Master copy

├── Node Group: Guest Services (DMZ)
│   ├─ ISE-DMZ-01 (10.20.20.100)
│   │  └─ Persona: Policy Service
│   │     └─ Guests ONLY, no corporate access
│   ├─ ISE-DMZ-02 (10.20.20.101)
│   │  └─ Persona: Policy Service
│   │     └─ Guests ONLY, no corporate access
│   └─ Load Balancer VIP: 10.20.20.99
│
└── Node Group: Corporate Services (Trusted)
    ├─ PSN-Corp-01 (10.30.30.60)
    └─ PSN-Corp-02 (10.30.30.61)
       └─ Personas: Policy Service
          └─ Corporate ONLY, no guest access
```

### Why This is Optimal

✅ **Single Cluster** = Easy management
✅ **Node Groups** = Separate workloads (guest vs corporate)
✅ **Persona Separation** = DMZ nodes have no admin
✅ **Policy Isolation** = Corporate policies in Trusted zone
✅ **Resilience** = Either group can fail independently
✅ **Standard** = Matches Cisco best practices
✅ **Cost-Effective** = One license per node, one cluster

---

## NETWORK CONNECTIVITY FOR UNIFIED CLUSTER

### Replication Between Zones

```
Primary ISE (10.30.30.50) → DMZ Nodes
  Protocol: HTTPS (TLS)
  Port: 8443 (admin port)
  Direction: One-way (Trusted → DMZ)
  
  Data Replicated:
  ├─ Guest policies (read-only copy to DMZ)
  ├─ Guest user database (snapshot)
  ├─ System configuration (applicable to both)
  ├─ Certificates
  └─ Policy updates (every 5 minutes auto-sync)

Authentication for Replication:
  ├─ Certificate-based (built-in ISE certificates)
  ├─ Encrypted tunnel
  └─ Automatic trust within cluster

Return Path (DMZ → Trusted):
  ├─ Session logs (syslog 514)
  ├─ Replication acknowledgment
  └─ Status heartbeat (every 30 seconds)
```

### Firewall Rules Needed

```
DMZ Firewall #1 (Guest Zone):
  ALLOW: ISE-DMZ nodes to reach load balancer
         └─ Source: ISE-DMZ-01/02
         └─ Destination: LB VIP 10.20.20.99
         └─ Ports: 80, 8443, 1812, 1813
         └─ Action: ALLOW

Trust Boundary Firewall (Internal):
  ALLOW: ISE-DMZ to Primary ISE (ADMIN SYNC ONLY)
         └─ Source: ISE-DMZ-01/02 (10.20.20.100-101)
         └─ Destination: Primary ISE (10.30.30.50)
         └─ Port: 8443 (HTTPS only)
         └─ Direction: Outbound from DMZ
         └─ Note: DMZ initiates, Trusted responds
         
  ALLOW: ISE-DMZ to Syslog (LOGGING)
         └─ Source: ISE-DMZ-01/02
         └─ Destination: Syslog Server (10.30.30.30)
         └─ Port: 514 (UDP)
         └─ Action: ALLOW
         
  BLOCK: All other DMZ-to-Trusted
```

---

## SUMMARY

**For your architecture:**

```
ISE-DMZ-01 and ISE-DMZ-02:
  ├─ YES, they belong to the SAME ISE CLUSTER as Primary ISE
  ├─ But with different personas (PSN vs PAN)
  ├─ They are in the SAME NODE GROUP (Guest Services)
  ├─ Database replicates from Primary ISE
  ├─ No admin access (security isolation)
  └─ Controlled firewall access (port 8443 for sync only)

Primary ISE (10.30.30.50):
  ├─ Same cluster (PAN role)
  ├─ Master copy of all policies
  ├─ Replicates guest policies to DMZ nodes
  ├─ DMZ nodes cannot modify policies
  └─ Completely protected in Trusted zone

Result: Single unified ISE deployment with geographic/logical separation
        Standard Cisco architecture
        Recommended for 99% of enterprises
```

---

**Document Version:** ISE Clustering Design 3.4  
**Recommendation:** Unified Cluster with Node Groups  
**Complexity:** Low (standard Cisco deployment)  
**Cost:** Minimal (no additional ISE licenses needed)
