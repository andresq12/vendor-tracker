# Pokémon TCG Vendor Tracker — Development Handoff

## Overview

A mobile-first transaction tracker for a 4-person Pokémon TCG vending group. The group sells cards at shows/events individually but also maintains a **shared community card pool** (called "Eclipse") that all 4 vendors co-own equally. Transactions are messy — a single customer interaction can involve multiple vendors' cards, trade-ins, cash flowing both directions, and personal money used for change. This app logs all of that and tells each vendor exactly what they're owed.

## Vendors

- **Vendor B, Vendor D, Vendor R, Vendor A (Andres)** — 4 individual sellers
- **Pool (Eclipse)** — a collectively owned card inventory; all 4 vendors have equal 25% equity

---

## Transaction Types

### 1. Straight Sale
Customer pays cash, gets cards from one or more vendors and/or the pool.

**Example:** Customer buys $25 of cards — $5 from Vendor B, $5 from Vendor D, $10 from Vendor R, $5 from Pool. Customer pays $40 cash.

### 2. Sale with Change
Same as above but customer overpays and needs change. Someone has to front the change from their own wallet or the pool's cash.

**Example:** Same $25 sale, customer pays $40. Vendor D pulls $10 from their wallet, Pool contributes $5 for the $15 change. Vendor D is owed their $5 card sale + $10 reimbursement = $15 total.

### 3. Trade-In + Cash
Customer trades in cards AND pays some cash. Trade-in cards are assigned to whoever keeps them.

### 4. Trade-In + Cashback
Customer trades in cards worth more than what they want. The group pays the customer cash for the difference. Someone fronts that cash.

### 5. Pure Trade
Cards for cards, no cash. Still need to track who gave up cards and who received trade-in cards.

### 6. Vendor Buyout (most important for trades)
When a customer does a big trade-in and the trade-in cards need to go somewhere, one vendor can "buy out" — they take ALL the trade-in cards and pay the other vendors cash for their card sales out of their own pocket. This avoids messy pool accounting.

---

## Core Data Model

### Transaction Object

```json
{
  "id": 1234567890,
  "ts": "2026-08-10T14:30:00.000Z",
  "sold": {
    "B": 5,
    "D": 18,
    "R": 93,
    "A": 12,
    "pool": 49
  },
  "tradeIn": {
    "B": 0,
    "D": 0,
    "R": 0,
    "A": 0,
    "pool": 177
  },
  "cashIn": 40,
  "cashOut": 15,
  "fronted": {
    "B": 0,
    "D": 10,
    "R": 0,
    "A": 0,
    "pool": 5
  },
  "buyout": {
    "vendor": "R",
    "total": 177
  },
  "note": "Big Charizard trade"
}
```

### Field Definitions

| Field | Description |
|-------|-------------|
| `sold` | Dollar value of cards each vendor/pool sold to the customer |
| `tradeIn` | Dollar value of trade-in cards assigned to each vendor/pool |
| `cashIn` | Total cash the customer paid |
| `cashOut` | Total cash given to the customer (change or trade surplus) |
| `fronted` | Who provided the cash given to the customer, from their own wallet or pool funds |
| `buyout` | If present, indicates one vendor took all trade-in cards (vendor buyout mode) |
| `note` | Optional freetext note |

---

## Accounting Logic

### Per-Transaction: Who Is Owed What

For each vendor/pool entity in a single transaction:

```
cash_owed = cards_sold + cash_fronted
cards_received = trade_in_value_assigned_to_them
net_cash = cash_owed - cards_received
```

- **Positive net** → they are owed cash from the cash pile
- **Negative net** → they owe cash back (took more in trade-in cards than they sold)
- **Zero** → square

### Validation Rules

The app enforces these before allowing a transaction to be logged:

1. **Something must happen:** total sold > 0 OR total trade-in > 0
2. **Cash must balance:** `cashIn - cashOut` must equal `totalSold - totalTradeIn`
   - This ensures the customer paid the right amount for the net value they received
3. **Fronted cash must be accounted for:** if `cashOut > 0`, the sum of all `fronted` values must equal `cashOut`
   - Every dollar given to the customer must be sourced from someone's wallet or the pool

### Settlement: Across All Transactions

The Settle Up view aggregates all transactions and calculates:

```
For each entity (vendor or pool):
  total_sold = sum of all their card sales across transactions
  total_fronted = sum of all cash they fronted across transactions
  total_trade_in = sum of all trade-in cards they received across transactions
  
  net_cash_owed = total_sold + total_fronted - total_trade_in
```

**Cash on hand** = total cashIn across all transactions - total cashOut across all transactions

**Payout instructions** are generated:
1. Anyone with negative net puts cash into the pile first
2. Then everyone with positive net gets paid from the pile
3. Verify: cash on hand + money returned = total owed out

---

## Trade-In Modes

### Split Manually
The user assigns trade-in card values to individual vendors and/or the pool field by field. Used when different people are keeping different cards from a trade-in.

### Vendor Buyout
One vendor takes ALL the trade-in cards. The app:
1. Takes a single total trade-in value
2. Lets user pick which vendor is buying out
3. Assigns entire trade-in value to that vendor
4. Shows a buyout summary: how much the buying vendor pays each other vendor (their card sale amounts)

**Buyout math example:**
- Cards sold: B=$5, D=$18, R=$93, A=$12, Pool=$49 (total=$177)
- Trade-in cards: $177 total, Vendor R buys out
- All $177 assigned to R in `tradeIn`
- R's net: $93 sold + $0 fronted - $177 trade-in = **-$84** (R owes $84)
- That $84 goes to: B=$5, D=$18, A=$12, Pool=$49
- R walks away with $177 in physical cards, having paid $84 out of pocket, keeping $93 as their own card's value

