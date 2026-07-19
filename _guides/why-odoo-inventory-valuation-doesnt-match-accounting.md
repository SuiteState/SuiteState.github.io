---
title: "Why Your Inventory Valuation Doesn't Match Your Accounting, and How to Fix It (Odoo 19)"
description: "Under Perpetual (at invoicing) valuation, the Inventory Valuation report should tie out to the Stock Valuation account. This guide covers what Odoo 19 enforces by default, where the differences actually come from, how to check your own database, and why the usual fixes don't hold."
lang: en
version: "19.0"
topic: "Inventory valuation"
date: 2026-07-18
updated: 2026-07-19
ref: inventory-valuation-gap
alt_url: /zh/guides/why-odoo-inventory-valuation-doesnt-match-accounting/
module: suite_data_guard
---
**Periodic (at closing)** — accounting entries are suggested manually in the inventory valuation report.

**Perpetual (at invoicing)** — an entry is created automatically to value the inventory **when a product is billed or invoiced**.

This guide covers Odoo 19 under **Perpetual (at invoicing)** with AVCO or FIFO costing. (Standard costing behaves differently, since every unit carries the same product-level cost. It is out of scope here.)

## 1. The gap, and what manual balancing hides

A finance lead compares the on-hand total in the **Inventory Valuation** report against the balance of the **Stock Valuation** account and finds the two disagree. The usual response is a manual adjusting entry for the difference. The books balance, the period closes, the matter is considered handled.

What was written off is a **net** figure. A difference of a few thousand on the balance sheet can be the residue of several individually significant amounts cancelling each other out.

Consider two purchase orders. Both were received and partly sold, and in both cases the vendor bill arrived and was posted after the goods had already shipped — one at a higher price than ordered, one at a lower price.

| Purchase order | What happened | Effect on the Stock Valuation account |
|---|---|---|
| PO1 | Received 200 @10, sold 150; bill posted after shipment at @12 | The 150 x 2 variance on the units already sold settles in the account: **overstated by 300** |
| PO2 | Received 100 @20, sold 80; bill posted after reconciliation at @17 | The 80 x 3 variance on the units already sold settles in the account: **understated by 240** |
| **Net difference, as reported** | | **60** |

Two amounts in opposite directions collapse into a single harmless-looking number. **As reported it is 60; underneath it is 540.** And nobody did anything wrong in either transaction: the goods were received, the goods were shipped, the bills were posted when they arrived.

**The magnitude of the difference says nothing about the severity of what sits underneath it.** If the cause is never established and the next period is handled the same way, the adjustment grows. Gross margin derived from cost of goods sold loses its basis in the same movement.

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

### Two ledgers, and two different moments

One point underpins everything that follows. Odoo 19 removed the interim goods-received account that earlier versions used, and in 19 **inventory valuation and the general ledger are two separate ledgers.**

The valuation subledger is computed in real time from stock movements, but it **never writes an accounting entry itself**. The general ledger has exactly three entry points: **document posting** (vendor bills and customer invoices), **location valuation accounts** (when one end of a move has a valuation account configured), and **period-end closing**. Nothing else books an entry.

The two sides work to different moments:

- **Inbound.** Inventory value is recorded provisionally at the order price when the goods are received; the vendor bill then revalues it at the billed price and writes the general ledger at the same time.
- **Outbound.** Inventory value is fixed at the cost prevailing when the delivery is completed and is never rewritten afterwards, while the general ledger records the cost of goods sold **at invoicing**.

**In summary.** These defaults do not themselves create divergence. When each physical movement is followed promptly by the posting of its document, the moment the ledger is written and the moment the inventory value is fixed sit together, the report and the account reconcile under deterministic rules, and the timing differences in between explain themselves. **The divergence appears when those two moments are pulled apart.**

## 3. Where the differences actually come from

Persistent divergence comes from operations that do not match the sequence the mechanism assumes. None of the cases below is conspicuous in daily work, and most involve no irregular action at all.

### Cause one: invoicing anchored to the order, so invoicing and delivery fall out of step

When the invoicing policy is set to Ordered quantities, the invoice is driven by the ordered quantity and can be raised before the goods ship. The two actions come apart from that point.

Cost of goods sold is recognised **at invoicing**, at whatever the cost was at that moment; the inventory reduction happens **at delivery**, at whatever the cost is then. If the product's cost changed anywhere in between, the two figures do not agree, the difference stays in the inventory account permanently, and nothing heals it.

