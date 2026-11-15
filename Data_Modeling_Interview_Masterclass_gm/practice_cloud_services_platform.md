# Lesson 19: Practice — Cloud Services Platform Data Model (AWS / GCP / Azure Billing & Telemetry)

> **Principal Mentor Note:** Cloud infrastructure platforms (AWS, GCP, Azure, Snowflake) are the ultimate test of **high-cardinality usage metering** (`50B+ resource usage records/day`), **complex enterprise pricing & commitment discounts** (Reserved Instances, Savings Plans, Committed Use Discounts), **resource tag explosion**, and **FinOps cost allocation**.

---

## 1. Requirements & Scope Assumptions

* **Scale:** `5M customer organizations`, `200M active cloud resources` (EC2 instances, S3 buckets, Kubernetes pods, Serverless functions), `25B hourly/sub-hourly usage metering records/day`.
* **Core Business Processes:**
  1. **Resource Usage Metering** (`fct_usage_metering_hourly` — raw metered consumption units: vCPU-seconds, GB-hours, API calls, egress bytes).
  2. **Rated Billing & Discount Ledger** (`fct_billing_line_items_daily` — applies On-Demand rates, Reserved Instance / Savings Plan amortization, Spot pricing, and Enterprise Discount Programs).
  3. **Resource Lifecycle & Capacity Snapshot** (`fct_resource_daily_snapshot` — tracks provisioned capacity vs. utilized capacity for FinOps right-sizing).
  4. **Dynamic Customer Cost-Allocation Tags** (`dim_tag_set` / `MAP<STRING, STRING>` — handles arbitrary user-defined tags like `env:prod`, `team:checkout`, `cost_center:402`).
* **Key Analytical Queries:**
  * Daily Unblended vs. Amortized Net Spend by Customer Organization, Cloud Service (`Compute`, `Storage`, `AI/ML`), Region, and Customer `cost_center` Tag.
  * Savings Plan / Reserved Instance Utilization & Coverage Rate (`Covered Spend / Total Eligible Compute Spend`).
  * Cloud Infrastructure Gross Margin (`Customer Billed Revenue - Hardware/Power/Network Amortization Cost`).

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)              | Date | Hour | Org/Account | Cloud Service | Region/AZ | Resource | Pricing/Contract | Tag Set
-------------------------------------------+------+------+-------------+---------------+-----------+----------+------------------+--------
fct_usage_metering_hourly (Transaction)    |  X   |  X   |      X      |       X       |     X     |    X     |                  |    X
fct_billing_line_items_daily (Transaction) |  X   |      |      X      |       X       |     X     |    X     |        X         |    X
fct_commitment_amortization_daily (Snap)   |  X   |      |      X      |       X       |     X     |          |        X         |
```

---

## 3. High-Level Constellation Diagram

```
+------------------------+       +---------------------+       +------------------------+
| dim_organization [SCD2]|       |  dim_date [CONFORM] |       | dim_cloud_service[SCD1]|
| PK: org_sk             |       |  PK: date_sk        |       | PK: service_sk         |
| payer_account_id,      |       +----------+----------+       | service_code (EC2, S3) |
| linked_account_id,     |            ^            ^           | sku_code, usage_unit   |
| enterprise_tier        |            |            |           +-----------+------------+
+-----------+------------+            |            |                       | 1
            | 1                       |            |                       |
            |                 +-------+----+  +----+---------------+       | N
            | N               | fct_usage_ |  | fct_billing_line_  |<------+
            +---------------->| metering_  |  | items_daily [TXN]  |
                              | hourly     |  | Grain: 1 /         |