---

## Pool Equity Logic (Important Context)

The pool is co-owned equally (25% each). This affects how trade-ins work:

### When Pool Absorbs Trade-In Cards
If a trade happens and the pool takes in trade-in cards, the pool is generally **made whole** — it gave up cards + cash, but received trade-in cards of equivalent value. The pool's total value (cash + card inventory) doesn't change, so nobody's 25% equity is affected. Vendors get paid their full card sale amounts fairly.

### When Equity-Adjustment Would Matter
Only in a hypothetical scenario where the pool pays out cash with NOTHING coming back (like giving away free cards as promotions). In normal sales and trades, trade-in cards always flow back in, so the pool self-corrects. **Equity-adjusted payouts were considered and intentionally excluded** — they solve a problem that doesn't exist in practice for this group.

### Why Buyouts Are Preferred for Big Trades
Even though pool absorption is technically fair when trade-in cards come back, buyouts are simpler:
- Immediate settlement, no carrying debt
- No confusion about pool equity
- One vendor takes the cards, pays cash, everyone's done
- The app shows exactly what the buying vendor owes each person

---

## Current Features (HTML Prototype)

### Log Tab
- Cards sold per vendor/pool with running total
- Trade-in mode toggle: Split Manually vs Vendor Buyout
- Buyout: total value input + vendor picker with live summary
- Cash in/out fields with auto-calculated balance hint
- "Who fronted the cash" section (appears only when cash goes to customer)
- Live payout preview showing cash owed + cards received per entity
- Validation: blocks submission until everything balances

### Settle Up Tab
- Cash on hand (total collected - total given back)
- Net position per vendor/pool (sold + fronted - trade-ins)
- Step-by-step payout instructions
- Balance verification
- Export to CSV
- Clear all data option

### History Tab
- Chronological transaction list
- Tap to expand with full breakdown per vendor
- Shows transaction type labels (Sale, Trade + Sale, Buyout, Trade-in)
- Delete individual transactions
- Export to CSV

### Export
- CSV with all fields per transaction
- Uses Web Share API on iOS (native share sheet → Files, AirDrop, email, etc.)
- Falls back to blob download on desktop/Android

---

## Technical Notes

### Current Stack
- Single HTML file with vanilla JS, CSS, localStorage
- No framework, no build step
- Mobile-first, designed for iPhone home screen (PWA meta tags)

### Architecture Decisions in Prototype
- **DOM inputs built once, never destroyed** — the original React-style "rerender everything on input" caused crashes on mobile because inputs lost focus mid-keystroke. Current approach: form inputs are static DOM elements, only computed displays (totals, previews, validation messages) update on input events.
- **localStorage for persistence** — data lives on one device only. A real app would need shared state (see below).

### Recommended Stack for Real App
- **React Native** or **Expo** for cross-platform mobile
- **Supabase** or **Firebase** for shared real-time database (all 4 vendors need to see the same data)
- **Auth** — each vendor logs in, transactions tagged to who logged them
- **Offline-first** — shows/events may have bad cell signal; queue transactions locally and sync when connected

---

## Features to Build Next

### High Priority
- [ ] Multi-device sync (all 4 vendors see same data)
- [ ] Vendor authentication (who's logging what)
- [ ] Session/event grouping ("Bay Area Card Show - Aug 10" as a container for that day's transactions)
- [ ] End-of-day settlement report per session
- [ ] Pool cash balance tracking (how much cash the pool actually has on hand, separate from card inventory)

### Medium Priority
- [ ] Transaction edit (not just delete and re-enter)
- [ ] Photo attachment for trade-in cards (snap a pic of what came in)
- [ ] Running pool inventory tracker (cards the pool currently holds)
- [ ] Notification when someone logs a transaction
- [ ] Payout confirmation (mark when someone has actually been paid)

### Nice to Have
- [ ] Historical analytics (revenue per vendor over time, trade-in volume trends)
- [ ] Card price lookup integration (TCGPlayer API for real-time values)
- [ ] Receipt generation for customers
- [ ] Tax-relevant export (total revenue per vendor for tax reporting)
- [ ] Buyout recommendation flag ("Vendor R sold 53% of this trade — consider a buyout")

---

## Example Scenarios for Testing

### Scenario 1: Simple Sale with Change
- Customer buys: B=$5, D=$5, R=$10, Pool=$5 (total $25)
- Customer pays $40 cash
- Change: $15 → D fronts $10, Pool fronts $5
- **Expected:** B owed $5, D owed $15, R owed $10, Pool owed $10

### Scenario 2: Pure Trade, Pool Absorbs
- Customer trades in $177 in cards (all go to pool)
- Customer gets: B=$5, D=$18, R=$93, A=$12, Pool=$49 (total $177)
- No cash either direction
- **Expected:** Pool's net = $49 sold - $177 trade-in = -$128 (owes $128). Other vendors get their sale amounts from pool.

### Scenario 3: Vendor Buyout
- Same as Scenario 2 but Vendor R buys out
- All $177 trade-in assigned to R
- **Expected:** R's net = $93 - $177 = -$84. R pays B=$5, D=$18, A=$12, Pool=$49.

### Scenario 4: Trade + Cash + Cashback
- Customer trades in cards at 85% of $380 = $323
- Customer gets $291 in vendor cards + $32 cash back
- Pool fronts the $32
- Pool absorbs $323 in trade-in cards
- Cards sold: Pool=$231, Andres=$60
- **Expected:** Pool gets reimbursed $32 + owed $231 for cards - $323 in trade-ins = -$60 net. Andres owed $60.

---

## File Reference
- `vendor-tracker.html` — current working prototype (standalone, runs on mobile browser)
