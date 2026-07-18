---
title: "Why Your Inventory Valuation Doesn't Match Your Accounting, and How to Fix It (Odoo 19)"
description: "Under Perpetual (at invoicing) valuation, the Inventory Valuation report should tie out to the Stock Valuation account. This guide covers what Odoo 19 enforces by default, where the differences actually come from, how to check your own database, and why the usual fixes don't hold."
lang: en
version: "19.0"
topic: "Inventory valuation"
date: 2026-07-18
ref: inventory-valuation-gap
alt_url: /zh/guides/why-odoo-inventory-valuation-doesnt-match-accounting/
module: suite_data_guard
---
Under **Periodic (at closing)**, accounting entries are suggested manually in the inventory valuation report. Under **Perpetual (at invoicing)**, an entry is created automatically to value the inventory **when a product is billed or invoiced**.

This guide covers Odoo 19 under **Perpetual (at invoicing)** with AVCO or FIFO costing. (Standard costing behaves differently, since every unit carries the same product-level cost. It is out of scope here.)

## 1. The gap, and what manual balancing hides

A finance lead compares the on-hand total in the **Inventory Valuation** report against the balance of the **Stock Valuation** account and finds the two disagree. The usual response is a manual adjusting entry for the difference. The books balance, the period closes, the matter is considered handled.

What was written off is a **net** figure. A difference of a few thousand on the balance sheet can be the residue of a large under-billed exposure sitting against a large over-invoiced one.

A test case makes the point. Two purchase orders: PO1 received 10 units but billed 20; PO2 received 10 units and never billed.

| Cause | Effect on the GL | Amount |
|---|---|---|
| PO2 received not billed | GL under-records | +2,000 |
| PO1 over-billed (received 10, billed 20) | GL over-records | −1,000 |
| **Net difference, as reported** | | **1,000** |

Two errors in opposite directions, each material on its own, collapse into a single harmless-looking number. **The magnitude of the difference says nothing about the severity of what sits underneath it.** A gap of a few tens of thousands can be concealing a six-figure received-not-billed position netted against a six-figure over-billing.

If the cause is never established and the next period is handled the same way, the adjustment grows while the reports feeding business decisions quietly lose accuracy.

Structurally, the Inventory Valuation report is a **subledger** of the Stock Valuation account. The account is the control total; the report is the detail behind it. Once the two diverge, the inventory figure on the balance sheet is no longer fully supported by operational records, gross margin derived from cost of goods sold becomes unreliable, and period close loses its meaning: the closing number can be signed off but not reconstructed from the movements underneath it.

A few units of difference from currency rounding is expected. A structural, growing difference is not. It means transactions changed inventory without changing the general ledger in step, or changed the ledger with no stock movement behind it.

## 2. What Odoo 19 does by default

Establishing the defaults narrows the search considerably. Of the items below, only one is enforced by the system; the rest are configurable.

- **Costing method.** Set in general settings and on the product category: AVCO or FIFO.
- **What participates in valuation.** Products with type **Goods** and **Track Inventory** enabled. Goods without inventory tracking, services and combos never appear in the valuation report.
- **Return write-back (on by default).** In the return wizard, the *Update quantities on SO/PO* column is ticked by default, so confirming a return decrements the received or delivered quantity on the source document. The column sits behind the optional-columns control and is hidden by default.
- **Sales invoicing policy (defaults to Ordered quantities).** On a new database, a product of type Goods carries an invoicing policy of **Ordered quantities**, not Delivered quantities.
- **Purchase bill control (defaults to Received quantities).** The purchase-side equivalent is Bill Control, and for goods it defaults to **Received quantities**.
- **Lock Confirmed Sales / Lock Confirmed Orders (off by default).** Confirmed documents remain editable unless these settings are enabled.
- **Done stock moves are read-only (system-enforced).** A stock move in the Done state is read-only to all users and cannot be modified or deleted, so the movement that drives valuation stays fixed. This is the only item on the list that no configuration can bypass. Odoo 19 protects the data at its source.

### Two ledgers, and three ways into the general ledger

One point underpins everything that follows: in Odoo 19, **inventory valuation and the general ledger are two separate ledgers.**

The valuation subledger is computed in real time from stock movements, but it **never writes an accounting entry itself**. The general ledger has exactly three entry points: **document posting** (vendor bills and customer invoices), **location valuation accounts** (when one end of a move has a valuation account configured), and **period-end closing**. Nothing else books an entry.

This produces a result that is counter-intuitive and can be verified in thirty seconds: **under Perpetual (at invoicing), receiving goods produces no accounting entry at all.** Neither the source nor the destination location carries a valuation account, so the gate stays shut. The goods arrive in the warehouse and the general ledger does not move until the bill is posted.

