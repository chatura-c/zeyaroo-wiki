This is **Deep Dive 6: The Vault (The Client Dashboard)**.

Once a client "Claims" their account (Topic 3), they move from a temporary "Magic Link" viewer to a permanent resident of **The Vault**. This is their personal archive of life’s moments. It must feel like a digital safe-deposit box: secure, permanent, and premium.

---

### **1. Wireframe Layout (Desktop)**
A **Spacious, Centered Content Layout** (Focus on high-quality project "tiles").

*   **The Identity Header (Top):**
    *   Large greeting in `font-serif`: "Welcome to your Vault, [Client Name]."
    *   A minimalist summary line: "3 life events secured • 4.2 GB used."
*   **The Project Collection (Center):**
    *   A clean grid of **Luxury Project Cards**.
    *   Each card features a large cover photo, the event date, and the Creator’s name.
    *   **Badges:** "Lifetime Access" (if paid) or "Expires in X days" (if on the creator's hosting).
*   **The "Discover Creators" Hook (Bottom):**
    *   A subtle "Hire Again" section showing profiles of creators the client has worked with before.
*   **The Settings Sidebar (Left/Right - Collapsible):**
    *   Security settings (Password change).
    *   Notification settings.
    *   Subscription (if the client upgrades to host their own files long-term).

---

### **2. Functional UX & Interactions**
*   **The "Hand-Over" Arrival:** When a client first logs in after claiming a project, show a one-time **"Welcome to the Vault"** modal with a subtle sparkle animation.
*   **Instant Access:** Clicking a project tile doesn't reload the page; it opens the project in a focused overlay/view so the user never feels like they are leaving their Vault.
*   **Bulk Download History:** A "Downloads" tab where clients can see the status of ZIP generations (since large projects take time to zip).

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Project Tile (The Vault Card)**
```tailwind
/* High-key White Theme */
relative group w-full aspect-[4/5] rounded-3xl overflow-hidden 
bg-white border border-gray-100 shadow-[0_4px_20px_rgba(0,0,0,0.03)] 
transition-all duration-500 hover:shadow-[0_20px_40px_rgba(0,0,0,0.08)] hover:-translate-y-1

/* Image */
w-full h-full object-cover transition-transform duration-700 group-hover:scale-105

/* Info Overlay (Bottom) */
absolute bottom-0 left-0 right-0 p-6 bg-gradient-to-t from-black/60 to-transparent
```

#### **The Account Status Badge**
```tailwind
/* Thin-stroke pill */
px-3 py-1 rounded-full border border-accent-primary/20 bg-accent-primary/5 
text-[10px] uppercase tracking-widest font-semibold text-accent-primary
```

---

### **4. Edge Cases**
*   **Only One Project:** If the client has only one project, the dashboard displays it as a massive **"Featured Hero"** rather than a tiny grid item to make the page feel full.
*   **Expiring Projects:** If a project is about to be deleted from the creator's storage, the card border pulses with a subtle amber glow and a "Save Forever" button appears.
*   **Mobile View:** A vertical list of large cards. The header summary becomes a horizontal scrollable bar at the top.

---

### **5. GitHub Issue: Implement Client "Vault" Dashboard UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use the "High-Key White" theme as the default for clients, as it feels more "archival" and neutral for family photos.
- **description**: Build the main dashboard for authenticated clients to manage their claimed projects.
- **requirements**:
    1. Implement the responsive project grid using the `VaultCard` design.
    2. Build the `VaultHeader` with account statistics (Total projects, storage used).
    3. Integrate "Project Expiry" visual logic (Amber warning for projects near deletion).
    4. Create the "Empty Vault" state: "No projects claimed yet. Use the link provided by your photographer to get started."
- **acceptance-criteria**:
    - Logged-in clients see only the projects they have claimed.
    - Project cards show the Creator's name as a link to their public profile.
    - Theme respects the user's "PreferredTheme" setting from Topic 9.

---

### **6. GitHub Issue: "Project Claimed" Success Celebration UI**

- **suggested-assignee**: Junior
- **priority**: Low
- **consideration**: Use a smooth `AnimatePresence` from Framer Motion for the modal entry.
- **description**: Create a moment of delight when a guest successfully converts their project into a Vault account.
- **requirements**:
    1. Build a `ClaimSuccessModal.tsx`.
    2. Include a "Your memories are now safe" Serif headline.
    3. Trigger a light-colored confetti burst (Topic 8).
- **acceptance-criteria**:
    - Modal appears immediately after the "Claim" API call returns success.
    - Clicking "Enter my Vault" closes the modal and shows the project grid.

---

### **7. GitHub Issue: Client Security & Access Log View**

- **suggested-assignee**: Backend
- **priority**: Medium
- **consideration**: This builds high trust for clients with sensitive/private photos.
- **description**: Allow clients to see when and where their Vault was accessed.
- **requirements**:
    1. API endpoint to list last 5 login events (Timestamp, Location/IP, Device).
    2. Simple list UI in the Vault settings section.
- **acceptance-criteria**:
    - Client can see a list of recent activities on their account.
    - Unauthorized access can be identified quickly.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 7: The Proposal Builder (Creator Flow)**? This is where the creator drags-and-drops milestones to build the "Invitation" we designed in Deep Dive 3.