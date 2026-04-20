### Feature Title: The Mutual Amendment Protocol
**Goal:** Allow the dynamic adjustment of project scope (Chapters and Pricing) after a project has already moved into the `Active` state, ensuring escrow security is never compromised.

---

### **Task 1: Implement Amendment Staging Models**
- **suggested-asignee**: Senior Backend
- **priority**: High
- **consideration**: We must not touch the live `chapters` table during the drafting phase. If the database schema has shifted to use "Chapters" instead of "Milestones," ensure these new models reflect that.
- **description**: Create the database structures to hold "Proposed" changes to a project.
- **requirements**:
    1. Create `ProjectAmendment` model: `id`, `project_id`, `version`, `status` (draft, pending_approval, applied, rejected), `creator_note`.
    2. Create `AmendmentChapter` model: This should be a near-duplicate of your current `Chapter` model but linked to an `AmendmentID` instead of a `ProjectID`.
- **acceptance-criteria**:
    - A migration exists adding these tables.
    - The backend can store a "Draft" set of 5 chapters for a project that currently only has 3 active chapters without affecting the client's view.

---

### **Task 2: Propose Amendment Service & Logic**
- **suggested-asignee**: Backend
- **priority**: High
- **consideration**: Only the Creator should be able to initiate an amendment in V1.
- **description**: Implement the service logic to transition a draft amendment to `pending_approval`.
- **requirements**:
    1. Endpoint: `POST /api/v1/projects/:id/amendments` (Creates a draft from current active chapters).
    2. Endpoint: `POST /api/v1/projects/:id/amendments/:aid/publish` (Moves to pending).
    3. Generate a unique `AmendmentToken` for guest-client access.
- **acceptance-criteria**:
    - Publishing an amendment locks the creator from further edits until the client responds.
    - An event is emitted to the notification system.

---

### **Task 3: "Edit Scope" Workspace UI (Creator)**
- **suggested-asignee**: Frontend
- **priority**: High
- **consideration**: Reuse the **Proposal Builder** components. The creator should feel like they are back in the "Drafting" phase.
- **description**: A UI mode inside the Project Workspace to modify the current contract.
- **requirements**:
    1. Add an "Amend Contract" button in the project header.
    2. Load current chapters into the `MilestoneBuilder` component.
    3. Allow adding new chapters, deleting *unfunded* ones, and changing prices.
- **acceptance-criteria**:
    - Creator can modify the list and see a "New Total" compared to the "Old Total."

---

### **Task 4: The "Contract Diff" Visualizer (Client)**
- **suggested-asignee**: Frontend
- **priority**: High
- **consideration**: Clarity is key for trust. Use red/green color coding to show what changed.
- **description**: A comparison view for the client to see exactly what the creator is proposing to change.
- **requirements**:
    1. Route: `/p/:token/amendment/:atoken`.
    2. Side-by-side view: "Current Chapters" vs "Proposed Chapters."
    3. Clearly highlight: Price increases (Green +), Price decreases (Red -), and New Chapters (Badge: New).
- **acceptance-criteria**:
    - Client can clearly see that "Chapter 2 was $500, now it is $700."

---

### **Task 5: The "Escrow Safeguard" Validator**
- **suggested-asignee**: Senior Backend
- **priority**: High
- **consideration**: This is the most dangerous edge case. We must prevent an amendment from "deleting" money Zeyaroo is already holding.
- **description**: Logic to prevent the removal or reduction of funded milestones.
- **requirements**:
    1. Middleware/Validator: If an active chapter has `status = 'funded'` or `'submitted'`, the amendment **cannot** delete it or reduce its `amount_cents`.
    2. It **can** increase the price of a funded chapter (which creates a "Balance Due" state).
- **acceptance-criteria**:
    - API returns `422 Unprocessable Entity` if the creator tries to lower the price of an already-paid chapter.

---

### **Task 6: The "Atomic Swap" (Apply Amendment)**
- **suggested-asignee**: Senior Backend
- **priority**: High
- **consideration**: This MUST be an atomic SQL transaction. If any part fails, the project must stay on the old contract.
- **description**: The logic that runs when the client clicks "Approve."
- **requirements**:
    1. `ApplyAmendment(amendmentID)` service.
    2. Step 1: Backup current chapters to a `History` table.
    3. Step 2: Delete current active chapters (soft delete).
    4. Step 3: Insert `AmendmentChapters` into the main `Chapters` table.
    5. Step 4: Update `Project.total_value`.
- **acceptance-criteria**:
    - Upon client approval, the Project dashboard immediately reflects the new chapter structure and pricing.

---

### **Task 7: "Top-Up" Payment Logic for Amendments**
- **suggested-asignee**: Backend
- **priority**: Medium
- **consideration**: If I already paid $100 for Chapter 1, and the amendment makes Chapter 1 cost $150, I owe $50.
- **description**: Handle price increases on chapters that were already marked as `funded`.
- **requirements**:
    1. If an applied amendment increases the price of a `funded` chapter, move that chapter's status back to `partially_funded` or keep it `funded` but add a "Due Balance" flag.
    2. (Recommended V1): Simply create a NEW Chapter called "Price Adjustment" for the difference.
- **acceptance-criteria**:
    - The client is prompted to pay the difference if an amendment increases the cost of a paid phase.

---

### **Task 8: Amendment Notification Loop**
- **suggested-asignee**: Fullstack
- **priority**: Medium
- **consideration**: Use WhatsApp for speed, as the project is already "Active" and the client is likely waiting.
- **description**: Trigger communications for the amendment lifecycle.
- **requirements**:
    1. Notification: "Project update proposed for [Project Name]. Please review and approve."
    2. Notification: "Scope changes approved! Your project has been updated."
- **acceptance-criteria**:
    - Client receives a link to the "Diff" view immediately after the creator publishes.

---

### **Task 9: Amendment Rejection Workflow**
- **suggested-asignee**: Junior Fullstack
- **priority**: Low
- **consideration**: Rejection shouldn't kill the project; it should just close the "Negotiation" window.
- **description**: Logic for when a client says "No" to the proposed changes.
- **requirements**:
    1. Update `Amendment.status` to `rejected`.
    2. Allow client to leave a "Reason for Rejection" note.
    3. Notify the creator.
- **acceptance-criteria**:
    - Creator sees the rejection reason and can choose to either "Re-draft" or continue with the original contract.

---

### **Task 10: "Contract Version" History Tab**
- **suggested-asignee**: Frontend
- **priority**: Low
- **consideration**: This builds massive professional trust for long-term projects.
- **description**: A "History" view in the project settings showing all previous versions of the contract.
- **requirements**:
    1. A simple list of "Applied Amendments."
    2. Ability to view what the project cost/structure looked like on Day 1 vs. Day 100.
- **acceptance-criteria**:
    - Users can see a chronological audit trail of all mutual agreements.

---

### **The "Amendment Protocol" Logic (Dev Note)**
*   **Safety Rule:** Never allow an amendment if a dispute is currently open.
*   **Naming:** In the code, use `ProposedChapter` and `ActiveChapter` to prevent naming collisions.
*   **State Conflict:** If a client releases a payment *while* an amendment is pending, the system should automatically "Auto-Reject" the pending amendment to prevent a state mismatch. (The creator must re-propose based on the new project state).