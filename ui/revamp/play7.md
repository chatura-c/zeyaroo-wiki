This is **Deep Dive 7: The Proposal Builder (Creator Flow)**.

This is where the creator’s "Business Intelligence" happens. Instead of a standard form, it must feel like a **collaborative drafting table**. The UI should provide immediate feedback and allow the creator to visualize exactly what the client will see.

---

### **1. Wireframe Layout (Desktop)**
A **Three-Column Drafting Workspace** (Maximize horizontal space).

*   **Left Column - Logistics (3 cols):**
    *   **Client Info:** Name and Email.
    *   **Cover Selector:** A mini-grid of the project’s photos to pick the "Hero" for the Digital Invitation.
    *   **Logistics:** Date, Location, and Currency (LKR/USD switcher).
*   **Center Column - The Canvas (6 cols):**
    *   **Milestone Stack:** A list of vertical cards. 
    *   **Interaction:** Drag handles on the left of each card to reorder.
    *   **Dynamic Inputs:** Milestone Title, Amount, Description, and "Deliverable Type" toggle.
    *   **The "+" Add Button:** A dashed-border card at the bottom: "Add a Phase."
*   **Right Column - Live Preview (3 cols):**
    *   **The "Mini-Invitation":** A scaled-down, read-only version of the page we designed in Deep Dive 3. 
    *   **Real-time Update:** As the creator types a milestone name, it appears instantly in the preview.

---

### **2. Functional UX & Interactions**
*   **The "Total" Watcher:** A sticky bar at the top of the builder shows the running total. If the sum of milestones doesn't match the "Total Contract Value," the bar turns amber with a "Math Mismatch" warning (Topic 2).
*   **Drag-to-Reorder:** Smooth reordering of milestones using `dnd-kit` or `framer-motion`. Reordering updates the `sequence` field in the database.
*   **Template Injection:** A "Magic Wand" icon at the top allowing creators to "Apply Wedding Template" or "Apply Studio Template," pre-filling 3-4 standard milestones.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Milestone Card (In-Builder)**
```tailwind
/* Container */
p-6 bg-white border border-gray-100 rounded-2xl shadow-sm transition-all focus-within:ring-2 focus-within:ring-accent-primary/20

/* Drag Handle */
cursor-grab active:cursor-grabbing text-gray-300 hover:text-gray-500

/* Input Styling */
bg-transparent border-none focus:ring-0 text-gray-900 placeholder:text-gray-300 font-medium
```

#### **The Live Preview Panel**
```tailwind
/* Scaling the preview down to fit a sidebar */
fixed top-24 right-8 w-80 h-[80vh] bg-white border border-gray-200 rounded-3xl overflow-hidden shadow-2xl origin-top scale-[0.8]
```

---

### **4. Edge Cases**
*   **Unsaved Changes:** If the creator tries to leave the page with an unsaved proposal, a "Luxury" modal appears: *"Your masterpiece isn't finished. Save as draft?"*
*   **Single Milestone:** If a creator only wants one payment (e.g., "Full payment upfront"), the builder collapses the list into a single "Simple Invoice" view.
*   **Mobile View:** The 3 columns stack vertically. The "Live Preview" becomes a floating "Eye" icon that opens a full-screen preview overlay.

---

### **5. GitHub Issue: Implement Proposal Builder Logic (Backend)**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Ensure the `total_amount` is always re-calculated on the server even if the client sends a total in the request body (Topic 2 security).
- **description**: Create the API endpoints to manage multi-milestone proposal drafts and versioning.
- **requirements**:
    1. Endpoint: `POST /api/v1/proposals` (Create draft).
    2. Endpoint: `PUT /api/v1/proposals/:id/milestones` (Bulk update and re-sequence milestones).
    3. Implement logic to store `CoverImageID` and `ProjectMetadata`.
    4. Validation: Check that the sum of milestones matches the proposal total.
- **acceptance-criteria**:
    - Backend accepts an array of milestones with specific sequence numbers.
    - Database reflects the correct order when queried by the frontend.

---

### **6. GitHub Issue: Interactive Drag-and-Drop Builder UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `Framer Motion` for smooth layout transitions when milestones are added or removed.
- **description**: Build the interactive center column for drafting milestones.
- **requirements**:
    1. Implement the `MilestoneBuilderCard` with inline editing.
    2. Use `dnd-kit` for drag-and-drop reordering.
    3. Add the "Add Milestone" placeholder with dashed animation.
    4. Build the "Total Watcher" sticky bar with mismatch validation alerts.
- **acceptance-criteria**:
    - Creator can add, remove, and reorder milestones without page lag.
    - Real-time total calculation works for 10+ milestones.
    - Dragging a milestone correctly updates its sequence in the local state.

---

### **7. GitHub Issue: Split-Screen "Live Preview" Implementation**

- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: Use a simple iframe or a shared component structure with the Public Proposal page to ensure the preview is 100% accurate.
- **description**: Create a real-time visualization of the proposal for the creator.
- **requirements**:
    1. Implement a `ProposalPreview` component that consumes the current "Builder State."
    2. Apply `transform: scale(0.8)` to the preview container to fit the sidebar.
    3. Ensure font styles (Instrument Serif) are correctly rendered in the preview.
- **acceptance-criteria**:
    - Typing in the builder updates the preview text instantly.
    - Selecting a different `CoverImageID` updates the preview header image instantly.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 8: The Inquiry & Lead Manager (CRM)**? This is where the creator organizes incoming requests before they become proposals.