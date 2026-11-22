# TravelWheel: a travel-booking platform built to survive unreliable suppliers

**Role:** CTO and lead engineer (since 2023): architecture, most of the code, code review, deployment; I lead a three-person team
**Scale:** ~1,500 users · ~400 bookings a month
**Stack:** Laravel 12 · PHP 8.2 · Livewire 4 · Filament 5 · MySQL · queues and scheduled jobs · PHPUnit
**By the numbers:** 1,100+ automated tests · 119 database migrations · 92 models · 14 operational workflows · 320+ commits
**Code:** private (company IP). This write-up covers architecture and decisions only. Partner names are left out.

---

## What TravelWheel does

TravelWheel sells travel to Nigerian individuals and companies:
- **Flights**, from two external flight-supplier APIs, with changes after ticketing: void, refund and reissue
- **Visa applications**, from search to submission, with document requests and handover to visa vendors
- **Airport services:** lounges, airport protocol (meet-and-assist), ground transport, car hire, transfers
- **Travel insurance** and **air cargo**
- **TravelFlex:** "travel now, pay later" flights financed through a lending partner
- **TravelWheel Corporate:** company accounts with wallets, negotiated rates, saved travellers, invoices and statements

Behind the customer site is a staff back office where every booking becomes a piece of work that someone owns, with a deadline.

---

## Part 1: Flights on suppliers you don't control

Flight supplier APIs fail, slow down and change prices within minutes, and a customer who has paid can't be told "sorry, the fare expired".

### 1. Automatic cut-off for failing suppliers (a circuit breaker)
**Problem:** when one supplier's API degraded, every search waited on it and failed with it.
**Built:** every search and price check is recorded. When a supplier's recent failure rate crosses a threshold, it is **paused** and left out of searches. After a cool-down it goes **on trial** with real traffic, and it is restored once calls succeed again. Thresholds can be set per supplier from an admin screen.
**Why it matters:** one bad supplier no longer takes the whole flight search down, and operations staff can react without a deploy.

### 2. A guard at the point where a booking becomes real
**Problem:** a supplier could be switched off between a customer's search and their booking.
**Built:** before anything exists at the supplier, the booking is refused and the same flight is offered from the other supplier instead. Once a booking exists at the supplier, it always carries on to payment and ticketing.
**Why it matters:** switching a supplier off stops **new** bookings only. It never strands a customer who has already paid.

### 3. Fares that expire mid-checkout
**Problem:** one supplier's fare tokens expire 10–15 minutes after pricing. By the time a customer had entered passenger details and paid, the reservation often failed, *after* payment.
**Built:**
- The fare is re-priced immediately before reserving, with one cheap pricing call the customer never notices.
- Only if that token has expired is the flight searched for again and re-priced.
- If the flight has gone, or now costs more than the customer approved, **nothing is charged**.

**Why it matters:** it removed the worst outcome in travel e-commerce, a customer who has paid but has no ticket.

### 4. Changing a ticket after it's issued: quote first, then act
Void, refund and reissue each have a **quote** step before the real operation. Staff see what the airline will charge or refund before committing, so no change happens blind. For disrupted trips, staff can search for **replacement flights** directly from the booking.

---

## Part 2: Money

### 5. "Travel now, pay later" without taking on bad debt (TravelFlex)
**Problem:** offering instalments means some customers will stop paying. If they do, TravelWheel has to recover its money from somewhere.
**Built:** the deposit is sized so that, if a customer stops paying, **cancelling their refundable ticket still covers the loss**:
- The required deposit is the larger of a **30% minimum** and **refund penalties + extras + processing fee + a risk buffer**, rounded up to the next 10% step.
- Fares the airline won't fully disclose refund penalties for are refused automatically, as are fares that would need more than a 90% deposit.
- The repayment schedule is stored **encrypted**, and payments are matched to instalments by position and due date.
- Recording a repayment runs in a transaction with a row lock, so two staff members can't record the same instalment twice.
- Reminders stop repeating once an instalment is overdue, and only go to customers whose booking is still live.

**Why it matters:** the business risk is controlled by a rule in the code, not by staff judgement on each application.

### 6. Holding a fare when the supplier can't hold seats
While the lending partner reviews a TravelFlex application, the booking is held **on TravelWheel's side**; nothing is reserved at the supplier. When the customer pays the deposit, the fare is re-confirmed: re-priced if it's still there, and refused if it has gone or gone up.

