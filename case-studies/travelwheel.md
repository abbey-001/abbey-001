# TravelWheel: a travel-booking platform built to survive unreliable suppliers

**Role:** CTO and lead engineer (since 2023): architecture, most of the code, code review, deployment; I lead a three-person team
**Scale:** ~1,500 users · ~400 bookings a month
**Stack:** Laravel 12 · PHP 8.2 · Livewire 4 · Filament 5 · MySQL · queues and scheduled jobs · PHPUnit
**Code:** private (company IP). This write-up covers architecture and decisions only. Partner names are left out.

---

## What TravelWheel does

TravelWheel sells travel to Nigerian individuals and companies:
- **Flights**, from two external flight-supplier APIs
- **Visa applications**
- **Airport services:** lounges, airport protocol (meet-and-assist), ground transport, car hire, transfers
- **Travel insurance** and **air cargo**
- **TravelFlex:** "travel now, pay later" flights financed through a lending partner
- **TravelWheel Corporate:** company accounts with wallets, negotiated rates, invoices and statements

Behind the customer site is a staff back office where every booking becomes a piece of work that someone owns, with a deadline.

## The hard part: building on suppliers you don't control

Flight supplier APIs fail, slow down and change prices within minutes, and a customer who has paid can't be told "sorry, the fare expired". Most of the engineering effort went into making sure that **money and bookings stay correct when something outside our control goes wrong**.

### 1. Automatic cut-off for failing suppliers (a circuit breaker)
Every search and price check is recorded. When a supplier's recent failure rate crosses a threshold, it is **paused** and left out of searches. After a cool-down it goes **on trial** with real traffic, and it is restored once calls succeed again. Thresholds can be set per supplier from an admin screen, so operations staff can react without a deploy.

### 2. A guard at the point where a booking becomes real
If a supplier is switched off between a customer's search and their booking, the booking is refused *before* anything exists at the supplier, and the same flight is offered from the other supplier instead. *After* a booking exists at the supplier, it always carries on to payment and ticketing. Switching a supplier off only stops **new** bookings, never strands a paid one.

### 3. Fares that expire mid-checkout
One supplier's fare tokens expire 10–15 minutes after pricing. By the time a customer has filled in passenger details and paid, the token was often stale, so the reservation failed *after payment*. The fix:
- The fare is **re-priced immediately before reserving**, with one cheap pricing call the customer never notices.
- Only if that token has expired is the flight searched for again and re-priced.
- If the flight has gone, or now costs more than the customer approved, **nothing is charged**.

### 4. "Hold and pay later" for a supplier that can't hold seats
For "travel now, pay later" (TravelFlex) and pay-later bookings, the hold lives on TravelWheel's side while the lending partner reviews the application. Nothing is reserved at the supplier until payment, at which point the fare is re-confirmed as above.

### 5. Corporate wallets that can't be double-spent
A single class is the only code allowed to write to a company wallet:
- Every entry **locks the company row inside a database transaction**, so two bookers at the same company can't spend the same balance.
- Every entry carries an **idempotency key**, so a payment webhook retry, a double-click or a re-run queue job posts exactly once.
- Balances may only go below zero down to the company's agreed credit limit.

### 6. Emails that are sent exactly once
Emails go through an outbox:
- Each email has a unique key, so a retry never sends a duplicate.
- Emails that no longer apply (for example, after a booking changes) can be cancelled before they go out.
- Internal alerts to many staff are queued, so the person who triggered them doesn't wait on the mail server.

## Running the operation: a workflow engine

Every booking type has a defined workflow; there are 14, covering flights, visas, lounges, protocol, car hire, insurance, cargo, support requests and more.
- **Steps and deadlines:** each step has a deadline, checked by a job every five minutes. Alerts escalate in three stages: the owner (or the whole team if nobody owns it), then the team by email when the deadline is missed, then the **CEO** if it's still open a while later.
- **Help and handoffs:** staff can ask a colleague for help or hand a booking over. Every action is written to the booking's history, and escalations sync with the team's issue tracker.
- **Permissions:** staff are grouped by department, money actions are limited to Finance, and every action is logged automatically.

## Engineering practice

- **1,100+ automated tests** across 99 test files, 119 versioned database migrations, 320+ commits
- **Design written before code:** features are planned in short written documents first (corporate flights, workflow system, TravelFlex repayments), each split into phases that can ship on their own
- **Supplier work on feature branches:** for example, supplier failover, hold-and-pay-later and refund penalties were each built on a branch and reviewed before merging

## What I'd highlight in an interview

- Designing around **failure of third parties**: circuit breakers, re-pricing just before booking, and a clear rule for where a booking becomes real.
- **Money correctness:** row locks and idempotency keys on wallets, and never charging for a fare that can't be honoured.
- Building **operations software for people**: deadlines, escalations and ownership, so nothing a customer paid for is forgotten.
