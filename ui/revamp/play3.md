This is **Deep Dive 3: The Digital Invitation (Public Proposal)**.

This is the most important "Sales" surface of the platform. It must feel like a premium digital magazine or a high-end invitation. Its goal is to build such high levels of trust that the client feels comfortable paying a large deposit immediately.

---

### **1. Wireframe Layout (Desktop)**
An **Editorial, Single-Column Scrolling Layout** (Centered, 800px max width for readability).

*   **The Cinematic Hero (Top):** 
    *   Full-width bleed. Height: 70vh.
    *   Large background photo (the `CoverImageID`).
    *   Bottom-left overlay: Client Name & Event Date in massive `font-serif`.
*   **The Intro (The Personal Touch):** 
    *   A centered minimalist avatar of the creator.
    *   Serif quote: "I am honored to capture your story."
*   **Deliverables (The Promise):** 
    *   Large typography. Instead of bullet points, use wide cards with generous padding.
    *   Example: "200+ Hand-edited High Resolution Photos."
*   **The Path (Milestones):** 
    *   A vertical timeline. Each step shows the status (To be paid, In Progress, Delivery).
    *   Currency is clearly displayed in the creator's choice (USD/LKR).
*   **The Logistics (Questionnaire):**
    *   A minimalist form embedded directly into the scroll flow to capture project details.
*   **Sticky Footer (The Decision):**
    *   A glassmorphic bar that appears only after the user scrolls past the hero. 
    *   Contains: [Total Amount], [Decline], and a prominent [Accept & Pay Deposit] button.

---

### **2. Functional UX & Interactions**
*   **Parallax Scroll:** The Hero image should have a slow parallax effect as the client scrolls down.
*   **Milestone Hover:** Hovering over a milestone reveals a tooltip explaining the Escrow protection ("Funds held safely by Zeyaroo until you approve").
*   **The "Agreement" Checkbox:** The "Accept" button is disabled until the client scrolls through the "Standard Service Agreement" section at the bottom.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Hero Typography Overlay**
```tailwind
/* Container */
absolute bottom-12 left-12 z-10

/* Client Name */
font-serif text-6xl md:text-8xl text-white tracking-tighter

/* Date/Location Label */
font-sans text-xs uppercase tracking-[0.4em] text-white/70 mt-4
```

#### **The Milestone Vertical Path**
```tailwind
/* Timeline Line */
absolute left-4 top-0 bottom-0 w-[1px] bg-gray-200

/* Active Dot */
w-8 h-8 rounded-full border border-gray-200 bg-white flex items-center justify-center shadow-sm
```

#### **Sticky Action Bar (Light Mode)**
```tailwind
/* Glass bar */
fixed bottom-6 left-1/2 -translate-x-1/2 w-[90%] max-w-3xl 
bg-white/80 backdrop-blur-xl border border-gray-100 shadow-[0_20px_50px_rgba(0,0,0,0.1)] 
rounded-2xl p-4 flex items-center justify-between
```

---

### **4. Edge Cases**
*   **No Cover Image:** If the creator hasn't set a cover, use a sophisticated "Canvas Grey" (#E5E5E5) background with a centered Serif monogram of the creator's initials.
*   **Mobile View:** The Hero text scales down significantly (`text-4xl`). The sticky footer becomes a full-width fixed bar at the very bottom (0px margin) to maximize thumb reach.
*   **Expired Token:** If the Magic Link token is invalid, show a beautiful 404 page: *"This invitation has expired. Please contact [Creator Name] for a new link."*

---

### **5. GitHub Issue: Implement Public Proposal (Digital Invitation) UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: This page must have zero "Layout Shift." Use height-preserving placeholders for the hero image.
- **description**: Build the public-facing proposal page. This is the unauthenticated view for clients. It must use the "Instrument Serif" font and follow the Editorial layout.
- **requirements**:
    1. Create route `/p/:token` with a minimalist unauthenticated layout.
    2. Implement the `CinematicHero` component with parallax scroll effects.
    3. Build the `VerticalMilestonePath` component to visualize project stages.
    4. Implement the `StickyProposalAction` bar with "Accept/Decline" logic.
    5. Add "Standard Service Agreement" text block with a mandatory "I Agree" checkbox.
    6. Ensure the page is responsive (Mobile-optimized for iPhone/Android browsers).
- **acceptance-criteria**:
    - Client can view all proposal details without logging in.
    - Hero image renders at full-screen width with an elegant Serif overlay.
    - Milestone list accurately reflects amounts and sequences from the DB.
    - "Accept" button triggers the DodoPayments/LKR funding flow for the first milestone.
    - Page passes Google PageSpeed mobile score of 90+ (essential for low-bandwidth clients).

---

### **6. GitHub Issue: Automated Proposal Invitation Email Template**

- **suggested-assignee**: Backend
- **priority**: Medium
- **consideration**: Use Inline CSS for high compatibility across Gmail and Outlook.
- **description**: Design and implement the HTML email sent to clients when a creator clicks "Send Proposal."
- **requirements**:
    1. Use a single-column minimalist design.
    2. Large "View Proposal" button in the center.
    3. Visual: Include the project thumbnail in the email body.
    4. Text: "You have received an invitation from [Creator Name] for [Project Title]."
- **acceptance-criteria**:
    - Email renders correctly on Gmail (Mobile/Desktop).
    - Clicking the button takes the user directly to the `/p/:token` page.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 4: The Shared Gallery (The Unboxing)**? This is the immersive masonry experience where the client finally sees their photos.