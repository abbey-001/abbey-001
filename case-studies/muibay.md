# Muibay: daily operations and reconciliation for a fuel-station business

**Role:** sole engineer: requirements, data model, frontend, backend, deployment
**Stack:** Next.js 16 · React · TypeScript · Supabase (PostgreSQL, Auth, Edge Functions) · Zod · Recharts · PDF export · Web Push
**Status:** delivered to the client; not yet in daily use
**Code:** private (client work). This write-up describes the system without client data.

---

## The problem

Every day, each station manager has to answer a simple question that is hard to get right on paper: **does the fuel that left the tanks match the money that came in?**

Before Muibay, that reconciliation lived in notebooks and spreadsheets:
- Opening stock, deliveries, pump meter readings, closing stock, cash, transfers, POS, credit sales and bank deposits were all recorded in different places.
- Losses from leaks, short deliveries, meter errors or missing cash were found days later, if at all.
- The business owner had no single view across stations.

## What I built

**For station managers**, a guided **8-step daily report**:

1. **Opening stock** per product (petrol, diesel, gas), entered from tank dip measurements
2. **Deliveries**, with supplier, invoice and waybill numbers, and tank dip measurements before and after the delivery
3. **Pump sales** from opening and closing meter readings per pump nozzle, split into price periods when the pump price changes mid-day, with payments split across cash, transfer and each POS terminal
4. **Closing stock**
5. **Expenses** by category
6. **Debtors:** credit sales and repayments, kept as a running ledger per customer
7. **Banking:** deposits sent, later matched against amounts the bank confirms receiving
8. **Summary** and submission

**For the owner and admins**, a dashboard across all stations:
- Flags waiting for review, revenue and litres sold
- Product mix and performance per station
- A debtor ledger and banking status
- Inventory for side products (drinks, engine oils)
- User and station management, and reports exportable as PDFs

## The reconciliation logic

The core of the system is three automatic checks, run every time a report is saved:

| Check | Rule | What it catches |
|---|---|---|
| Fuel variance | For each product: `opening + deliveries − closing − pump sales`. Anything over **50 litres** is flagged | Leaks, theft, meter faults, recording errors |
| Delivery discrepancy | The tank dip after delivery is compared with the dip before plus the invoiced quantity. A shortage or surplus of 1 litre or more is flagged | Short deliveries from suppliers |
| Banking variance | Cash sent compared with the amount the bank confirms receiving | Missing or delayed deposits |

Flags carry the manager's note, so the owner sees the explanation next to the number. The owner is also notified when a report is submitted.

## Design decisions

- **Operational day, not calendar day.** Stations sell past midnight, so a "day" ends at a configurable cutoff (default 4am). Sales at 1am count toward the previous day's report, as staff expect.
- **Meters are the source of truth for sales.** Litres sold come from meter differences per nozzle, never from a typed total. Split price periods keep revenue correct when the pump price changes mid-shift.
- **A delivery is accepted by measurement, not by invoice.** The accepted quantity is what the tank dip actually shows. The invoiced quantity is kept for the supplier conversation.
- **Debtors as a ledger.** Credit sales are debits and repayments are credits, so every customer's balance can be traced back to the individual transactions.
- **Drafts first, then submit.** Managers can save partway through a shift. Flags are recalculated on every save, so they always reflect the latest numbers.
- **Accounts by invitation only.** Users are created through an Edge Function, and alerts go out as browser push notifications.

## What I'd improve next

- **Make saving atomic.** A draft save currently writes several tables in sequence from the server. I'd move it into a single PostgreSQL function, so a network failure can never leave a half-saved report.
- **Make the variance threshold configurable.** It's a fixed 50 litres today; I'd make it a setting per product and per station, since gas and diesel behave differently.
- **Add tests around the reconciliation maths**, the part of the system where a bug costs real money.
