# Campus Wallet

## Overview

Every user on the platform has a digital wallet. Students and staff use their wallet balance to purchase items from the campus shop. Cashiers load funds onto accounts by accepting cash and running a top-up. All balance changes are processed server-side through Supabase RPCs — the app never writes a balance figure directly to the database.

## Roles

| Role | Wallet capabilities |
|---|---|
| Student | View own balance and transaction history, spend at campus shop |
| Staff | View own balance and transaction history, spend at campus shop |
| Cashier | Top up any account, view all transactions across all accounts |
| Merchant staff | Receive payments at POS (via shop purchase flow) |
| Admin | Same as cashier; no personal wallet (admin accounts are operational) |

## Architecture

```
User action
    ↓
WalletRepository (requires online)
    ↓  RPC call
Supabase (wallet_deposit / wallet_purchase / wallet_refund)
    ↓  updates wallet_accounts + inserts wallet_transactions
WalletRepository.refreshCache()
    ↓  reads fresh balance + transactions
SQLite (wallet_accounts, wallet_transactions)
    ↓
UI displays from cache
```

SQLite is a **display cache only**. The balance shown in the app is the last value pulled from Supabase after a successful RPC. An offline device shows the last known balance with an "offline" indicator.

## Wallet Screen (`/wallet/me`)

Available to all authenticated users. Shows:
- Current balance (large, coloured card)
- Full transaction history (newest first)
- Each transaction row: type icon, type label, merchant (if any), date, amount (green credit / red debit), running balance after

An offline banner appears when the last balance fetch failed or connectivity is absent.

## Cashier Top-Up (`/cashier/top-up`)

Accessible to cashier and admin roles only.

**Step 1 — Find account**

The cashier searches by:
- Name (partial match, case-insensitive)
- Student number (exact)
- Staff number (exact)

Results are fetched from Supabase `profiles` table in real time.

**Step 2 — Top up**

After selecting an account, the cashier sees:
- Profile name, email, staff number, role
- Current balance
- Deposit amount field

Tapping **Add to Wallet** calls `wallet_deposit` RPC with the profile ID and amount. On success:
- Balance is refreshed from Supabase
- A receipt card is displayed: name, ID, amount deposited, new balance, timestamp
- The transaction appears immediately in the transaction history list

## All Transactions (`/cashier/transactions`)

A cashier/admin screen showing all wallet transactions across all users, ordered newest first.

## Supabase RPCs

All RPCs use `SECURITY DEFINER` and check `auth.uid()` before executing.

### `wallet_deposit(p_profile_id, p_amount, p_notes)`
Adds funds to a wallet. Enforced for cashier/admin roles via RLS. Creates a `deposit` transaction row.

### `wallet_purchase(p_profile_id, p_amount, p_merchant, p_notes)`
Deducts funds for a purchase. Raises an exception if the balance is insufficient. Called indirectly via `create_shop_purchase` (which handles atomicity). Creates a `purchase` transaction row.

### `wallet_refund(p_profile_id, p_amount, p_reference, p_notes)`
Credits funds back for a refund. Called by `refund_purchase`. Creates a `refund` transaction row.

## SQLite Schema

```sql
wallet_accounts (
  id TEXT PRIMARY KEY,
  profile_id TEXT,
  student_id TEXT,
  balance TEXT NOT NULL DEFAULT '0.00',
  updated_at TEXT NOT NULL
)

wallet_transactions (
  id TEXT PRIMARY KEY,
  wallet_id TEXT NOT NULL,
  profile_id TEXT,
  student_id TEXT,
  type TEXT NOT NULL,          -- 'deposit' | 'purchase' | 'refund' | 'reversal' | 'adjustment'
  amount TEXT NOT NULL,
  balance_after TEXT NOT NULL,
  merchant TEXT,
  status TEXT NOT NULL DEFAULT 'completed',
  reference TEXT,
  notes TEXT,
  created_at TEXT NOT NULL
)
```

## Transaction Types

| Type | Direction | Description |
|---|---|---|
| `deposit` | Credit | Cash loaded by cashier |
| `purchase` | Debit | Campus shop payment |
| `refund` | Credit | Purchase reversed and credited back |
| `reversal` | Credit | Manual correction |
| `adjustment` | Either | Admin balance correction |

## Campus Shop Integration

When a student buys an item, `create_shop_purchase` RPC runs atomically:
1. Checks `wallet_accounts.balance >= item.price * quantity`
2. Calls `wallet_purchase` to debit the balance
3. Decrements item stock (if tracked)
4. Creates a `purchases` row
5. Generates a unique reference number (`PUR-YYYYMMDD-XXXXX`) and a 6-char alphanumeric short code
6. Returns the purchase object including QR payload and short code

The student can then show the QR or read out the short code at the counter for collection.

## Redemption & Refund

### Redemption
Merchant staff or cashier scans the purchase QR or enters the short code at `/shop/redeem`. The `redeem_purchase` RPC marks the purchase as `COLLECTED` and `is_redeemed = true`. The redemption is single-use — a second scan raises "already redeemed".

### Refund
At `/shop/refund`, the cashier or admin enters the purchase reference. The `refund_purchase` RPC:
1. Verifies the purchase is refundable (not already refunded/cancelled)
2. Credits the buyer's wallet via `wallet_refund`
3. Restores item stock if applicable
4. Marks the purchase as `REFUNDED`
