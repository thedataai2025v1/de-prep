# Snowflake Interview Preparation Guide

## Overview

| Category | Details |
|---|---|
| **Role Level** | Principal Data Engineer (17+ years) |
| **Target Companies** | Atlassian, Snowflake, Databricks, Uber, Confluent, Google, Adobe |
| **Interview Frequency** | Very High — core platform knowledge for every data engineering role |
| **Topics** | Architecture, Virtual Warehouses, Zero-Copy Cloning, Micro-partitions, Time Travel, Streams & Tasks, Semi-structured Data, Security, Performance Tuning, Cost Optimization |

## Files

| File | Description | Questions |
|---|---|---|
| [Theory Questions](./01-Theory-Questions.md) | 30 theory questions covering architecture, storage, compute, security, and optimization | Q1–Q30 |
| [Practical Questions](./02-Practical-Questions.md) | 30 hands-on coding & design scenarios with solutions | Q1–Q30 |

## How to Use

1. **Read the theory file first** — understand the architecture and concepts
2. **Study the practical questions** — each one mirrors a real interview scenario
3. **Practice explaining the solutions aloud** — focus on trade-offs and Principal-level thinking
4. **Review the follow-up questions** — these are what interviewers actually ask after your initial answer

## Topic Index

### Theory Questions (Q1–Q30)
- Q1: Snowflake Architecture (3-Layer Design)
- Q2: Virtual Warehouses & Concurrency Scaling
- Q3: Cloud Services Layer
- Q4: Micro-partitions & Data Sorting
- Q5: Zero-Copy Cloning
- Q6: Time Travel & Fail-safe
- Q7: Data Loading (COPY INTO)
- Q8: Semi-structured Data (VARIANT, PARSE_JSON)
- Q9: Streams & Tasks (CDC & Orchestration)
- Q10: Materialized Views
- Q11: Secure Views & Dynamic Data Masking
- Q12: Resource Monitors & Cost Management
- Q13: Performance Optimization & Query Acceleration
- Q14: Snowpipe vs Bulk Load
- Q15: Snowflake vs Redshift vs BigQuery
- Q16: Result Caching
- Q17: Snowpark
- Q18: Data Sharing & Data Marketplace
- Q19: Dynamic Tables
- Q20: Data Retention & Fail-safe
- Q21: Pruning & Query Optimization
- Q22: Network Policies & IP Access Control
- MFA & RBAC
- Q24: Multi-cluster Warehouses
- Q25: Query Acceleration Service (QAS)
- Q26: Data Governance & Columnar Access Control
- Q27: Stored Procedures & UDFs
- Q28: Replication & Multi-region
- Q29: Snowflake Editions & Feature Matrix
- Q30: Incident Response & Performance Debugging

### Practical Questions (Q1–Q30)
- Q1: Semi-structured JSON parsing with FLATTEN
- Q2: Zero-copy clone for development environments
- Q3: Time travel query for data recovery
- Q4: Stream-based incremental processing
- Q5: Materialized view design
- Q6: Dynamic table with target refresh lag
- Q7: Query performance tuning with EXPLAIN
- Q8: Secure view with row-level security
- Q9: Snowpipe auto-ingestion setup
- Q10: VARIANT column for flexible schema
- Q11: MERGE statement for upserts
- Q12: Window functions for running totals
- Q13: Task scheduling with dependencies
- Q14: Data masking with masking policies
- Q15: CROSS JOIN UNNEST for semi-structured data
- Q16: Cost analysis query on query history
- Q17: Deduplication with QUALIFY
- Q18: SCD Type 2 with Streams
- Q19: Multi-cluster warehouse configuration
- Q20: Columnar access control
- Q21: Snowpark Python UDF
- Q22: Replication between regions
- Q23: Session variables & QUALIFY
- Q24: Complex pivot/unpivot operations
- Q25: Cohort analysis with window functions
- Q26: Change tracking for CDC
- Q27: Resource monitor alerts
- Q28: JSON path queries with nested objects
- Q29: Automated data quality checks
- Q30: End-to-end data pipeline design

## Tips for Principal-Level Interviews

1. **Always discuss trade-offs** — every design choice has pros and cons
2. **Quantify** — use real numbers (credit costs, scaling thresholds, latency)
3. **Show breadth** — connect Snowflake to the wider data platform (Kafka, Airflow, dbt, Spark)
4. **Discuss failures** — explain what happens when things go wrong and how you recover
5. **Think about cost** — Principal engineers own the budget and optimization strategy
