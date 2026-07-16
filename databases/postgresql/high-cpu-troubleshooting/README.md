---
Article Type: Engineering Case Study
Category: Database
Technology: PostgreSQL, AWS RDS
Difficulty: Advanced
Tags: PostgreSQL, AWS RDS, Performance, Database, CPU Optimization, Production Support
---

# How Do You Troubleshoot High Database CPU Usage in Production?

**Estimated Reading Time:** 7 Minutes

## Skills Demonstrated

- PostgreSQL
- AWS RDS
- Performance Tuning
- Production Troubleshooting
- Database Optimization
- Performance Monitoring
- Software Architecture

---

## Overview

Production performance issues rarely have a single root cause.

This case study shares how increasing data volume and customer growth resulted in high PostgreSQL CPU utilization, delayed background jobs, and slower reporting. Rather than immediately scaling infrastructure, the team followed a structured investigation to identify the actual bottleneck and implement a long-term architectural solution.

---

## Business Scenario

The application was a centralized **Data Lake** built on **PostgreSQL** running on **AWS RDS**. It stored encrypted prescription and transaction data received from more than **100 pharmacies**.

Each pharmacy generated approximately **500–700 transactions per day**. As additional pharmacies were onboarded, both the data volume and workload increased significantly.

The issue was not caused by a deployment or application bug. Instead, the system reached a point where growing data and increasing customer usage exposed an architectural bottleneck.

---

## Symptoms

The first production symptoms appeared during peak business hours, typically after **3:00 PM**.

The team observed:

- High PostgreSQL CPU utilization
- ETL and background job timeouts
- Growing processing queues
- Delayed transaction synchronization
- Slow pharmacy dashboards
- Customer complaints about missing or delayed transaction data

Although the Data Lake was not directly exposed to end users, downstream systems were affected by the processing delays.

---

## Investigation

The investigation followed a structured approach instead of assuming the infrastructure was the problem.

The team reviewed:

- AWS CloudWatch metrics
- AWS Performance Insights
- Slow query logs
- Long-running queries
- Blocking sessions
- Database locks
- Active connections
- Table sizes
- Missing indexes
- Query execution plans

Several optimizations were implemented:

- VACUUM ANALYZE
- Index optimization
- Query tuning
- Batch size optimization
- Job scheduling improvements
- Connection pooling

These changes improved the situation but did not eliminate the CPU spikes.

---

## Root Cause

The investigation identified that several frequently executed queries searched directly on encrypted columns.

A simplified version of the search looked like:

```sql
WHERE Decrypt(Column) = SearchValue
```

As the database continued to grow, decrypting large numbers of rows during filtering became increasingly expensive and placed significant pressure on the database CPU.

The largest contributors were the core transaction tables and request payload data, which had grown over many years of production usage.

---

## Engineering Decision

Instead of immediately scaling infrastructure again, the team focused on redesigning the search strategy.

The production database was temporarily upgraded to provide immediate stability while the engineering team worked on a permanent solution.

The long-term solution introduced **hash-based searching**.

```text
Legacy Design

User Search
      │
      ▼
WHERE Decrypt(Column) = Value
      │
      ▼
Decrypt Large Number of Rows
      │
      ▼
High CPU Usage


────────────────────────────────────────────


Improved Design

User Search
      │
      ▼
Generate Hash
      │
      ▼
Compare Hash Column
      │
      ▼
Find Matching Records
      │
      ▼
Decrypt Only Required Rows
      │
      ▼
Reduced CPU Usage
```

Instead of decrypting every candidate row, the application first compared hashed values and only decrypted the small number of matching records.

---

## Results

After implementing the redesigned search strategy:

- Database CPU utilization decreased by approximately **25–30%**
- Query execution improved by approximately **15–17%**
- Overall application performance improved by **22–26%**
- Background jobs completed successfully
- Dashboard responsiveness returned to normal
- Production stability improved significantly

---

## Lessons Learned

The immediate temptation during production incidents is often to scale infrastructure.

While upgrading the database instance provided temporary relief, it did not address the underlying problem.

The long-term improvement came from redesigning how encrypted data was searched.

> **Infrastructure upgrades solve immediate problems. Architectural improvements solve long-term problems.**

---

## Best Practices

- Investigate before scaling infrastructure.
- Review slow query logs and execution plans.
- Optimize indexes and database statistics.
- Measure every optimization.
- Avoid searching directly on encrypted columns whenever possible.
- Consider hash-based lookups for searchable encrypted data.
- Treat infrastructure scaling as a temporary mitigation rather than the final solution.

---

## Key Takeaways

- High CPU usage is often a symptom rather than the root cause.
- Database monitoring should always precede optimization.
- Query design has a greater long-term impact than hardware upgrades.
- Hash-based searching can significantly reduce the cost of querying encrypted data.
- Sustainable performance improvements come from architectural decisions rather than infrastructure alone.

---

Thank you for reading.

I hope this case study provides practical insights into troubleshooting production database performance issues in enterprise applications.

Feel free to explore the other Engineering Journal articles in this repository.

---

**Article Version:** 1.0

**First Published:** July 2026

**Last Reviewed:** July 2026