The costing method changes how this is exposed but does not remove it. Under average cost, **any receipt** inside the window revalues the cost, so the two must diverge. Under FIFO the two agree as long as the delivery consumes the same batch that was priced at invoicing — but if that batch is consumed by another order first, or its price is changed by a landed cost or a late bill, the same mismatch follows.

**The exposure is governed by the length of the window between invoicing and delivery, not by the costing method.**

### Cause two: operations that change stock without a document

Inventory adjustments, scrap, and opening stock imported as a count all change quantity and value directly. These locations carry no valuation account by default, so no accounting entry is produced: the inventory report moves and the general ledger does not.

Opening stock deserves particular attention. Inventory is imported as a count while the ledger is established by a separate manual entry, so the two figures need never have agreed in the first place, and every subsequent monthly adjustment is topping up that original gap.

Configuring a valuation account on these locations does produce entries, but then a single inventory adjustment made by a warehouse operator writes straight into the profit and loss statement with no approval step. Both positions carry a cost.

### Cause three: overriding the return write-back

The *Update quantities on SO/PO* column is ticked by default and hidden by default. Unticking it is a deliberate act: the goods move back, but the received or delivered quantity on the source document is untouched. The document continues to read as fully received or delivered against stock that is no longer there, and every invoicing decision built on that quantity is wrong from that point on. Because the column is hidden, the override is difficult to spot when the transaction is reviewed later.

### Cause four: cost corrections that arrive after the goods have shipped

The value of an outgoing move is fixed when the delivery is completed and is never rewritten. So when a corrected bill price or a landed cost is entered after the goods have been sold, the units still on hand are revalued and **the units already shipped never receive the new cost**. Under AVCO and FIFO no price variance entry is generated for them — the variance account belongs to standard costing — so the difference can only be absorbed by revaluing stock on hand, and for the shipped portion there is no stock left to revalue.

Where that difference comes to rest depends on how it was entered:

- **A corrected vendor bill price.** The variance attributable to the shipped units stays in the **inventory account** until period-end closing sweeps it into stock variation.
- **A landed cost.** Only the portion matching the quantity still on hand is capitalised into inventory. The remainder is not a residual awaiting close — it is **left in the expense account configured on the landed cost line** and becomes current-period expense immediately.

In both cases the total is correct and the classification is not: an amount that belongs in the cost of goods sold for those units is recorded somewhere else, and gross margin is distorted accordingly. This class of difference never shows up in a balance; it shows up in margin. The example in section 1 is of this kind.

### Cause five: a bill left in a cancelled state

Posting a bill recomputes the inventory cost layer. Resetting a posted bill to draft, cancelling it or deleting it removes the general ledger entry only and **does not recompute the cost layer in reverse**.

If the bill is then corrected and posted again, the revaluation runs a second time, the cost layer is recomputed at the new price and **overwrites** the old value, the ledger entry is rebuilt at the new price, and the two ledgers realign. That is the normal correction path; the act of cancelling is not itself the problem.

The problem is a bill **left** in the cancelled state. The ledger entry is gone while the cost layer still stands at the cost the cancelled bill established. If any of the stock has been sold, the cost-of-goods-sold credit raised by the customer invoice remains against the inventory account while the corresponding purchase debit no longer exists — **and the inventory account carries a credit balance.**

Reversing with a credit note behaves the same way: the ledger is squared, the cost layer takes no part in it. Unless a correct bill is posted afterwards, the cost layer will not return to the right value on its own and a manual inventory revaluation is required.

### One difference that looks like a gap but is not

When a vendor bill posts at a price different from the purchase order price, AVCO and FIFO treat the bill price as the true cost and revalue the stock move accordingly. Ledger and subledger are updated together and no divergence is created. Recognising this case matters as much as recognising the real ones — it is exactly the kind of self-resolving noise that conceals genuine exposure once everything is netted into a single adjustment.

Beyond these, allowing negative stock, the behaviour of the different costing methods, and manufacturing costs such as labour and depreciation each have their own mechanics and are not enumerated here.

**In summary.** A difference need not come from an irregular action. Most of the time it comes from **the ledger moment and the inventory moment being pulled apart** — in quantity, or in cost.

## 4. Checking your own system

Before commissioning any analysis, these steps establish the size and direction of the problem.