### 7. Corporate wallets that can't be double-spent
A single class is the only code allowed to write to a company wallet:
- Every entry **locks the company row inside a database transaction**, so two bookers can't spend the same balance.
- Every entry carries an **idempotency key**, so a webhook retry, a double-click or a re-run queue job posts exactly once.
- Balances may go below zero only down to the company's agreed credit limit.

### 8. Prices the customer can trust
- **Visa fees:** the estimate shown at search uses the same fees, exchange rates and rounding as the quotation at checkout, so the price doesn't jump between search and payment.
- **Transfers:** the price is an itemised formula (base fare + wear-and-tear + fuel by drive time + admin fee), so staff can explain every naira.
- **Corporate rates:** each company can have its own service charges, and any rate left blank falls back to the default corporate rate.

---

## Part 3: Operations and people

### 9. A workflow engine so nothing paid for is forgotten
- **Workflows:** every booking type has one (14 in total), each with steps and deadlines checked every five minutes.
- **Escalation:** alerts go to the owner (or the whole team if nobody owns it), then to the team by email once the deadline is missed, then to the **CEO** if it's still open a while later.
- **Help and handoffs:** staff can ask for help or hand a booking over. Every action is written to the booking's history and synced with the team's issue tracker. Tracker IDs are looked up by name and cached, so nothing breaks if a team there is recreated.
- **Permissions:** staff are grouped by department, money actions are limited to Finance, and every action is logged automatically.

### 10. The visa pipeline
- **From search to submission:** eligibility checks, a quotation, payment, then review by an officer.
- **Document requests:** officers can ask for more documents. When the applicant uploads everything outstanding from their portal, the application **moves back to review on its own**, with no staff action needed.
- **Handover to visa vendors:** the full application with all uploads goes to each vendor contact as a **separate email**, so recipients never see each other's addresses. Every send is audited.
- **Vendor onboarding** follows the order of the company's paper process: documents verified, then compliance, then a decision.

### 11. Emails that are sent exactly once
All emails go through an outbox with a unique key per email. A retry never sends a duplicate, and emails that no longer apply can be cancelled before they go out.
**The subtle part:** the same guarantee can drop a message silently. If the ticket-issued email reused the key of the earlier booking-confirmed email, the outbox would treat it as already sent. So the rule is explicit in the ticketing code: a genuinely new email always gets its own key.

---

## Part 4: Security and reliability

### 12. Protecting personal data
- **Encryption:** passport and traveller details, TravelFlex applications and repayment plans, and vendor applications are **encrypted in the database**, not just protected by access rules.
- **Corporate sign-in:** uses one-time email codes that are **hashed, short-lived, single-use and limited to a few attempts**.

### 13. A design choice that came from an outage
Corporate sign-in codes are sent **directly, never through the queue**. A broken scheduler had once stalled the queue on staging; if sign-in depended on it, every company admin would be locked out. Critical paths shouldn't depend on background infrastructure that can fail silently.

### 14. A health dashboard for the whole system
I built **18 automated checks in 8 groups** that operations can run at any time:

| Group | Checks |
|---|---|
| Application | deployment configuration, PHP runtime and extensions, public routes and frontend assets |
| Background work | email delivery, queue backlog and failures, scheduler heartbeat |
| Data and infrastructure | database connection, migrations, cache and locks, storage |
| Integrations | flight supplier, payment gateway |
| Operations | corporate wallet balances, **stuck flight payments and ticketing exceptions**, reporting sync |
| Products and security | exchange rates and flight charges, product tables, sensitive-data protection |

Problems show up on a dashboard before customers report them.

### 15. Engineering practice
- **1,100+ automated tests** across 99 test files
- **Design written before code:** features are planned in short documents first (corporate flights, the workflow system, TravelFlex repayments), each split into phases that can ship on their own
- **Risky supplier work on feature branches**, reviewed before merging: supplier failover, hold-and-pay-later and refund penalties

---

## Interview story bank

| If they ask… | Tell |
|---|---|
| "A hard technical problem you solved" | #3, fares expiring after payment |
| "How do you design for failure?" | #1 circuit breaker, #2 where a booking becomes real, #13 sign-in that doesn't depend on the queue |
| "A subtle bug or edge case you guarded against" | #11, a duplicate-proof outbox that could silently drop a new email |
| "How do you handle money correctly?" | #7 wallets (row locks and idempotency), #5 TravelFlex deposit rule |
| "Balancing business risk and product" | #5 TravelFlex |
| "Working with non-engineers and operations" | #9 workflows and escalations, #10 visa pipeline |
| "Security" | #12 encryption and one-time codes |
| "How do you know production is healthy?" | #14 health dashboard |
| "Leading a team" | written design before code, phases that ship on their own, code review, the test suite |
