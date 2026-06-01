---
layout: post
title: "Database Replication & HA - Nhân bản và sẵn sàng cao"
date: 2026-06-01
categories: [database]
tags: [replication, failover, raft, split-brain, high-availability]
---

## Mục lục
1. [Góc nhìn tổng quan - Hệ thống dự phòng](#overview)
2. [Primary-Replica Replication](#primary-replica)
3. [Synchronous vs Asynchronous](#sync-async)
4. [Failover - Chuyển đổi tự động](#failover)
5. [Split-Brain Problem](#split-brain)
6. [Raft Consensus Algorithm](#raft)
7. [Read Replicas - Scale đọc](#read-replicas)
8. [PostgreSQL Streaming Replication](#postgres-repl)
9. [Cloud managed HA (RDS Multi-AZ)](#cloud-ha)
10. [Tổng kết và architecture patterns](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

Database replication giống **hệ thống văn phòng dự phòng**:

- **Primary** = văn phòng chính (nhận mọi yêu cầu viết)
- **Replica** = văn phòng chi nhánh (copy tài liệu từ trụ sở, phục vụ đọc)
- **Sync replication** = fax NGAY lập tức mọi thay đổi (chậm nhưng nhất quán)
- **Async replication** = gửi bản copy cuối ngày (nhanh nhưng có thể thiếu data mới nhất)
- **Failover** = khi trụ sở cháy, chi nhánh lên làm trụ sở mới
- **Split-brain** = 2 chi nhánh cùng tự nhận mình là trụ sở → conflict!

---

## 2. Primary-Replica Replication {#primary-replica}

```
        Writes            Reads
          ↓                 ↓
    ┌──────────┐     ┌──────────┐
    │ Primary  │────▶│ Replica 1│ (read-only)
    │ (Leader) │────▶│ Replica 2│ (read-only)  
    └──────────┘     └──────────┘
         │
    WAL/Binlog stream (changes)

Benefits:
- Read scaling: distribute reads across replicas
- HA: promote replica if primary fails
- Backups: backup from replica (no impact on primary)
- Geographic: replica near users (low latency reads)

Limitations:
- Single point of write (primary bottleneck)
- Replication lag (stale reads from replica)
- Failover complexity
```

---

## 3. Synchronous vs Asynchronous {#sync-async}

```
SYNCHRONOUS:
  Client → Write to Primary → Replicate to Replica → ACK to Client
  ✅ No data loss (replica always has latest)
  ❌ Higher latency (wait for replica)
  ❌ Primary blocked if replica slow/down
  Use: Financial data, critical systems

ASYNCHRONOUS:  
  Client → Write to Primary → ACK to Client (immediately)
  Background: Primary → Replicate to Replica (eventually)
  ✅ Low latency writes
  ✅ Primary not affected by replica issues
  ❌ Data loss possible if primary crashes before replication
  ❌ Replication lag (reads from replica may be stale)
  Use: Most applications, read-heavy workloads

SEMI-SYNCHRONOUS:
  Write confirmed when AT LEAST 1 replica acknowledges
  Balance between safety and performance
```

---

## 4-5. Failover và Split-Brain {#failover}

### Failover Process
```
1. Detection: Primary health check fails (heartbeat timeout)
2. Election: Choose new primary (most up-to-date replica)
3. Promotion: Selected replica becomes new primary
4. Reconfiguration: Other replicas point to new primary
5. Client redirection: Applications connect to new primary

Automatic failover tools:
- PostgreSQL: Patroni, pg_auto_failover
- MySQL: MySQL InnoDB Cluster, Orchestrator
- Cloud: RDS Multi-AZ (automatic)
```

### Split-Brain Prevention
```
Split-brain = TWO nodes both think they are primary
→ Both accept writes → DATA DIVERGENCE → DISASTER!

Prevention:
1. Quorum/Fencing: Majority vote required to be primary
2. STONITH: "Shoot The Other Node In The Head" 
   (fence failed node: power off, network isolate)
3. Witness/Arbiter: Third node breaks tie
4. Raft/Paxos: Formal consensus algorithm
```

---

## 6. Raft Consensus {#raft}

```
Raft ensures EXACTLY ONE leader at any time:

States: Follower → Candidate → Leader

Election:
1. Leader sends heartbeats every 150ms
2. If follower receives no heartbeat for 300ms → becomes Candidate
3. Candidate requests votes from all nodes
4. Majority votes → becomes new Leader
5. New Leader sends heartbeats

Term numbers prevent stale leaders:
- Each election increments term
- Node with higher term wins disputes
- Stale leader (lower term) steps down

Used by: etcd (K8s), CockroachDB, TiKV, Consul
```

---

## 7-10. Read Replicas, PostgreSQL, Cloud HA {#read-replicas}

### PostgreSQL Streaming Replication
```bash
# Primary: postgresql.conf
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = 'replica1'

# Replica: recovery or standby.signal
primary_conninfo = 'host=primary port=5432 user=replicator'
```

### AWS RDS Multi-AZ
```
Multi-AZ (HA):
- Synchronous standby in different AZ
- Automatic failover (30-120 seconds)
- Same endpoint (DNS switch)
- NOT for read scaling!

Read Replicas:
- Asynchronous replication
- Separate endpoint
- For read scaling
- Can be cross-region
- Can be promoted to standalone
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Designing Data-Intensive Applications Ch5 | Replication theory |
| Raft paper (raft.github.io) | Consensus algorithm |
| Patroni documentation | PostgreSQL HA |

---
