# Jira Database High Availability Options - FAQ

## Overview
This document addresses common questions about database high availability options for Jira Data Center deployments to meet SLA requirements.

---

## 1. If we backup and restore the DB on secondary can it still work?

### ✅ Answer: Yes, with limitations

Jira's disaster recovery guide supports a "cold standby" strategy where you backup and restore the database to a secondary instance.

### What Works:
- Database replication to standby instance is officially supported
- You can restore from database backups on the secondary instance
- Search index can be recreated from the database (though this takes significant time)

### ⚠️ Important Limitations:
- **Cold standby approach** - the standby instance isn't continuously running
- **Manual intervention required** to start standby and ensure it's ready for business use
- **Index recreation time** - can take significant time for large installations, reducing functionality until fully recovered
- **Additional replication needed** - must also replicate the shared home directory (attachments, avatars, installed apps)

### Implementation Requirements:
- Set `disaster.recovery=true` in jira-config.properties on standby instance
- Implement database replication strategy that meets your RPO/RTO requirements
- Configure file replication for shared home directory
- Test the disaster recovery process regularly

---

## 2. Can MSSQL replication work instead of using Always On?

### ✅ Answer: Yes, but Always On is the preferred modern approach

### SQL Server Replication Support:
- **Traditional replication supported** - Jira's disaster recovery guide specifically mentions that supported database suppliers provide their own replication solutions
- **Transactional, merge, snapshot replication** all work with Jira

### Always On Availability Groups (Recommended):
- **Supported since Jira 7.5.0** using Microsoft JDBC driver 6.2.1
- **Customer success** - Many customers successfully use Always On availability groups
- **Not officially tested** - Atlassian doesn't test Always On configurations, though customers use them successfully

### ⚠️ Important Considerations:
- **READ_COMMITTED_SNAPSHOT limitation** - Always On doesn't support this directive which Jira prefers
- **SQL Server Failover Clustering alternative** - Works better with Jira with minimal downtime (10-20 second window)
- **Reindexing after failover** - May be required with some configurations (resolved in Jira 7.9.2+)

---

## 3. Database Cluster Support for Jira

### ✅ OFFICIALLY SUPPORTED: PostgreSQL with PGpool-II

**NEW: Official clustering support has been added**

#### PostgreSQL + PGpool-II Features:
- **No single point of failure** - Eliminates database SPoF
- **Connection pooling** - Reduces overhead and improves performance  
- **Load balancing** - Built-in load balancer distributes requests across multiple PostgreSQL servers
- **Automatic failover** - Detects primary server failure and promotes standby automatically
- **Online recovery** - Supports high availability configurations

#### Supported Versions:
- Available across multiple Jira Data Center versions (9.4, 9.12, 9.13, 9.16, 10.7+)
- Also supported for Confluence Data Center

#### Implementation:
- Uses Docker images from Bitnami by VMware
- Requires PostgreSQL nodes with repmgr for replication management
- PGpool-II acts as middleware/proxy between Jira and PostgreSQL cluster
- Full setup documentation available in Atlassian official docs

### ✅ Other Officially Supported Options:
- **Amazon Aurora** (AWS cloud deployments only)
- **General clustering support** - Data Center documentation states "clustered database technology is supported and recommended"

### ❌ NOT Officially Supported (But May Work):
- **Oracle RAC** - Not officially tested by Atlassian
- **MySQL Cluster** - Known compatibility issues with hibernate_unique_key table
- **MSSQL Cluster** - Not officially tested

### 🔧 Community Workarounds:
- **Oracle RAC + Data Guard** - Some users report success (2017+ reports)
- **MySQL Galera Cluster** - Requires custom HAProxy configurations
- **PostgreSQL with PGpool-II** - Previously community-only, now officially supported

---

## Recommendations for Meeting SLA Requirements

### **Recommended Approach (Priority Order):**

1. **🥇 PostgreSQL with PGpool-II** 
   - ✅ **OFFICIALLY SUPPORTED**
   - Best choice for open-source stack
   - True database clustering with automatic failover
   - Eliminates database single point of failure

2. **🥈 SQL Server Always On Availability Groups**
   - Uses Microsoft JDBC driver 6.2.1
   - Customer-proven but not officially tested
   - Modern Microsoft HA solution

3. **🥉 Amazon Aurora**
   - If AWS cloud deployment is acceptable
   - Built-in high availability
   - AWS Quick Start available

4. **SQL Server Failover Clustering**
   - More reliable than Always On for Jira workloads
   - Brief failover window (10-20 seconds)
   - Well-tested with Jira

5. **Database + Application Layer HA**
   - Combine database HA with Jira Data Center clustering
   - Provides redundancy at multiple levels

### **High Availability Scripts:**
Your colleague's mention of MSSQL high availability scripts likely refers to:
- Always On Availability Groups setup automation
- Failover Clustering configuration scripts
- Both are viable options, with Failover Clustering being more tested for Jira

### **Key Takeaway:**
**PGpool-II with PostgreSQL** is now the **recommended approach** for achieving true database clustering with full Atlassian support, providing enterprise-grade high availability while eliminating the database as a single point of failure.

---

## Implementation Examples

### PGpool-II Setup (Basic Configuration)

