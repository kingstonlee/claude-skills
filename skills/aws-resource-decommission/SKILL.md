---
name: aws-resource-decommission
description: Use when verifying AWS resources are safe to release or terminate — Elastic IPs, stopped EC2 instances, EBS volumes, or other idle resources before deletion
---

# AWS Resource Decommission

## Overview

Before releasing or terminating any AWS resource, verify it has no recent activity and no active dependencies. The key check is CloudTrail: if no API calls touch the resource in 90 days, it's effectively abandoned. Association state and stopped duration are secondary signals.

## Elastic IPs

### 1. Confirm unassociated

```bash
aws ec2 describe-addresses \
  --filters "Name=allocation-id,Values=<eipalloc-id>" \
  --region <region> \
  --query 'Addresses[0].{IP:PublicIp,AssociationId:AssociationId,InstanceId:InstanceId}'
```

Safe if `AssociationId` and `InstanceId` are both null/empty.

### 2. Check CloudTrail for last activity

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<eipalloc-id> \
  --start-time $(date -u -v-90d +%Y-%m-%dT%H:%M:%SZ) \
  --region <region> \
  --query 'Events[*].{Time:EventTime,Event:EventName}'
```

On Linux: `$(date -u --date='90 days ago' +%Y-%m-%dT%H:%M:%SZ)`

No results = not touched in 90 days = safe to release.

### 3. Release

```bash
aws ec2 release-address --allocation-id <eipalloc-id> --region <region>
```

## Stopped EC2 Instances

### 1. Confirm stopped and get metadata

```bash
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --region <region> \
  --query 'Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,Name:Tags[?Key==`Name`].Value|[0],Launched:LaunchTime}'
```

### 2. Check CloudTrail for last API activity

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<instance-id> \
  --start-time $(date -u -v-90d +%Y-%m-%dT%H:%M:%SZ) \
  --region <region> \
  --query 'Events[*].{Time:EventTime,Event:EventName,User:Username}'
```

No results = no one has started, stopped, modified, or accessed it in 90 days.

### 3. Check attached EBS volumes

```bash
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --region <region> \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].{Device:DeviceName,VolumeId:Ebs.VolumeId}'
```

For a test/VPN instance stopped 12+ months: snapshot is optional, terminate without snapshot unless the name suggests it holds irreplaceable data.

### 4. Terminate

```bash
aws ec2 terminate-instances --instance-ids <instance-id> --region <region>
```

## Decision Table

| Signal | Safe? |
|--------|-------|
| No CloudTrail events in 90 days | Yes |
| EIP unassociated since 12+ months | Yes |
| Instance stopped 12+ months, test/dev name | Yes |
| Name contains "prod", "primary", "db" | Verify with team first |
| IP name suggests external whitelist (VPN, egress) | Check external allowlists before releasing |

## Common Mistakes

- **Releasing EIPs whitelisted in external firewalls** — names that imply a fixed egress/ingress role (e.g. a NAT or VPN IP) suggest the address may be allowlisted in third-party systems; confirm with your network team if unsure
- **Terminating without checking EBS** — volumes are deleted with the instance by default; if the volume has valuable data, snapshot first
- **90-day window too short for rarely-used systems** — for DR/backup instances, extend to 180 days
- **macOS date syntax** — `date -v-90d` is macOS; Linux uses `--date='90 days ago'`
