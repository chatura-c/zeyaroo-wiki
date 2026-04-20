This is **Deep Dive 5: The Selection Room (Client Favorites)**.

The Selection Room is a specialized "subset" of the Shared Gallery. Many professional photographers offer packages like "Shoot 100, Edit 20." This interface provides a high-trust, friction-free way for clients to pick their final images and for creators to receive that "order."

---

### **1. Wireframe Layout (Desktop)**
A **Split-Focus Interface** with a persistent **Selection Tray**.

*   **The Quota Header (Top):**
    *   A centered glassmorphic pill.
    *   Text: "Pick your 20 favorites to be edited."
    *   **Progress Indicator:** A thin, high-end progress line (e.g., `12 / 20 selected`).
*   **The Selection Grid (Center):**
    *   The Masonry Gallery but with a "Selection Toggle" overlay on every image.
    *   Images already selected have a soft, brand-accented glow or a checkmark.
*   **The Review Tray (Bottom - Sticky):**
    *   A horizontal scrolling bar of small thumbnails.
    *   Shows exactly what the client has picked so far.
    *   **Action:** A prominent [Submit Final Selection] button.
*   **The Comparison Mode (Modal/Overlay):**
    *   Triggered when a client clicks a "Compare" icon.
    *   Shows 2 or 3 similar photos side-by-side to help the client choose the best one.

---

### **2. Functional UX & Interactions**
*   **The "One-Tap" Select:** Clicking the "Heart" on a masonry item adds it to the bottom tray with a flight animation (the thumbnail "flies" into the tray).
*   **Quota Enforcement:** If the client tries to select 21 when the limit is 20, a gentle toast appears: *"You've reached your limit. Deselect an image to add a new one."*
*   **Locking:** Once the client hits "Submit," the selection is locked and an email is sent to the creator. The client can no longer change their mind without the creator "Unlocking" the project.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Selection Progress Pill**
```tailwind
/* Fixed top center */
fixed top-24 left-1/2 -translate-x-1/2 z-40
bg-white/90 backdrop-blur-md px-6 py-2 rounded-full border border-gray-100 shadow-sm
/* Text */
font-sans text-xs font-medium tracking-wider text-gray-900
```

#### **The Selection Tray (Bottom)**
```tailwind
/* Sticky Tray */
fixed bottom-0 left-0 right-0 h-28 bg-white/80 backdrop-blur-2xl border-t border-gray-100 
flex items-center px-8 gap-4 z-50 transform translate-y-0 transition-transform

/* Thumbnail in tray */
w-16 h-16 rounded-lg object-cover border border-gray-200 hover:border-accent-primary
```

#### **Compare View Sidebar**
```tailwind
/* side-by-side flex container */
flex flex-row w-full h-[80vh] gap-4 p-8
/* Image container */
flex-1 rounded-2xl overflow-hidden bg-surface-800
```

---

### **4. Edge Cases**
*   **No Quota Set:** If the creator doesn't set a limit, the room acts as a "General Favorites" list for the client's own organization.
*   **Vertical vs Horizontal in Compare View:** The comparison logic should use `object-contain` to ensure the client can see the full crop of both images regardless of orientation.
*   **Mobile View:** The bottom tray becomes a "View Selection" button that opens a full-screen drawer of picked images.

---

### **5. GitHub Issue: Implement Client Selection Database Logic**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Selection limits should be stored at the `Project` level.
- **description**: Create the data structures required to track client picks within a project.
- **requirements**:
    1. Add `SelectionLimit int` to `models.Project`.
    2. Create `models.ClientSelection` table: `id`, `project_id`, `media_id`, `user_id/guest_email`.
    3. Endpoint: `POST /api/v1/projects/:id/selection` (bulk update list of IDs).
    4. Endpoint: `POST /api/v1/projects/:id/selection/submit` (locks the selection).
- **acceptance-criteria**:
    - Backend correctly stores a list of IDs for a specific project.
    - Attempting to add more than the `SelectionLimit` returns a `400 Bad Request`.

---

### **6. GitHub Issue: Implement Selection Mode UI & Progress Tracking**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use a local state for the "Selection Tray" to make it feel fast, but sync with the API periodically.
- **description**: Build the visual "Selection Mode" for the public gallery.
- **requirements**:
    1. Implement the `SelectionProgressPill` component.
    2. Add "Heart" toggle to Masonry items.
    3. Build the `SelectionTray` component (Sticky bottom bar).
    4. Add a "Submit Selection" confirmation modal.
- **acceptance-criteria**:
    - Selecting an image updates the counter and adds the thumbnail to the tray.
    - Counter turns red or shows "Limit Reached" when the quota is hit.
    - Submit button triggers the backend lock.

---

### **7. GitHub Issue: Media Comparison Tool (Side-by-Side)**

- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: This is a "Premium" feel feature. Ensure it works well on tablets (iPad).
- **description**: Build an interface that allows clients to view 2 images side-by-side for comparison.
- **requirements**:
    1. Add a "Compare" button to the Lightbox.
    2. Create a `ComparisonModal` that takes 2 `MediaObject` IDs.
    3. Allow zooming/panning in sync (Optional/Stretch Goal).
- **acceptance-criteria**:
    - Client can select two similar shots and view them at 50% width each to decide which to "Heart."

---

**Next Deep Dive?**
Shall we move to **Deep Dive 6: The Vault (The Client Dashboard)**? This is the private area for clients who have "Claimed" their projects (Topic 3).