1. **Confirm the architecture.** Receive goods on the target instance and look at the journal items. **No entries** confirms Odoo 19 Perpetual (at invoicing), and everything in this guide applies. If a receipt does post an entry, the premise does not hold and the situation needs separate assessment.
2. **Establish the gap and its direction.** At a single cut-off date, compare the Inventory Valuation report total with the Stock Valuation account balance and record the signed difference. Direction matters as much as amount.
3. **Check for credit balances on inventory accounts, and for purchase bills sitting in draft or cancelled.** A credit balance on an asset account usually points to the latter (cause five).
4. **Check whether valuation modes are mixed.** Review the valuation setting on each product category. If some are Periodic and some Perpetual, they cannot share one valuation account and cannot be reconciled the same way.
5. **Strip out the timing differences.** At the same date, quantify what has been received but not billed and delivered but not invoiced. This is expected timing, not error. What remains after removing it is the real gap.
6. **Review the anchors, and measure the window.** Check the sales invoicing policy and the purchase bill control on your highest-volume goods. Ordered quantities on a fast-moving stocked product is the most common structural cause. Then count, for those products, how many order lines were invoiced before they shipped and the average number of days in between — the longer the window, the larger the exposure from cause one.
7. **Decompose to causes and test the sum.** Quantify received-not-billed, delivered-not-invoiced, over-billing, posted bills that were cancelled or deleted, the cost divergence created when invoicing and delivery fall out of step, and the variance a late bill leaves on units already sold. Their **algebraic sum should equal the reported difference.** If it does not, a further cause is present and has not yet been identified.

These steps establish the size and rough location of the difference. Tracing it to individual documents requires analysis against your own chart of accounts and process design.

> **Important: structural errors must be reconciled on a posted-quantity basis before period-end closing is run.**

## 5. The existing answers, and where they stop

Responses to a valuation gap fall into several layers. None of them seals the source.

- **Manual adjustment** is the most common. It balances the total for the close and leaves no record of what the difference consisted of, hiding gross exposure behind a net figure.

- **Period-end closing** is what many businesses actually rely on, and its nature is worth stating plainly: **closing is an eraser, not a corrector.** It sweeps every divergence between valuation and the ledger into stock variation without distinguishing between them. Returns with no credit note, invoiced quantities that do not match what shipped, posted bills that were cancelled, cost divergence from invoicing out of step with delivery — once swept, each is legitimised as an expense and the trail is permanently lost. The total does balance. The cause becomes unrecoverable.

- **The native lock settings** (*Lock Confirmed Sales*, *Lock Confirmed Orders*) are off by default and, when enabled, prevent confirmed documents from being edited. They constrain whether a document **can be changed**, not whether the two moments **can be pulled apart**. With both enabled, goods can still be invoiced before they ship, a return can still be confirmed with the write-back unticked, a posted bill can still be reset to draft, and a late bill still cannot revalue what has already shipped. **None of the causes above is the result of someone editing a confirmed order.**

- **Detective tooling** — audit trails, lock dates, change logs — records who did what, after the event. It is genuinely useful for accountability, but it does not stop the transaction and does not reach the operational layer where the divergence is created: order lines, pickings, returns and reconciliation.

- **Corrective tooling** — the various valuation adjustment modules — lists the products where report and account disagree and posts an entry to force them into line. The engineering is sound and it helps at close, but it treats the symptom. The next invoice raised ahead of its delivery, the next return with the write-back overridden, reopens the gap and the module runs again next month.

What none of these layers provides is **prevention**: constraining the operation to the sequence the mechanism allows at the moment it is attempted, and surfacing the exposure on the document rather than as an aggregate at month end.

## 6. How SuiteState approaches it

SuiteState runs its own wholesale operation on Odoo and met this problem from the inside: a Stock Valuation account that would not tie out to the Inventory Valuation report, traced back through receipts, returns and late bills to a small set of operations that did not match the sequence the mechanism assumes — with a net difference far smaller than the gross exposure on either side of it.

Two things came out of that work, and they are deliberately different in kind.

**A diagnostic service** — a read-only analysis that decomposes an existing gap into its causes (received not billed, delivered not invoiced, returns without a credit note, price variance that has already self-resolved) until the residual is zero and every component traces to specific documents. It is built against the client's own environment and cannot be a generic tool: the decomposition depends on the client's chart of accounts, product categories, process design, and where their accounting emphasis sits.

**A prevention product** — [Data Guard](https://apps.odoo.com/apps/modules/19.0/suite_data_guard/) holds warehouse operations to the sequence the mechanism allows and surfaces the signed difference between moved quantity and posted quantity on the order line itself: positive for over-invoicing, negative for stock that moved and was never billed. The exposure stops being an aggregate question on a report and becomes a specific item on a close checklist, so the gap does not quietly reopen once it has been closed.

If your inventory value and your general ledger have stopped agreeing, the first step is not another adjusting entry. It is establishing what the difference is actually made of.
