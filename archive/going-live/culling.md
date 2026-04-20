 From both a technical and a UX standpoint, "trapdooring" the rejects into a specialized system-level container is the only way to maintain the "Luxury" feel of the platform.

If you don't hide them aggressively, the creator never feels like they are making progress. Professional culling is about **reducing noise.**

Here is the definitive architectural and UX logic for handling Rejects:

---

### 1. The "Trapdoor" Logic (The UX)
When a creator hits **`X`** in the Culling View, the photo should visually "drop" or "fade out."
*   **Immediate Action:** The photo is removed from the current view.
*   **Logical Move:** It is stripped from the "All Media" view and any user-created albums (e.g., "Ceremony").
*   **Destination:** It is placed in a system-generated view called **"Project Trash"** or **"Rejected"**.

### 2. Why a "System Album" is better than a "Folder"
In Zeyaroo, an image should only have one **Culling Status** but can be in many **Albums**. 
*   **The Problem:** If you just "hide" them, the "All Media" count stays at 1,000, which is overwhelming.
*   **The Practical Solution:** Treat "Rejected" as a **State**, not just a location. 
    *   `status = 'active'` -> Visible in grid, All Media, and user albums.
    *   `status = 'rejected'` -> Hidden from everything **except** the "Rejected" system view.

### 3. The "Client Shield" (The most critical part)
The biggest risk in this business is a "technical leak"—where a client accidentally sees a photo of themselves blinking or a blurry test shot because a developer forgot a filter.
*   **The Backend Rule:** The `Public/Shared Gallery API` must have a **Hard Filter** in the repository layer: 
    *   `SELECT * FROM media WHERE project_id = ? AND culling_status != 'reject'`
*   **The Safety Net:** Even if a creator accidentally adds a "Rejected" photo to a "Best Of" album, the system should prevent it from being rendered to the client.

---

### 4. Detailed "Junior Step-by-Step" Tasks

#### **Task: Implement the "Reject" State Engine**
- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Use a GORM "Scope" or a middleware filter to ensure Rejects are excluded from standard queries by default.
- **description**: Create a robust state transition for media items being rejected.
- **requirements**:
    1. Add `culling_status` (pick, reject, none) to the `media` table.
    2. Logic: When `status` moves to `reject`, the API must automatically remove that `media_id` from all entries in the `album_media` join table.
    3. Add a "Rescue" endpoint: `POST /media/:id/rescue` to move it back to `none`.
- **acceptance-criteria**:
    - Rejecting a photo via API makes it disappear from the `GET /projects/:id/media` response.
    - The photo remains in the database (not deleted).

#### **Task: Build the "Project Trash" (Rejects) View**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: This view should look "dimmed." Use a grayscale filter on this specific page to reinforce that these items are "dead."
- **description**: Create a "safety net" UI where creators can review and potentially "Rescue" rejected photos.
- **requirements**:
    1. Create a "Rejected" tab in the Project Sidebar.
    2. Display a masonry grid of only `culling_status == 'reject'`.
    3. Add a "Restore" button on hover.
    4. Add a "Empty Trash" button to permanently delete these files from Backblaze.
- **acceptance-criteria**:
    - Creator can see all their mistakes/rejects in one place.
    - Restoring a photo moves it back to the main "All Media" pool.

---

### 5. Practical Value to the Creator
1.  **Mental Clarity:** After culling 1,000 photos, the "All Media" view now only shows the 200 "keepers." It feels like the project is 80% done.
2.  **Safety:** They know they haven't deleted the files yet (just in case), but they don't have to look at them.
3.  **Speed:** By hiding the rejects, the main gallery loads faster and requires less scrolling.

### 6. Practical Value to YOU (The Owner)
1.  **The "Purge" Trigger:** You can now see exactly how much "trash" is in your B2 buckets. 
2.  **Monetized Cleaning:** You can send an automated email: *"You have 15GB of rejected photos across 5 projects. Empty your trash to free up space or upgrade your plan."*

**Verdict:**  move them to a "basement" view. **"Out of sight, out of mind"** is the most valuable thing you can offer a busy photographer.