One further change matters a great deal to finance teams: **Odoo 19 removed the interim goods-received account.** In earlier versions, a quantity or value mismatch between receipt and bill parked in a dedicated interim account — inelegant, but visible, reviewable, and someone eventually cleared it. That layer is gone; the difference now lands directly in the inventory account.

**In summary.** These defaults do not themselves create divergence. When the sequence is followed in full — purchase, receipt, bill raised from that receipt; sale, delivery, invoice raised from that delivery — the report and the account reconcile under deterministic rules, and the timing differences in between explain themselves.

What deserves attention is the combined effect of two defaults. The valuation entry is realised **at invoicing**, while sales invoicing is driven by default from the **quantity agreed on the order** rather than from **what physically moved** — and the purchase side is already anchored to received quantity. **The two default anchors are not symmetrical.** Nothing is broken out of the box, but the standard configuration does not by itself guarantee that sales invoicing follows what the warehouse actually did. That alignment is an operational decision, and it is the first configuration to review in any database where the report and the account have stopped agreeing.

## 3. Where the differences actually come from

Persistent divergence comes from operations that step outside the standard sequence. The recurring cases follow.

### Configuration A: invoicing policy set to Ordered quantities

**Invoicing anchored to the order rather than to delivery.** Bills and invoices are driven by the quantity agreed on the purchase or sales order rather than by what the warehouse actually moved. Goods can be invoiced before they ship, or ship and never be invoiced, and the delivered-equals-invoiced relationship that valuation depends on no longer holds.

This configuration is not wrong in itself. For services, for goods without inventory tracking, and for businesses where ordering and collection are a single act with no partial delivery and no returns, ordered and delivered quantities are identical and either anchor gives the same number. **Its precondition is that the state "ordered 10, delivered 9" does not exist in the business.** Once partial shipment, backorders or returns are possible, the anchor and the physical reality must diverge, and the consequence lands on both statements: revenue is recognised on the ordered quantity while cost of goods sold follows the stock movement that actually occurred, so the two are mismatched.

### Configuration B: invoicing policy set to Delivered quantities

**A bill or invoice that does not mirror its receipt or delivery.** In the standard flow the vendor bill is generated from the receipt and the quantities agree by construction. Keying a bill by hand at a different quantity, or raising one without reference to the receipt at all, separates the two tracks: **the general ledger records the billed quantity into the inventory account, while inventory valuation follows the stock move and recognises only what was received.** The two are computed independently, and the difference settles into the stock variation at period-end closing. The same applies on the sales side when the invoiced quantity does not match what was delivered.

Because Odoo 19 no longer has an interim account, this kind of difference **leaves no intermediate balance to investigate**. It inflates or deflates the inventory asset directly and shows up only as a shadow inside a net figure. Quantity validation therefore has to move forward, to the invoicing action itself — after the fact there is no account left to check.

**Overriding the return write-back.** The *Update quantities on SO/PO* column is ticked by default and hidden by default. Unticking it is a deliberate act: the goods move back, but the received or delivered quantity on the source document is untouched. The document continues to read as fully received or delivered against stock that is no longer there, and every invoicing decision built on that quantity is wrong from that point on. Because the column is hidden, the override is difficult to spot when the transaction is reviewed later.

**Cost corrections that arrive after the goods have shipped.** The value of an outgoing move is fixed **at the moment the delivery is completed**, and a vendor bill posted later does not rewrite it. So when a landed cost or a corrected bill price is entered after the goods have been sold, the units still on hand are revalued correctly and **the units already shipped never receive the new cost**. Under AVCO and FIFO no price variance entry is generated for them — the variance account belongs to standard costing — so the difference can only be absorbed by revaluing stock on hand, and for the shipped portion there is no stock left to revalue.

That residual stays in the inventory account until period-end closing sweeps it into stock variation. **The total therefore ends up correct; the classification does not.** An amount that belongs in cost of goods sold is recorded as stock variation instead, and gross margin is distorted accordingly. This class of difference never shows up in a balance; it shows up in margin.

### One difference that looks like a gap but is not

When a vendor bill posts at a price different from the purchase order price, AVCO and FIFO treat the bill price as the true cost and revalue the stock move accordingly. Ledger and subledger are updated together and no divergence is created. Recognising this case matters as much as recognising the real ones — it is exactly the kind of self-resolving noise that conceals genuine exposure once everything is netted into a single adjustment.

Beyond these, mixing invoicing policies within one product category (invoicing policy is set per product, while the valuation account is set no lower than the category), allowing negative stock, delivering ahead of the vendor bill, and manufacturing costs such as labour and depreciation each have their own mechanics and are not enumerated here.

