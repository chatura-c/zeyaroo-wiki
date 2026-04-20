This is **Deep Dive 8: The Inquiry & Lead Manager (CRM)**.

The Inquiry Manager is the "Front Desk" of the studio. While traditional CRMs are cluttered with tables, Zeyaroo’s manager must feel like an **organized guest list for an exclusive event**. It bridges the gap between a random DM/Email and a formal Project.

---

### **1. Wireframe Layout (Desktop)**
A **Pipeline-Centric View** with a Focus on Velocity.

*   **The Command Header (Top):**
    *   Stat Summary: "14 Active Leads • 5 Proposals Sent • Avg. Response: 2h."
    *   View Switcher: [Pipeline View] | [Table View].
    *   Search Bar with "Smart Filters" (e.g., `budget > 5000`, `date:this_month`).
*   **The Studio Pipeline (Center Stage):**
    *   Horizontal columns representing the "Lead Temperature":
        1.  **New (Inbox):** Fresh inquiries from the "Link in Bio" or public profile.
        2.  **Qualifying:** Creator is chatting with the client or checking availability.
        3.  **Proposal Sent:** Waiting for the client to view or fund the first milestone.
        4.  **Booking:** Milestones are being funded.
*   **The Lead Card (Kanban Item):**
    *   Client Name in `font-serif`.
    *   Event Date & Location tag.
    *   **Activity Pulse:** A tiny glowing dot if the client just viewed their proposal.
*   **The Details Drawer (Right Slide-over):**
    *   Triggered by clicking a card.
    *   Contains the full message, project details, and a "Quick Response" text area.

---

### **2. Functional UX & Interactions**
*   **Drag-to-Advance:** Move a lead from "New" to "Qualifying" by dragging the card. Dragging a lead to "Proposal Sent" automatically opens the **Proposal Builder** (Deep Dive 7).
*   **The "Heat" Indicator:** Cards for leads that haven't been replied to in 24 hours develop a subtle amber border to signal urgency.
*   **One-Tap Archive:** Swipe a card left (on mobile) or click a "dismiss" icon to archive cold leads.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Pipeline Column**
```tailwind
/* High-key White Style */
w-80 flex-shrink-0 bg-surface-800/40 rounded-3xl p-4 border border-subtle
/* Column Title */
font-serif text-xl text-text-primary mb-4 px-2
```

#### **The Lead Card**
```tailwind
/* Container */
bg-white p-5 rounded-2xl border border-gray-100 shadow-sm 
hover:shadow-md hover:border-accent-primary/20 transition-all cursor-grab

/* Lead Name */
font-serif text-lg leading-tight text-gray-900

/* Tags */
inline-flex items-center px-2 py-0.5 rounded-md bg-gray-50 
text-[10px] uppercase font-bold tracking-tighter text-gray-500
```

#### **Activity Pulse Animation**
```tailwind
/* Pulsing Green Dot */
w-2 h-2 rounded-full bg-emerald-400 animate-pulse ring-4 ring-emerald-400/20
```

---

### **4. Edge Cases**
*   **The "Double Inquiry":** If a client sends two inquiries using the same email, the system should "Stack" them under one lead card with a "2 Inquiries" badge.
*   **Empty Inbox:** Show a "Luxury Zero State" (Topic 8): *"Your inbox is a blank canvas. Share your Link in Bio to start receiving inquiries."*
*   **Lead without a Date:** Many clients inquire before they have a date. The UI should display "TBD" (To Be Determined) in an elegant, italicized serif.

---

### **5. GitHub Issue: Implement Inquiry Pipeline Backend Logic**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Use a `Status` enum for leads to ensure strict state transitions (e.g., you can't go from 'Archived' to 'Booking' without a proposal).
- **description**: Build the API to support a Kanban-style inquiry management system.
- **requirements**:
    1. Create `models.InquiryPipeline`: Link inquiries to a state (`new`, `qualifying`, `proposal_sent`, `booked`, `archived`).
    2. Endpoint: `GET /api/v1/inquiries/pipeline` (Return grouped by status).
    3. Endpoint: `PATCH /api/v1/inquiries/:id/status` (Update status for drag-and-drop).
    4. Implement "Lead Deduplication" logic based on email.
- **acceptance-criteria**:
    - Backend returns inquiries categorized into columns.
    - Status updates are reflected immediately in the database.

---

### **6. GitHub Issue: Build Kanban Pipeline UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `dnd-kit` to match the "Drag-and-Drop" feel of the Proposal Builder (Deep Dive 7).
- **description**: Build the horizontal pipeline view for managing leads.
- **requirements**:
    1. Implement the `InquiryPipeline` component with 4 default columns.
    2. Create the `LeadCard` component with "Heat" indicators and "Activity Pulse."
    3. Build the `InquiryDetailsDrawer` (slide-over) for reading long messages.
    4. Add a "Create Proposal" shortcut on every card.
- **acceptance-criteria**:
    - Smooth drag-and-drop between columns.
    - Clicking a card opens the detail drawer without changing the URL.
    - Responsive: On mobile, columns become vertically stacked or a "Tabbed" view.

---

### **7. GitHub Issue: "Quick Action" Lead Response Bar**

- **suggested-assignee**: Junior
- **priority**: Medium
- **consideration**: This feature reduces time-to-reply, which is the #1 metric for closing deals.
- **description**: Add a minimalist text area at the bottom of the Inquiry Drawer for instant replies.
- **requirements**:
    1. Add a "Send Message" input to the `InquiryDetailsDrawer`.
    2. Logic: Sends an email to the client using the Resend service.
    3. Show a "Sent" confirmation with a timestamp.
- **acceptance-criteria**:
    - Creator can reply to a lead in under 10 seconds.
    - Reply history is visible in the drawer.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 9: The Billing & Revenue Dashboard**? This is where the creator manages their Zeyaroo subscription and tracks their payout history.