# Requirements — Financial Spend Monitor

## Personas
- **Analyst**: reviews customer spending patterns and flags anomalies

## In scope

REQ-001: An analyst can look up a customer by customer ID and see their
         current account status and risk tier.

REQ-002: An analyst can view a customer's card transactions from the
         last 30 days, sorted newest first, showing merchant name,
         category, amount, currency, and posted date.

REQ-003: The system computes a customer's average daily spend over the
         prior 12 months from historical transaction data.

REQ-004: The system presents last-30-day total spend alongside the
         12-month daily average so an analyst can spot unusual activity
         at a glance.

REQ-005: Open fraud alerts for a customer are returned alongside their
         transaction list, including severity and alert type.

REQ-006: A federated summary endpoint joins live card activity (last
         30 days) against the historical archive (prior 12 months) and
         returns both figures in a single response.

REQ-007: All list endpoints return results within 5 seconds under
         normal cluster load.

## Out of scope

- Authentication and user management
- Writing or modifying transaction records
- Cross-customer reporting or aggregations
- Real-time streaming or webhooks