**In summary.** Under Perpetual (at invoicing), a difference arises when the sequence of operations makes the billed or invoiced quantity something other than the ordered or actually delivered quantity.

## 4. Checking your own system

Before commissioning any analysis, these steps establish the size and direction of the problem.

1. **Confirm the architecture (thirty seconds).** Receive goods on the target instance and look at the journal items. **No entries** confirms Odoo 19 Perpetual (at invoicing), and everything in this guide applies. If a receipt does post an entry, the premise does not hold and the situation needs separate assessment.
2. **Establish the gap and its direction.** At a single cut-off date, compare the Inventory Valuation report total with the Stock Valuation account balance and record the signed difference. Direction matters as much as amount.
3. **Check whether valuation modes are mixed.** Review the valuation setting on each product category. If some are Periodic and some Perpetual, they cannot share one valuation account and cannot be reconciled the same way.
4. **Strip out the timing differences.** At the same date, quantify what has been received but not billed and delivered but not invoiced. This is expected timing, not error. What remains after removing it is the real gap.
5. **Review the anchors.** Check the sales invoicing policy and the purchase bill control on your highest-volume goods. Ordered quantities on a fast-moving stocked product is the most common structural cause.
6. **Decompose to causes and test the sum.** Quantify received-not-billed, delivered-not-invoiced, over-billing, and deleted posted bills separately. Their **algebraic sum should equal the reported difference.** If it does not, a fifth cause is present and has not yet been identified.

These steps establish the size and rough location of the difference. Tracing it to individual documents requires analysis against your own chart of accounts and process design.

> **Important: structural errors must be reconciled on a posted-quantity basis before period-end closing is run.**

## 5. The existing answers, and where they stop

Responses to a valuation gap fall into several layers. None of them seals the source.

**Manual adjustment** is the most common and the weakest. It balances the total for the close and leaves no record of what the difference consisted of, hiding gross exposure behind a net figure.

**Period-end closing** is what many businesses actually rely on, and its nature is worth stating plainly: **closing is an eraser, not a corrector.** It sweeps every divergence between valuation and the ledger into stock variation without distinguishing between them. Returns with no credit note, invoiced quantities that do not match what shipped, deleted posted bills, over-billing — once swept, each is legitimised as an expense and the trail is permanently lost. The total does balance. The cause becomes unrecoverable.

**The native lock settings** (*Lock Confirmed Sales*, *Lock Confirmed Orders*) are off by default and, when enabled, prevent confirmed documents from being edited. They constrain whether a document **can be changed**, not whether quantities **can disagree**. With both enabled, a receipt of 10 can still be billed at 20, a return can still be confirmed with the write-back unticked, and a late bill still cannot revalue what has already shipped. **None of the causes above is the result of someone editing a confirmed order.**

**Detective tooling** — audit trails, lock dates, change logs — records who did what, after the event. It is genuinely useful for accountability, but it does not stop the transaction and does not reach the operational layer where the divergence is created: order lines, pickings, returns and reconciliation.

**Corrective tooling** — the various valuation adjustment modules — lists the products where report and account disagree and posts an entry to force them into line. The engineering is sound and it helps at close, but it treats the symptom. The next hand-keyed bill, the next return with the write-back overridden, reopens the gap and the module runs again next month.

What none of these layers provides is **prevention**: constraining the operation to the standard sequence at the moment it is attempted, and surfacing the exposure on the document rather than as an aggregate at month end.

## 6. How SuiteState approaches it

SuiteState runs its own wholesale operation on Odoo and met this problem from the inside: a Stock Valuation account that would not tie out to the Inventory Valuation report, traced back through receipts, returns and late bills to a small set of operations that had stepped outside the standard sequence — with a net difference far smaller than the gross exposure on either side of it.

Two things came out of that work, and they are deliberately different in kind.

**A diagnostic service** — a read-only analysis that decomposes an existing gap into its causes (received not billed, delivered not invoiced, returns without a credit note, price variance that has already self-resolved) until the residual is zero and every component traces to specific documents. It is not offered as a generic tool because it cannot be one: the decomposition depends on the client's chart of accounts, product categories, process design, and where their accounting emphasis sits. A tool that assumed a single shape of business would return a confident number that was quietly wrong. The analysis is built against the client's own environment.

**A prevention product** — [Data Guard](https://apps.odoo.com/apps/modules/19.0/suite_data_guard/) holds operations to the standard sequence and surfaces the signed difference between moved quantity and posted quantity on the order line itself: positive for over-invoicing, negative for stock that moved and was never billed. The exposure stops being an aggregate question on a report and becomes a specific item on a close checklist, so the gap does not quietly reopen once it has been closed.

If your inventory value and your general ledger have stopped agreeing, the first step is not another adjusting entry. It is establishing what the difference is actually made of.
