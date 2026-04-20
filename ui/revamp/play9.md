This is **Deep Dive 9: The Billing & Revenue Dashboard**.

This page turns the "business side" of being a creator into a visual achievement. It needs to clearly differentiate between money the creator **has earned** (available for payout) and money **Zeyaroo is holding safely** (escrow). It must remain clean and avoid looking like a cluttered bank statement.

---

### **1. Wireframe Layout (Desktop)**
A **Top-Down Balanced Layout** focusing on liquidity and status.

*   **The Financial Summary (Top Row):**
    *   Three minimalist high-impact blocks:
        1.  **Total Value Secured:** Sum of all active milestones in Escrow.
        2.  **Available for Payout:** Money from completed milestones ready to be sent to the bank.
        3.  **Net Life-time Earnings:** A cumulative "Success Metric" in `font-serif`.
*   **The Zeyaroo Plan Card (Middle Left - 4 cols):**
    *   Current Tier display (Pro/Studio).
    *   Next billing date.
    *   **Storage Pulse:** A small horizontal meter showing how much of their 100GB/500GB is used.
    *   Action: [Change Plan] button.
*   **Payout Method (Middle Right - 8 cols):**
    *   Status of the Dodo Payments or LKR Bank verification.
    *   "Last payout: $1,200 on Oct 12."
    *   Action: [Request Payout] (if balance > 0).
*   **Transaction Ledger (Bottom Row - 12 cols):**
    *   A clean list of recent inflows, payouts, and platform fees.
    *   Each row uses a tiny project thumbnail to provide visual context.

---

### **2. Functional UX & Interactions**
*   **The "Fee Transparency" Toggle:** Hovering over any "Net Amount" in the ledger reveals a breakdown: *(Gross: $100.00 - Platform Fee: $2.00 - Tax: $0.00 = Net: $98.00)*.
*   **Payout Animation:** When a creator clicks "Request Payout," the "Available" number should count down to zero with a smooth "sliding" animation as the money "moves" to the processing state.
*   **Currency Context:** If the creator has both LKR and USD projects, the summary blocks should show a toggle or a split view to avoid mixing currencies.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Summary Numbers**
```tailwind
/* Label */
font-sans text-[10px] uppercase tracking-[0.2em] text-text-muted mb-2
/* Value */
font-serif text-5xl text-text-primary tracking-tighter
```

#### **The Transaction Row**
```tailwind
/* Container */
flex items-center justify-between py-4 border-b border-subtle hover:bg-surface-800/20 px-2 transition-colors

/* Type Icon (Inflow/Outflow) */
w-10 h-10 rounded-full flex items-center justify-center 
bg-emerald-50 text-emerald-600 /* for Inflow */
bg-gray-50 text-gray-600 /* for Payout */
```

#### **Plan Status Badge**
```tailwind
/* Studio Tier (Gold Style) */
bg-amber-50 text-amber-700 border border-amber-200 
text-[10px] font-bold px-2 py-0.5 rounded-md
```

---

### **4. Edge Cases**
*   **Pending Verification:** If Dodo Payments needs more KYC (Identity) info, the Payout card turns amber with a "Verification Required" link.
*   **Zero Earnings:** The ledger shows a beautiful empty state: *"Your first milestone payment is a few clicks away."*
*   **LKR/USD Split:** If a creator operates in two currencies, the summary cards become "Tabs" allowing them to switch between their USD wallet and LKR wallet.

---

### **5. GitHub Issue: Implement Billing & Ledger API (Backend)**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Payout requests must be immutable once created. Use a state machine for payouts (`pending`, `processing`, `paid`, `failed`).
- **description**: Build the financial backbone for the creator's billing dashboard.
- **requirements**:
    1. Endpoint: `GET /api/v1/billing/summary` (Calculates life-time, available, and escrow totals).
    2. Endpoint: `GET /api/v1/billing/transactions` (Paginated list of all movements).
    3. Endpoint: `POST /api/v1/billing/payouts` (Triggers the Dodo/Wise payout flow).
    4. Implement "Currency Isolation" logic so USD and LKR totals are never added together.
- **acceptance-criteria**:
    - API correctly calculates fees based on the user's `SubscriptionTier`.
    - Payout requests are rejected if `AvailableBalance < RequestAmount`.

---

### **6. GitHub Issue: Build Creator Revenue Dashboard UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `Intl.NumberFormat` to handle currency symbols correctly based on the creator's setting.
- **description**: Build the visual revenue dashboard using the high-end "Studio" style guide.
- **requirements**:
    1. Implement the `FinancialSummary` header blocks.
    2. Build the `TransactionLedger` with "Fee Breakdown" tooltips.
    3. Create the `SubscriptionCard` showing storage usage and current tier.
    4. Add the "Payout Request" modal flow.
- **acceptance-criteria**:
    - Numbers animate (count up) on initial load.
    - Transaction list includes project thumbnails for easy recognition.
    - Payout button is disabled if the balance is $0.

---

### **7. GitHub Issue: Subscription Management & Upgrade Flow**

- **suggested-assignee**: Fullstack
- **priority**: Medium
- **consideration**: Moving from Pro to Studio should be prorated via Dodo Payments.
- **description**: UI and logic for creators to manage their Zeyaroo monthly fee.
- **requirements**:
    1. Implement the "Pricing Tiers" selection modal.
    2. Integration with Dodo's Subscription Checkout.
    3. Add logic to show "Overage" warnings if a user downgrades their tier but has too much data.
- **acceptance-criteria**:
    - Clicking "Upgrade" leads to a successful Dodo checkout.
    - Tier status updates in the dashboard immediately after the webhook hits.

---

**Next Deep Dive 10: Marketplace & Discovery (The Creator Search)**