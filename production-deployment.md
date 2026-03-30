# Production-Level Deployment & PostgreSQL Core Integration

## Overview
This guide outlines transitioning our Linux-based TDE implementation to production and integrating it into PostgreSQL core.

---

## Part 1: Production Architecture

### Infrastructure Stack
```
Physical Storage (SAN/RAID-10) → LVM → dm-crypt LUKS 
→ Filesystem (ext4/XFS) → PostgreSQL Data
```

**Key Improvements Over Development:**
- RAID-10 for redundancy and performance
- LVM for dynamic resizing and snapshots
- XFS for large datasets
- Systemd mount units for robust automation
- HSM integration for key storage

---

## Part 2: High Availability

### Replication Architecture
- Streaming replication over SSL
- Same encryption key for both primary and replica
- Separate keys for sensitive tablespaces (defense in depth)
- Automated failover via Patroni with DCS (Etcd/Consul)

### Failover Flow
```
PRIMARY ─SSL Replication→ REPLICA
           ↓
    Patroni Monitors Both
           ↓
   Primary Fails → Promote Replica
           ↓
  Applications Reconnect (via VIP)
```

---

## Part 3: PostgreSQL Core Integration

### Proposed PostgreSQL Patches

#### 1. TDE Configuration Module (`src/backend/utils/misc/tde.c`)
```
postgresql.conf additions:
- tde.enabled = on
- tde.key_manager = 'vault'|'file'|'hsm'
- tde.vault_address = 'http://vault:8200'
- tde.encrypted_data_path = '/mnt/pg_encrypted'
```

#### 2. Key Management Command (`pg_encrypt_wallet`)
```
pg_encrypt_wallet --init              # Initialize TDE
pg_encrypt_wallet --show-keys         # List keys
pg_encrypt_wallet --add-key <key>     # Add new key
pg_encrypt_wallet --rotate-keys       # Rotate keys
pg_encrypt_wallet --verify            # Verify integrity
```

#### 3. System Catalog Extensions
```sql
SELECT name, encryption_enabled, key_id FROM pg_encrypted_tablespaces;
SELECT key_id, algorithm, rotation_date FROM pg_encryption_keys;
SELECT operation, key_id, principal, timestamp FROM pg_encryption_audit;
```

#### 4. Monitoring Functions
```sql
SELECT pg_tde_status();              -- Current TDE state
SELECT pg_tde_verify_volumes();      -- Check mount status
SELECT pg_tde_key_rotation_due();    -- Check rotation needs
SELECT pg_tde_performance_stats();   -- Encryption overhead
```

### Implementation Timeline
- **Months 1-2**: Code quality, community engagement
- **Months 3-6**: Feature development and testing
- **Months 7-12**: Upstream contribution to PostgreSQL
- **Year 2+**: Integration into PostgreSQL 19+

### Alternative: Extension Approach
If core integration delayed, deploy as `pg_tde_enterprise` extension for faster (2-3 month) deployment.

---

## Part 4: Operations & Deployment

### Deployment Checklist
```
Pre-Deployment: Security audit, key infrastructure, network encryption
Deployment: Encrypted volumes, LVM/RAID, LUKS setup, PostgreSQL init
Post-Deployment: Verify encryption, test failover, validate backups
```

### Monitoring
```
Metrics:
- Volume mount status, key age, access frequency
- Encryption overhead %, CPU usage, latency impact
- Failed key access attempts, certificate expiration

Tools: Prometheus + Grafana, Vault audit logs, syslog
```

### Backup Strategy
```
Tier 1: WAL archiving (10-second RPO)
Tier 2: Daily encrypted base backups (24-hour RPO)
Tier 3: Monthly off-site backups (AWS Glacier, 7-year retention)

All backups encrypted at storage + application level
```

### Disaster Recovery
```
Replica Failure: 10-20 min (Patroni auto-failover)
Primary Failure: 1-2 min (Promote replica, VIP switch)
Data Center Outage: 30 min - 2 hours (Recover from backup + keys)
```

---

## Part 5: Security & Compliance

### Encryption Standards
- **Algorithm**: AES-256-XTS (NSA approved for Top Secret data)
- **Key Size**: 256-bit master key (32 bytes)
- **Backup Keys**: Separate 256-bit keys
- **TLS**: 2048-bit RSA or 256-bit ECDP, renewed annually

### Access Control
```
Roles:
- TDE Admin: Manage encryption + key rotation
- DBA: Manage database objects (no key access)
- App Service: Connect via SSL, no encryption visibility
- Auditor: Read-only to audit logs

Vault policies: IP-restricted, time-limited access
```

### Compliance
| Framework | Implementation |
|-----------|---|
| GDPR | Encryption + audit logging + data retention policy |
| HIPAA | AES-256 + TLS + HSM storage + 6-year audit retention |
| PCI-DSS | Encryption + key management + change control + testing |

---

## Part 6: Cost-Benefit Analysis

### Annual TCO (100GB deployment)
```
Hardware:           $50,000
Software (Free):    $0 (PostgreSQL, Vault community)
Operations (2 DBA): $460,000
──────────────────
Total:              $510,000

vs Oracle TDE: $600,000+ (plus higher complexity)
```

### Risk Mitigation Value
- **Data breach via disk theft**: Prevented, data remains encrypted
- **Backup exposure**: Eliminated through dual-layer encryption
- **Compliance violation**: Avoids €10M+ GDPR fines
- **ROI**: Breaks even with single major incident prevented

---

## Part 7: Migration Path

### Environment Progression
1. **Development**: WSL2, virtual disks, learning
2. **Staging**: Mirrors production, full testing
3. **Production**: Hardware-backed, HA, HSM keys

### Three Migration Strategies

**Option 1: Logical Migration** (< 1TB, 2-8 hours)
- pg_dump from prod → import to encrypted instance
- Verify checksums, confirm replication current

**Option 2: Physical Migration** (> 1TB, 4-24 hours)
- Block-level copy of data directory
- Configured as new PostgreSQL instance

**Option 3: Dual-Write Migration** (Zero downtime)
- Keep both systems running, replay backlog
- Switch reads first, then writes, then retire old system

### Rollback Procedures
- **Abort before switch**: No risk, redeploy
- **Abort after switch (< 24h)**: Switch VIP back to old system
- **Abort post-cutover**: Restore from archive (2-4 hours)

---

## Part 8: Enterprise Integration

### Active Directory/LDAP
PostgreSQL ← LDAP Auth ← AD → Client Certificates → SSL/TLS

### SIEM Integration
PostgreSQL logs → Filebeat → ELK Stack → Kibana dashboards → Alerts

### Change Management
ServiceNow → Terraform → Test in Staging → Deploy → Verify → Close

---

## Part 9: 4-Year Roadmap

| Year | Q | Milestone |
|------|---|-----------|
| 1 | Q1-Q2 | Post proposal, conference presentation, gather feedback |
| 1 | Q3-Q4 | Develop patches, security audit |
| 2 | Q1-Q3 | Submit to community, address reviews, multiplatform testing |
| 3 | Q1-Q2 | Prepare for PostgreSQL 19/20 release |
| 4+ | - | Production deployments, ecosystem development |

---

## Conclusion

This TDE architecture provides enterprise-grade encryption without Oracle:
- ✓ Equivalent security and functionality
- ✓ Transparent operation for applications
- ✓ HA without compromise
- ✓ Regulatory compliance (GDPR/HIPAA/PCI-DSS)
- ✓ Significant cost savings
- ✓ Clear path to PostgreSQL core integration

Deploy confidently with proven, open-standard technology.
