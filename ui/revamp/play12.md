This is **Deep Dive 12: The Marketplace (Creator Discovery)**.

This is the high-level entry point where the platform's brand is established. It needs to feel like a **luxury directory** where the technology disappears behind the art. The goal is to make the "Search" feel like "Discovery."

---

### **1. Wireframe Layout (Desktop)**
A **"Magazine Rhythm" Grid** with a **Minimalist Filter Header**.

*   **The Atmospheric Header (Top):**
    *   A massive, centered search input that blends into the background.
    *   **Serif Tagline:** "Find the eye that sees your story."
    *   **Quick Filter Pills:** [Wedding] [Editorial] [Film] [Commercial].
*   **The Grid Canvas (Main Stage):**
    *   A **12-column CSS Grid** using `grid-auto-flow: dense`.
    *   **Artist Tiles:** Varied sizes. Some tiles take 1 column, others take 2 columns (Featured) to create a high-fashion editorial layout.
*   **The Context Drawer (Right - Slide-over):**
    *   Triggered when clicking a "Quick View" icon on a tile.
    *   Shows a mini-bio and the "Send Inquiry" form without leaving the search results.

---

### **2. Functional UX & Interactions**
*   **The "Style Reveal" Hover:** When a mouse enters an Artist Tile, the static cover image begins a cross-fade slideshow of their 4 most impressive portfolio shots (interval: 1.5s).
*   **Parallax Fade:** As the user scrolls, tiles should fade in and slide up slightly from the bottom using a `0.2s` staggered delay.
*   **Smart Query Persistence:** If a user clicks into a profile and goes back, the search position and filters are perfectly preserved.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Artist Tile (Marketplace Card)**
```tailwind
/* Container */
relative aspect-[3/4] rounded-[2.5rem] overflow-hidden bg-surface-800 group

/* Featured Tile (2x size) */
col-span-1 md:col-span-2 row-span-2

/* The Image Slideshow Layer */
absolute inset-0 z-0 transition-opacity duration-700

/* Info Overlay (Bottom) */
absolute bottom-0 left-0 right-0 p-8 z-10 bg-gradient-to-t from-black/80 to-transparent
```

#### **The High-End Search Input**
```tailwind
/* Centered Pill */
w-full max-w-2xl bg-white/5 backdrop-blur-xl border border-white/10 
rounded-full py-5 px-10 text-xl font-light tracking-wide
/* Focus State */
focus:bg-white/10 focus:border-accent-primary/40 focus:ring-4 focus:ring-accent-primary/5
```

---

### **4. Edge Cases**
*   **Low Result Count:** If only 3 creators match, don't leave massive white space. Display the 3 matches as "Featured Large" tiles.
*   **Missing Thumbnails:** Use a "Silk-Textured" placeholder with a single centered letter (the creator's initial) in a thin serif font.
*   **Mobile View:** The "Dense Grid" moves to a single vertical column of large, edge-to-edge cards to mimic an Instagram feed.

---

### **5. GitHub Issue: Implement Dense Grid Search API (Backend)**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Use `GORM` scopes to chain filters dynamically based on query params.
- **description**: Build the search endpoint that powers the marketplace.
- **requirements**:
    1. Endpoint: `GET /api/v1/public/creators`.
    2. Filters: `category` (array), `location` (city/string), `verified` (bool), `price_tier` (1-3).
    3. Weighted Sort: Rank verified creators and those with `is_featured: true` at the top.
    4. Limit/Offset: Default to 24 results per page.
- **acceptance-criteria**:
    - Querying `/creators?category=Wedding&location=Colombo` returns only matching creators.
    - Verified creators appear at the top of the list.

---

### **6. GitHub Issue: Build Magazine-Rhythm Grid UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `Tailwind`'s `grid-flow-dense` to ensure there are no empty "holes" in the layout when 2x2 tiles are mixed with 1x1 tiles.
- **description**: Implement the high-end discovery grid with varying tile sizes.
- **requirements**:
    1. Implement the `ArtistGrid` using CSS Grid.
    2. Logic: For every 5 tiles, make the 3rd tile a "Featured" tile (`col-span-2`).
    3. Create the `ArtistTile` component with the cross-fade slideshow animation.
    4. Implement "Scroll Reveal" using `IntersectionObserver`.
- **acceptance-criteria**:
    - The layout is non-uniform and feels like a professional magazine.
    - Images are responsive and use the `web_preview` version (Topic 7) to save bandwidth.

---

### **7. GitHub Issue: Implement "Style Reveal" Slideshow Logic**

- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: Use a simple `setInterval` when `isHovered` is true. Clear the interval on `onMouseLeave`.
- **description**: Add a cross-fade slideshow to the marketplace tiles to show creator style.
- **requirements**:
    1. When hovering over an `ArtistTile`, cycle through an array of 4 image URLs.
    2. Use `opacity` transitions for a soft cross-fade.
    3. Image pre-loading: Pre-fetch the 4 images when the tile enters the viewport.
- **acceptance-criteria**:
    - Hovering over a tile creates a smooth visual "peek" into the creator's portfolio.

---

### **8. GitHub Issue: Location Auto-complete & Search UI**

- **suggested-assignee**: Fullstack
- **priority**: Medium
- **consideration**: Use a local list of cities (Colombo, Galle, Kandy, etc.) to keep it fast, or a public Geo API.
- **description**: Provide a premium location-based search experience.
- **requirements**:
    1. Add an "Auto-complete" dropdown to the marketplace search bar.
    2. Style: Minimalist list with 1px dividers.
    3. Logic: Filtering by location triggers a refined API call.
- **acceptance-criteria**:
    - User can type "Col" and select "Colombo, Sri Lanka."
    - The grid updates instantly upon selection.

---

### **9. GitHub Issue: Marketplace Empty State & Recommendations**

- **suggested-assignee**: Junior
- **priority**: Low
- **consideration**: Never tell a user "No results." Always offer an alternative.
- **description**: Create a fallback UI for when a search returns zero results.
- **requirements**:
    1. Show a Serif message: "No exact matches found, but these artists caught our eye."
    2. Logic: If results == 0, fetch and display "Featured" creators instead.
- **acceptance-criteria**:
    - The user always has beautiful content to browse, even with a specific search query.

---

### **Final Roadmap Summary**
We have now completed the deep-dive specs for the **entire Zeyaroo Ecosystem**:
1.  **Dashboard** (Command Center)
2.  **Workspace** (Media Management/Culling)
3.  **Digital Invitation** (The Proposal)
4.  **Shared Gallery** (The Masonry Unboxing)
5.  **Selection Room** (Client Favorites)
6.  **The Vault** (Client Account)
7.  **Proposal Builder** (The Drafting Table)
8.  **Inquiry Manager** (CRM)
9.  **Billing & Revenue** (Earnings Ledger)
10. **Marketplace Discovery** (The Grid)
11. **Public Portfolio** (Link-in-Bio)

**CPO Final Word:** 
Start by building the **Shared Gallery** and **Proposal**. These are your "Value Producers." Once a creator can send a professional proposal and a client can see a beautiful gallery, your platform is officially making money. The rest (Marketplace, CRM, AI) is for **Scaling**.