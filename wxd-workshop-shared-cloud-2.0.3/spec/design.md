# Design — Financial Spend Monitor

## Stack

- **Language**: Python 3.11
- **Framework**: FastAPI
- **Cassandra client**: `cassandra-driver` (reuse `.venv` from connect-workshop.sh)
- **Presto client**: raw HTTP + Bearer token (mirror `smoke_test.py` pattern)
- **Config**: `python-dotenv` reading `.env` written by `connect-workshop.sh`

---

## Data sources by REQ

| REQ | Store | Table(s) |
|-----|-------|----------|
| REQ-001 | Cassandra | `financial_user38.customers` |
| REQ-002 | Cassandra | `financial_user38.card_status_current`, `financial_user38.card_transactions_recent` |
| REQ-003 | Iceberg (Presto) | `iceberg_data.financial_reference.transactions_archive` |
| REQ-004 | Cassandra + Iceberg | federated via Presto (see REQ-006) |
| REQ-005 | Cassandra | `financial_user38.fraud_alerts_open` |
| REQ-006 | Federated (Presto) | `cassandra.financial_user38.card_transactions_recent` + `iceberg_data.financial_reference.transactions_archive` |
| REQ-007 | — | enforced by 5s timeout on all queries |

---

## Endpoints

### GET /customers/{customer_id}
**REQ-001**

Cassandra single-row lookup.

```python
SELECT customer_id, first_name, last_name, email,
       account_status, risk_tier, kyc_verified, updated_at
FROM financial_user38.customers
WHERE customer_id = ?
```

### GET /customers/{customer_id}/transactions
**REQ-002, REQ-005**

Two Cassandra reads, merged in-app:

**Step 1** — get the customer's cards:
```python
SELECT card_id FROM financial_user38.card_status_current
WHERE customer_id = ?   -- secondary index
```

**Step 2** — for each card, fetch last-30d transactions:
```python
SELECT card_id, txn_date, transaction_id, merchant_name,
       merchant_category, amount, currency, posted
FROM financial_user38.card_transactions_recent
WHERE card_id = ?
  AND txn_date >= ?     -- today - 30 days
ORDER BY txn_date DESC, transaction_id DESC
```

**Step 3** — fetch open fraud alerts:
```python
SELECT alert_id, raised_at, severity, alert_type,
       triggering_txn_id, risk_score
FROM financial_user38.fraud_alerts_open
WHERE customer_id = ?
```

Response merges transactions + fraud alerts, sorts by `txn_date DESC`.

### GET /customers/{customer_id}/spend-summary
**REQ-003, REQ-004, REQ-006**

Single federated Presto query joining hot Cassandra data against the
cold Iceberg archive. This is the workshop's core demo.

```sql
SELECT
    recent.customer_id,
    recent.spend_last_30d,
    recent.txn_count_last_30d,
    hist.avg_daily_spend_12mo,
    hist.total_spend_12mo,
    ROUND(recent.spend_last_30d / 30.0, 2) AS avg_daily_spend_recent,
    ROUND(
        (recent.spend_last_30d / 30.0) / NULLIF(hist.avg_daily_spend_12mo, 0),
        2
    ) AS spend_ratio
FROM (
    SELECT
        customer_id,
        SUM(amount)   AS spend_last_30d,
        COUNT(*)      AS txn_count_last_30d
    FROM cassandra.financial_user38.card_transactions_recent
    WHERE customer_id = '{{customer_id}}'
      AND txn_date >= DATE '{{thirty_days_ago}}'
) recent
JOIN (
    SELECT
        customer_id,
        SUM(amount)                                      AS total_spend_12mo,
        SUM(amount) / NULLIF(COUNT(DISTINCT txn_date), 0) AS avg_daily_spend_12mo
    FROM iceberg_data.financial_reference.transactions_archive
    WHERE customer_id = '{{customer_id}}'
      AND txn_year  >= {{year_12mo_ago}}
      AND txn_month >= {{month_12mo_ago}}
      AND status    =  'posted'
) hist ON hist.customer_id = recent.customer_id
```

`spend_ratio > 1.0` means the customer is spending above their historical
daily average; `> 2.0` is a flag-worthy spike.

---

## Repo layout

```
wxd-workshop-shared-cloud-2.0.3/
  spec/
    requirements.md
    design.md           ← this file
  api/
    openapi.yaml        ← next
  src/
    main.py
    cassandra_client.py
    presto_client.py
    routes/
      customers.py
      transactions.py
      spend_summary.py
  tests/
    test_customers.py
    test_transactions.py
    test_spend_summary.py
```

---

## Connection notes

Both clients read from `.env`. See `AGENTS.md` for the Cassandra
`RouteEndPointFactory` pattern (required — without it the driver stalls
~15s per connect). Presto uses the two-step Bearer token flow from
`setup/lib/smoke_test.py`.

Partition key for `card_transactions_recent` is `(card_id)` with
clustering on `(txn_date DESC, transaction_id DESC)`. Queries must
supply `card_id` — scanning by `customer_id` alone requires the
secondary index on `card_status_current` first (Step 1 above).

Iceberg `transactions_archive` is partitioned by `txn_year, txn_month`.
Always filter on both to avoid full-table scans on the shared Presto
coordinator.
