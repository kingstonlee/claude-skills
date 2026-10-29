---
name: rds-snapshot-restore
description: Use when restoring an RDS snapshot to a temporary database instance for data investigation or backfill — covers snapshot inspection, instance configuration, restore, password resets, and MySQL client compatibility workarounds
---

# RDS Snapshot Restore

## Overview

Restores an RDS snapshot to a temporary DB instance in a non-production environment, resets passwords, and verifies connectivity. Covers the MySQL 9.x local client incompatibility workaround (`ERROR 2059`).

> Replace every `<...>` placeholder with values from your own environment. Never hard-code production subnet groups, security groups, or credentials into a temp restore.

## When to Use

- Data recovery / backfill from a production snapshot
- Investigating deleted records
- Spinning up a read-only temp copy for a support investigation

## Steps

### 1. Inspect the snapshot

```bash
aws rds describe-db-snapshots \
  --db-snapshot-identifier <snapshot-id> \
  --region <region> --output json
```

Note: `Engine`, `EngineVersion`, `AllocatedStorage`, `Iops`, `StorageType`, `KmsKeyId`, `MasterUsername`.

### 2. Get source instance config (for reference)

```bash
aws rds describe-db-instances \
  --db-instance-identifier <source-db-id> \
  --region <region> \
  --query 'DBInstances[0].{SubnetGroup:DBSubnetGroup.DBSubnetGroupName,SecGroups:VpcSecurityGroups,Class:DBInstanceClass}'
```

For temp instances, always use your **non-prod** subnet group and a **non-prod** security group — never the production values returned above.

### 3. Choose instance name and class

- **Naming convention:** make it identifiable and clearly temporary, e.g. `temp-<snapshot-base-name>-<date>`
- **Instance class:** downsize from prod (e.g. `db.m6i.2xlarge` instead of `db.m6i.8xlarge`)
- **Multi-AZ:** No
- **Publicly accessible:** No

Confirm name and class with whoever requested the restore before proceeding.

### 4. Restore the snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier <temp-db-name> \
  --db-snapshot-identifier <snapshot-id> \
  --db-instance-class db.m6i.2xlarge \
  --db-subnet-group-name <nonprod-subnet-group> \
  --vpc-security-group-ids <nonprod-security-group-id> \
  --no-multi-az \
  --no-publicly-accessible \
  --storage-type io1 \
  --iops <iops-from-snapshot> \
  --region <region>
```

### 5. Wait for available

Poll until status is `available` (restores typically take ~15 min):

```bash
aws rds describe-db-instances \
  --db-instance-identifier <temp-db-name> \
  --region <region> \
  --query 'DBInstances[0].DBInstanceStatus' --output text
```

### 6. Reset master password

```bash
aws rds modify-db-instance \
  --db-instance-identifier <temp-db-name> \
  --master-user-password <new-password> \
  --apply-immediately \
  --region <region> \
  --query 'DBInstance.{Status:DBInstanceStatus,Endpoint:Endpoint.Address}'
```

### 7. Change app user passwords

**Do NOT use the local MySQL client if it is version 9.x** — it cannot load `mysql_native_password` and will fail with `ERROR 2059`. Use Docker with a pinned 8.0 client instead:

```bash
docker run --rm mysql:8.0 mysql \
  -h <endpoint> \
  -u <master-user> -p<master-password> \
  -e "ALTER USER '<app-user>'@'%' IDENTIFIED BY '<new-password>'; FLUSH PRIVILEGES;"
```

## Quick Reference

| Parameter | Value |
|-----------|-------|
| Subnet group (temp) | your non-prod subnet group |
| Security group (temp) | your non-prod DB security group |
| Multi-AZ | No |
| Publicly accessible | No |
| MySQL client (local 9.x) | Use `docker run --rm mysql:8.0` |

## Common Mistakes

- **Using prod subnet/security groups** — always switch to non-prod for temp instances
- **Keeping prod instance class** — downsize; temp instances don't need the prod footprint
- **Using local MySQL 9.x client** — it drops `mysql_native_password`; use Docker MySQL 8.0
- **Forgetting the date in the DB name** — include the snapshot date so it's identifiable and reap-able