+------------------------+    | [TXN-HRLY] |  | (acct,res,sku,day) |    +------------------------+
| dim_region_az [SCD1]   |--->| Grain: 1 / |  +----+---------------+    | dim_pricing_contract   |
| PK: az_sk              |1:N | (res,sku,hr|       ^                    | [SCD2]                 |
| region, az, country    |    +------------+       | 1:N                | ON_DEMAND, SPOT,       |
+------------------------+                +--------+--------+           | SAVINGS_PLAN_3YR       |
                                          | dim_tag_set     |           +------------------------+
                                          | [HASH-DEDUPED]  |
                                          | tag_set_sk,     |
                                          | tags_map (MAP)  |
                                          +-----------------+
```

---

## 4. Core Production DDL & FinOps Modeling Patterns

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_organization (
    org_sk                  BIGINT PRIMARY KEY,
    linked_account_id       VARCHAR(32) NOT NULL,  -- Sub-account where resource runs
    payer_account_id        VARCHAR(32) NOT NULL,  -- Consolidated billing parent org
    enterprise_org_id       VARCHAR(32) NOT NULL,
    account_segment         VARCHAR(24) NOT NULL,  -- ENTERPRISE, GROWTH, SELF_SERVE
    edp_discount_pct        DECIMAL(5,4) NOT NULL, -- Enterprise Discount Program rate (SCD2!)
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- HASH-DEDUPLICATED TAG SET DIMENSION (Solves Arbitrary User Tag Explosion!)
-- Instead of storing 500 bytes of JSON tags on every one of 25B daily rows,
-- hash the sorted canonical tag map into a 64-bit surrogate key (tag_set_sk).
-- ============================================================================
CREATE TABLE dim_tag_set (
    tag_set_sk              BIGINT PRIMARY KEY,    -- XXHASH64(canonical_sorted_tags)
    tags_map                MAP<STRING, STRING>,   -- e.g., {'env': 'prod', 'team': 'payments'}
    cost_center_tag         VARCHAR(128),          -- Promoted top-level tag for fast GROUP BY
    environment_tag         VARCHAR(64),
    application_tag         VARCHAR(128)
);

-- ============================================================================
-- FACT: RATED DAILY BILLING LINE ITEMS (Transaction / Rating Fact)
-- Grain: One row per (usage_date_sk, linked_account_id, resource_id, sku_sk, pricing_contract_sk)
-- ============================================================================
CREATE TABLE fct_billing_line_items_daily (
    usage_date_sk           INT NOT NULL,
    org_sk                  BIGINT NOT NULL,
    payer_account_id        VARCHAR(32) NOT NULL,  -- Denormalized for fast tenant isolation pruning
    resource_id             VARCHAR(128) NOT NULL, -- Degenerate Dimension (200M+ cardinality)
    service_sk              INT NOT NULL,
    az_sk                   INT NOT NULL,
    pricing_contract_sk     INT NOT NULL,          -- ON_DEMAND, SPOT, RI_COVERED, SP_COVERED, FREE_TIER
    tag_set_sk              BIGINT NOT NULL,       -- FK -> dim_tag_set
    -- Additive Usage & Financial Measures
    usage_quantity          DECIMAL(20,6) NOT NULL,-- e.g., 24.000000 vCPU-hours or 1048.576 GB-hours
    public_on_demand_cost_usd DECIMAL(14,4) NOT NULL, -- List price before any discounts
    unblended_billed_cost_usd DECIMAL(14,4) NOT NULL, -- Cash invoice amount (0.00 for RI-covered usage!)
    amortized_net_cost_usd  DECIMAL(14,4) NOT NULL, -- True economic cost (includes upfront RI amortization)
    infra_hardware_cogs_usd DECIMAL(14,4) NOT NULL  -- Internal cloud provider cost for gross margin
)
PARTITION BY (usage_date_sk)
CLUSTER BY (payer_account_id, service_sk);
```

---

## 5. Validation Query (FinOps Amortized Spend & Savings Rate by Customer Tag)

```sql
-- Query: Enterprise Payer Amortized Spend, Effective Discount Rate & Tag Allocation
SELECT
    d.year_month,
    COALESCE(t.cost_center_tag, 'UNTAGGED')                         AS cost_center,
    s.service_category,
    SUM(b.public_on_demand_cost_usd)                                AS list_price_usd,
    SUM(b.unblended_billed_cost_usd)                                AS cash_billed_usd,
    SUM(b.amortized_net_cost_usd)                                   AS true_amortized_cost_usd,
    ROUND(
        1.0 - (SUM(b.amortized_net_cost_usd) / NULLIF(SUM(b.public_on_demand_cost_usd), 0)),
    4)                                                              AS effective_savings_rate
FROM fct_billing_line_items_daily b
JOIN dim_date d ON b.usage_date_sk = d.date_sk
JOIN dim_cloud_service s ON b.service_sk = s.service_sk
JOIN dim_tag_set t ON b.tag_set_sk = t.tag_set_sk
WHERE b.payer_account_id = 'ACCT-99887766'
  AND d.calendar_date >= DATE '2026-10-01'
GROUP BY 1, 2, 3
ORDER BY 6 DESC;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why do you store BOTH `unblended_billed_cost_usd` AND `amortized_net_cost_usd` on `fct_billing_line_items_daily`?"**
   * **Expected Insight:** If a customer pays `$8,760` upfront for a 1-year Reserved Instance (`$1/hr`), on January 1st their **Unblended Cash Bill** spikes by `$8,760` while their hourly EC2 usage rows show `$0.00`—making engineering teams think their servers are free and distorting daily cost trends. **Amortized Cost** spreads the upfront commitment evenly (`$1/hr`) across the actual resources that consumed the reservation, giving FinOps teams true daily unit economics while preserving `unblended_billed_cost_usd` for exact invoice reconciliation!
2. **Top-Up 2: "A Kubernetes cluster has 500 ephemeral pods spinning up and down every 10 minutes (`50,000` unique pod IDs per day). If we keep `resource_id = pod_id` at the daily billing grain, our billing table explodes by 100x. How do you model container/serverless cost allocation?"**
   * **Expected Insight:** Roll up ephemeral container telemetry in Silver to the **Kubernetes Cluster + Namespace + Deployment Label** grain (`cluster_id + namespace + deployment_name`) before writing `fct_billing_line_items_daily`, keeping raw pod-level metering in a short-retention (14-day) operational telemetry table.

---

## 7. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Hash-deduplicates arbitrary user tag maps (`dim_tag_set`)** instead of duplicating huge JSON strings on 25B daily rows. | Stores unnormalized `VARCHAR(4000)` JSON tag strings on every atomic metering row. |
| **Separates Unblended Invoice Cost from Amortized Cost** to handle Reserved Instances / Savings Plans accurately. | Uses a single `cost` column, breaking either invoice reconciliation or daily FinOps cost attribution. |
| **Treats ephemeral `resource_id` as a Degenerate Dimension** rather than building a 200M-row SCD2 dimension for short-lived VMs/pods. | Builds an SCD Type 2 `dim_resource` for 500M ephemeral containers that live for 4 minutes. |