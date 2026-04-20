This is **Deep Dive 4: The Shared Gallery (The Unboxing)**.

This is the "moment of truth." The client has paid their milestones and is finally seeing the result. It must feel like an **exclusive art exhibition**, not a file-sharing link.

---

### **1. Wireframe Layout (Desktop)**
An **Immersive, Full-Width Masonry Layout**.

*   **The Global Status Bar (Top):**
    *   A ultra-slim floating glass bar.
    *   **Left:** Creator’s logo/monogram.
    *   **Center:** Project Title (e.g., *"The Silva-Perera Wedding"*) in `font-serif`.
    *   **Right:** [Download All] button and [Heart/Selection] counter.
*   **The Content Stage (Center):**
    *   **Masonry Grid:** Zero-border images with varying heights.
    *   **Section Headers:** (Optional) If the creator has albums like "Ceremony" or "Reception," these appear as elegant Serif titles that break the grid.
*   **The Interaction Layer:**
    *   When an image is clicked, it expands into a **Distraction-Free Lightbox**.
*   **The Monetization Trigger (Bottom):**
    *   If the project isn't fully paid, a persistent "locked" bar sits at the bottom: *"You are viewing watermarked previews. Release final payment to unlock original high-resolution files."*

---

### **2. Functional UX & Interactions**
*   **"The Reveal" Animation:** On page load, images should fade in one-by-one with a slight upward slide (`staggerChildren` in Framer Motion).
*   **Hover State:** Hovering over an image reveals a subtle "Heart" icon and a "Magnify" icon.
*   **The Lightbox:** 
    *   Black background (even in light mode) to make the photo colors stand out.
    *   Keyboard navigation (`Left`/`Right` arrows).
    *   Download button per-image (if project is unlocked).
*   **Lazy Loading:** As the client scrolls, new batches of images load seamlessly without a "Load More" button.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Masonry Grid Container**
```tailwind
/* Using CSS Columns for lightweight masonry */
columns-1 sm:columns-2 lg:columns-3 xl:columns-4 gap-6 p-6 lg:p-12
```

#### **The Preview Badge (Watermarked State)**
```tailwind
/* Elegant small pill in top-right of image */
absolute top-4 right-4 px-2 py-1 bg-white/90 backdrop-blur-sm 
text-[9px] font-bold tracking-widest text-black rounded-full 
border border-black/5 shadow-sm opacity-0 group-hover:opacity-100 transition-opacity
```

#### **Floating Action Bar (Status Bar)**
```tailwind
/* High-key Light Mode */
fixed top-6 left-1/2 -translate-x-1/2 z-50 w-[95%] max-w-6xl
bg-white/70 backdrop-blur-2xl border border-gray-100 shadow-xl 
rounded-full py-3 px-8 flex items-center justify-between
```

---

### **4. Edge Cases**
*   **Portrait vs Landscape Balance:** The masonry engine should prioritize keeping the "weight" of the columns even so the bottom isn't jagged.
*   **Video Playback:** Videos in the gallery show a "Duration" badge. Clicking them opens the Lightbox with an auto-playing minimalist video player (no heavy YouTube/Vimeo UI).
*   **Mobile View:** Single column masonry. The top bar collapses into a minimalist "Menu" icon to keep the screen focused on the art.

---

### **5. GitHub Issue: Implement Public Shared Gallery (Masonry) UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Performance is the priority. Use `srcset` for responsive images so mobile users don't download 20MB files (Topic 7).
- **description**: Build the public-facing gallery using a masonry layout. This page must support both "Gallery White" and "Obsidian" themes based on creator preference.
- **requirements**:
    1. Implement the `MasonryGrid` using CSS columns or `react-plm-masonry`.
    2. Build the `FloatingGalleryNav` component with dynamic "Download All" status.
    3. Implement "Fade-in & Slide-up" stagger animations for image loading.
    4. Create the `WatermarkBadge` UI for unreleased projects.
    5. Support for high-resolution video thumbnails.
- **acceptance-criteria**:
    - Gallery loads 50+ images smoothly without scroll lag.
    - Native aspect ratios are preserved (no square cropping).
    - "Download All" button is disabled/hidden if the final milestone is unpaid.
    - Page is fully responsive.

---

### **6. GitHub Issue: Implementation of High-End Media Lightbox**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Lightbox should support touch gestures (swipe to close/next) for mobile users.
- **description**: Create a premium full-screen lightbox for viewing individual media items.
- **requirements**:
    1. Dark-mode backdrop with 90% opacity.
    2. Image metadata display (Filename, Date).
    3. [Heart/Favorite] toggle in the lightbox.
    4. [Download Original] button (linked to pre-signed URL from Topic 7).
    5. Transition animations between images (Horizontal slide).
- **acceptance-criteria**:
    - Clicking any image in the grid opens the lightbox at the correct index.
    - Keyboard arrows work for navigation.
    - Closing the lightbox returns the user to their exact scroll position in the grid.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 5: The Selection Room (Client Favorites)**? This is where the client picks specific photos for the creator to edit/print.