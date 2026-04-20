This is **Deep Dive 2: The Project Workspace (Media Management & Culling)**.

This is the most complex page in the application. It needs to handle high volumes of media while maintaining a "minimalist studio" aesthetic. It’s where the creator performs the actual "work" of the project.

---

### **1. Wireframe Layout (Desktop)**
We move to a **Three-Pane Integrated Layout** to maximize productivity.

*   **Left Navigation (Cols 1-2):**
    *   **The Tree:** Minimalist list of Albums.
    *   **Smart Views:** "All Photos," "Recently Uploaded," "Favorites," and "Trash."
    *   **The Janitor:** Storage indicator for this specific project.
*   **Center Stage (Cols 3-10):**
    *   **The Gallery:** The high-resolution Masonry Grid.
    *   **The Mode Switcher:** A subtle floating tab at the top-center: [Grid View] | [Culling View].
    *   **The Filter Bar:** Icons for "Filter by Rating," "Filter by Type (Photo/Video)," and "Search Tags."
*   **Right Inspector (Cols 11-12):**
    *   **Selection Info:** Visible only when an item is selected. Shows EXIF data (ISO, Lens, Camera).
    *   **Quick Actions:** [Set as Cover], [Move to Album], [Watermark On/Off], [Delete].

---

### **2. Functional UX & Interactions**
*   **The "Speed-Run" Culling:** When in "Culling Mode," the UI enters full-screen.
    *   `Arrow Keys`: Navigate through photos.
    *   `Spacebar`: Toggle "Favorite."
    *   `X Key`: Mark as "Rejected" (dims the image).
    *   `Number Keys (1-5)`: Set star rating.
*   **Bulk Selection:** Standard `Shift + Click` or `Cmd + Click` behavior to batch watermark or move 100+ files to an album.
*   **Drag-to-Album:** Dragging selected items from the grid over an album name in the Left Navigation highlights the album and moves the files.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Masonry Grid Item**
```tailwind
/* Image Container */
relative overflow-hidden rounded-xl bg-surface-800 transition-all duration-300

/* Selection State */
ring-2 ring-accent-primary ring-offset-4 ring-offset-surface-primary

/* Hover Overlay (Glass) */
absolute inset-0 bg-white/10 dark:bg-black/10 backdrop-blur-[2px] opacity-0 group-hover:opacity-100 transition-opacity
```

#### **The Mode Switcher (Floating)**
```tailwind
/* Floating Tab */
fixed top-6 left-1/2 -translate-x-1/2 z-50 flex p-1 glass rounded-full border border-subtle shadow-2xl
```

#### **The Inspector (Right Sidebar)**
```tailwind
/* Header */
font-sans text-[10px] uppercase tracking-widest text-text-muted mb-4
/* Data Points */
flex justify-between py-2 border-b border-subtle text-xs
```

---

### **4. Edge Cases**
*   **Processing State:** While thumbnails are being generated (Topic 7), the grid item shows a "developing" shimmer effect (Blurry placeholder + loading spinner).
*   **Video Playback:** Videos show a "Play" icon in the center. On hover, a 3-second silent "GIF-style" preview plays automatically.
*   **Massive Projects:** For projects with 500+ items, the scrollbar becomes a "Timeline Scroller" showing month/date as you scroll (Virtualized Grid).

---

### **5. GitHub Issue: Implement Project Workspace & Culling UI**

- **suggested-assignee**: Fullstack
- **priority**: High
- **consideration**: Use `react-window` or `react-virtuoso` for the media grid to prevent browser lag with 1,000+ photos.
- **description**: Build the 3-pane project workspace with a focus on high-speed culling and bulk media management.
- **requirements**:
    1. Implement 3-pane grid layout (Folders | Media | Inspector).
    2. Build the `CullingMode` component (Full-screen, keyboard-driven navigation).
    3. Implement `MultiSelect` logic for bulk actions using a Zustand store.
    4. Create `SmartFilters` component (Star ratings, rejected status).
    5. Add "Video Preview" logic (Hover to play muted preview).
- **acceptance-criteria**:
    - Users can switch between Grid and Culling mode without page reload.
    - Keyboard shortcuts (P, X, 1-5) update the database/local state instantly.
    - Dragging media into the Sidebar Album list successfully updates the album relationship.
    - Inspector pane accurately displays EXIF metadata from the file.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 3: The Digital Invitation (Public Proposal)**? This is the client's high-impact entry point into the Zeyaroo experience.