#### 1. Primary PostgreSQL Node:
```bash
docker run --detach --rm --name pg-0 \
  -p 5432:5432 \
  --network my-network \
  --env REPMGR_PARTNER_NODES={PG-0-IP},{PG-1-IP} \
  --env REPMGR_NODE_NAME=pg-0 \
  --env REPMGR_NODE_NETWORK_NAME={PG-0-IP} \
  --env REPMGR_PRIMARY_HOST={PG-0-IP} \
  --env REPMGR_PASSWORD=repmgrpass \
  --env POSTGRESQL_POSTGRES_PASSWORD=adminpassword \
  --env POSTGRESQL_USERNAME=jiradbuser \
  --env POSTGRESQL_PASSWORD=jiradbpass \
  --env POSTGRESQL_DATABASE=jiradb \
  bitnami/postgresql-repmgr:latest
```

#### 2. Standby PostgreSQL Node:
```bash
docker run --detach --rm --name pg-1 \
  -p 5432:5432 \
  --network my-network \
  --env REPMGR_PARTNER_NODES={PG-0-IP},{PG-1-IP} \
  --env REPMGR_NODE_NAME=pg-1 \
  --env REPMGR_NODE_NETWORK_NAME={PG-1-IP} \
  --env REPMGR_PRIMARY_HOST={PG-0-IP} \
  --env REPMGR_PASSWORD=repmgrpass \
  --env POSTGRESQL_POSTGRES_PASSWORD=adminpassword \
  --env POSTGRESQL_USERNAME=jiradbuser \
  --env POSTGRESQL_PASSWORD=jiradbpass \
  --env POSTGRESQL_DATABASE=jiradb \
  bitnami/postgresql-repmgr:latest
```

#### 3. PGpool-II Load Balancer:
```bash
docker run --detach --name pgpool \
  --network my-network \
  -p 5432:5432 \
  --env PGPOOL_BACKEND_NODES=0:{PG-0-HOST}:5432,1:{PG-1-HOST}:5432 \
  --env PGPOOL_SR_CHECK_USER=postgres \
  --env PGPOOL_SR_CHECK_PASSWORD=adminpassword \
  --env PGPOOL_USERNAME=jiradbuser \
  --env PGPOOL_PASSWORD=jiradbpass \
  --env PGPOOL_POSTGRES_USERNAME=postgres \
  --env PGPOOL_POSTGRES_PASSWORD=adminpassword \
  --env PGPOOL_AUTO_FAILBACK=yes \
  bitnami/pgpool:latest
```

### Jira Database Configuration
In your Jira `dbconfig.xml`, point to the PGpool instance:
```xml
<url>jdbc:postgresql://{PGPOOL-HOST}:5432/jiradb</url>
<username>jiradbuser</username>
<password>jiradbpass</password>
```

---

## Disaster Recovery Configuration

### jira-config.properties Settings:
```properties
# For standby instance
disaster.recovery=true

# Optional: Custom secondary home location
jira.secondary.home=/path/to/secondary/home
```

### Database Replication Verification:
```sql
-- Check replication status
SELECT * FROM repmgr.nodes;

-- Verify cluster health
SELECT application_name, state, sync_state 
FROM pg_stat_replication;
```

---

## Troubleshooting Common Issues

### PGpool-II Connection Issues:
- Verify all nodes can communicate with each other
- Check firewall rules for PostgreSQL ports (5432)
- Ensure repmgr is properly configured on all nodes

### Always On Availability Groups Issues:
- Upgrade to Jira 7.9.2+ to avoid reindexing after failover
- Use Microsoft JDBC driver 6.2.1 or later
- Monitor for READ_COMMITTED_SNAPSHOT compatibility

### General HA Considerations:
- Test failover scenarios regularly
- Monitor replication lag
- Implement proper backup strategies
- Document recovery procedures

---

## Additional Resources

- [Atlassian Disaster Recovery Guide for Jira](https://confluence.atlassian.com/enterprise/disaster-recovery-guide-for-jira-692782022.html)
- [Connecting Jira to PGpool-II](https://confluence.atlassian.com/adminjiraserver/connecting-jira-applications-to-pgpool-ii-1252332126.html)
- [High Availability Guide for Jira](https://confluence.atlassian.com/enterprise/high-availability-guide-for-jira-288657149.html)
- [Jira Data Center Architecture Options](https://confluence.atlassian.com/enterprise/atlassian-data-center-architecture-and-infrastructure-options-994321215.html)
- [Database Setup for PGpool-II](https://confluence.atlassian.com/doc/database-setup-for-pgpool-ii-1331233074.html)

---

## Quick Reference

| Database Solution | Support Level | Failover Time | Complexity | Recommended For |
|------------------|---------------|---------------|------------|-----------------|
| PGpool-II + PostgreSQL | ✅ Official | Automatic | Medium | Enterprise, Open Source |
| SQL Server Always On | ⚠️ Community | Automatic | Medium | Microsoft Environments |
| SQL Server Failover Cluster | ⚠️ Community | 10-20s | Medium | Microsoft Environments |
| Amazon Aurora | ✅ Official | Automatic | Low | AWS Cloud |
| Cold Standby | ✅ Official | Manual | High | Budget-constrained |

---

*Last Updated: July 2025*  
*Status: Verified with latest Jira Data Center documentation*  
*Version: 1.0*