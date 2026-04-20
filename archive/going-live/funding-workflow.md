This is a critical "Revenue vs. Risk" module. By allowing offline payments but charging a service fee, you transform Zeyaroo from a simple payment processor into a **Workflow Orchestrator.**

As a CPO, I recommend the **"Pay-to-Unlock"** model for the service fee. Even if the client pays the creator $1,000 in cash, they pay Zeyaroo a small fee (e.g., $5) online to move the project state forward and use the "Project Brief" or "Media Delivery" features.

Here is the professional workflow model and the corresponding tasks.

---

### The Workflow: "Request for Funding"

1.  **The Trigger:** Creator clicks **"Request Funding"** on a specific Chapter.
2.  **The Context:** Creator chooses allowed methods for *this* chapter:
    *   `[ ] Online (Zeyaroo Escrow)`
    *   `[ ] Offline (Direct to Creator)`
3.  **The Communication:** Client receives a WhatsApp/Email: *"Chapter [Name] is ready to begin. Please select a payment method to proceed."*
4.  **The Client Choice:**
    *   **Path A (Online):** Client pays via Dodo. Milestone moves to `Funded`.
    *   **Path B (Offline):** Client sees a "Payment Instructions" note from the creator.
5.  **The Verification (Offline Path):**
    *   Creator decides: **"Self-Verify"** (I'll just click a button when I have the cash) or **"Client Receipt"** (Client must upload a screenshot of the bank transfer).
6.  **The State Change:** Once verified, the Chapter moves to `Funded (External)`.

---

### Feature: Flexible Chapter Funding & Liability Shield

#### **Task 1: Update Chapter Model for Payment Strategies**
- **suggested-asignee**: Backend
- **priority**: High
- **consideration**: We need to track the *intended* methods and the *actual* method used.
- **description**: Extend the Chapter (Milestone) model to support multiple payment avenues and tracking.
- **requirements**:
    1. Add `AllowedMethods` (bitmask or JSON: `zeyaroo_escrow`, `offline`).
    2. Add `OfflineInstructions` (text) - e.g., "Pay to Commercial Bank Acc: 12345".
    3. Add `RequiresReceipt` (boolean).
    4. Add `ActualMethodUsed` (string) and `ServiceFeeCents` (int64).
- **acceptance-criteria**:
    - The database can store different payment configurations for each chapter in a project.

---

#### **Task 2: "Request Funding" Creator UI**
- **suggested-asignee**: Frontend
- **priority**: High
- **consideration**: Use the **Workspace Style** (12px radius) for this internal tool.
- **description**: The modal/interface where a creator triggers a payment request.
- **requirements**:
    1. A button "Request Funding" on each Chapter card.
    2. A configuration modal:
        *   Toggle: Enable Online Payments.
        *   Toggle: Enable Offline Payments.
        *   Text Area: Offline instructions (saved as default).
        *   Checkbox: Require client to upload receipt.
- **acceptance-criteria**:
    - Clicking "Send Request" updates the chapter status to `awaiting_funds` and triggers notifications.

---

#### **Task 3: Client Payment Selection Page (Magic Link)**
- **suggested-asignee**: Frontend
- **priority**: High
- **consideration**: This page must contain the **Liability Disclaimer** prominently.
- **description**: The public-facing view where a client chooses how to pay.
- **requirements**:
    1. Display two large cards: "Secure Online Checkout" and "Direct Payment."
    2. **Liability Shield:** Under "Direct Payment," display: *"Note: This transaction happens outside Zeyaroo. Zeyaroo does not hold these funds in escrow and is not liable for disputes regarding this payment."*
    3. If Offline is chosen, show the creator's instructions.
- **acceptance-criteria**:
    - Client can select a method. Selecting "Online" opens Dodo. Selecting "Offline" moves to the next step (Instructions/Receipt).

---

#### **Task 4: Service Fee Billing Logic (The "Zeyaroo Tax")**
- **suggested-asignee**: Senior Backend
- **priority**: High
- **consideration**: If the client pays the creator offline, Zeyaroo still needs its fee. We can charge this to the client's card as a "Platform Service Fee."
- **description**: Implement the logic to calculate and collect a fixed or percentage-based fee for offline-marked chapters.
- **requirements**:
    1. Logic: If `ActualMethod == 'offline'`, generate a Dodo Checkout for *only* the `ServiceFeeCents`.
    2. Prevent the chapter from moving to `funded` until the Service Fee is paid.
- **acceptance-criteria**:
    - Even for an offline project, Zeyaroo collects its $2-$5 "Management Fee" before the project continues.

---

#### **Task 5: Offline Receipt Upload & Storage**
- **suggested-asignee**: Fullstack
- **priority**: Medium
- **consideration**: Use the pre-signed upload logic from Topic 7.
- **description**: Provide a way for clients to prove they paid the creator directly.
- **requirements**:
    1. If `RequiresReceipt` is true, show a file uploader on the payment page.
    2. Store the image in B2 under `projects/{id}/receipts/{chapter_id}.jpg`.
    3. Notify creator: "Client has uploaded a payment receipt. Please verify."
- **acceptance-criteria**:
    - Creator can view the uploaded receipt image in their dashboard.

---

#### **Task 6: Creator "Acknowledge Payment" Action**
- **suggested-asignee**: Junior Fullstack
- **priority**: High
- **consideration**: This is the manual override for the automated escrow logic.
- **description**: Allow creators to manually mark a chapter as paid.
- **requirements**:
    1. A button "Mark as Paid" visible only on chapters in `awaiting_funds` status.
    2. If clicked, show a confirmation: "I confirm I have received the funds for this chapter."
- **acceptance-criteria**:
    - Clicking confirm moves the chapter to `funded` and unlocks the next project phase.

---

#### **Task 7: The "Zero Liability" Legal Modal**
- **suggested-asignee**: Frontend
- **priority**: Medium
- **consideration**: Must be non-intrusive but legally clear. 
- **description**: An interstitial or "I Agree" checkbox when choosing the offline path.
- **requirements**:
    1. Text: "I understand that by paying outside the platform, I waive my right to Zeyaroo's Escrow Protection and Dispute Resolution services."
    2. Store the agreement timestamp in the `EscrowTransaction` record.
- **acceptance-criteria**:
    - The client must acknowledge the lack of protection before they can view the offline instructions.

---

#### **Task 8: Automated "Service Fee" Invoice**
- **suggested-asignee**: Backend
- **priority**: Low
- **consideration**: Useful for accounting/taxes.
- **description**: Generate a mini-invoice for the small service fee paid to Zeyaroo.
- **requirements**:
    1. Create a `models.Invoice` for the platform fee.
    2. Email it to the client after a successful "Offline Choice" fee payment.
- **acceptance-criteria**:
    - Client receives an email receipt for the $5 platform fee.

---

#### **Task 9: Payout Dashboard: Online vs Offline Stats**
- **suggested-asignee**: Frontend
- **priority**: Low
- **consideration**: Help the creator see their "Total Cash Flow" vs "Zeyaroo Wallet."
- **description**: Split the revenue dashboard to show the difference in payment streams.
- **requirements**:
    1. Add a chart or list showing "Direct Revenue" (Offline) vs "Platform Revenue" (Online).
- **acceptance-criteria**:
    - Creator can see that they made $5,000 this month, even if $4,000 of it was paid in cash.

---

#### **Task 10: Notification: "Payment Acknowledged"**
- **suggested-asignee**: Junior
- **priority**: Medium
- **consideration**: This provides the client with a "Digital Receipt."
- **description**: Send a confirmation when the creator marks an offline payment as received.
- **requirements**:
    1. Trigger Email/WhatsApp: "[Creator Name] has confirmed receipt of your payment for Chapter: [Name]. The project is moving forward."
- **acceptance-criteria**:
    - Client feels secure that the creator has officially updated the system.

---

### CPO Strategy Note:
The **Service Fee** is your defense against "Platform Leakage." If you make it small (e.g., $1-$5), users won't mind paying it for the convenience of the "Project Brief" and "Organized Gallery" features. If you make it $0, you are providing a free service that costs you money in server time and support. **Always charge for the workflow.**