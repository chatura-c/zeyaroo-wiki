This is **Deep Dive 10: Marketplace & Discovery (The Creator Search)**.

This is the bridge between the world and the platform. Unlike a standard directory, Zeyaroo’s discovery must feel like **curating a guest list for a high-end gala**. Every creator should look like an elite artist, and the search experience must be fluid and visual.

---

### **1. Wireframe Layout (Desktop)**
A **Visual-First Grid** with a **"Soft Filter" Sidebar**.

*   **The Curated Header (Top):**
    *   A massive, elegant search bar centered at the top.
    *   Serif text: "Find the eye that sees your story."
*   **The Filter Wing (Left - 3 cols - Sticky):**
    *   **Categories:** Wedding, Editorial, Commercial, Film.
    *   **Location:** City-based search (e.g., Colombo, Dubai, London).
    *   **Pricing:** [Budget] | [Mid-Range] | [Premium].
    *   **The "Verified Only" Toggle:** High-end switch for vetted creators.
*   **The Artist Gallery (Right - 9 cols):**
    *   A grid of **"Artist Tiles"**.
    *   Each tile is not just a name, but a **style preview**.
    *   Layout: Alternating tile sizes (1x1 and 2x1) to create a high-fashion magazine rhythm.

---

### **2. Functional UX & Interactions**
*   **The "Style Reveal" Hover:** Hovering over an Artist Tile replaces the static cover with a fast-fading slideshow of their 4 best portfolio shots.
*   **Infinite "Artistic" Scroll:** As the user scrolls, new tiles fade in with an offset delay (`0.1s`, `0.2s`, `0.3s`).
*   **Instant Filter:** Selecting a category (e.g., "Wedding") instantly filters the grid without a page reload (using React Query's `keepPreviousData` for smoothness).

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Artist Tile (The Discovery Card)**
```tailwind
/* Container */
relative aspect-[3/4] rounded-3xl overflow-hidden bg-surface-800 group transition-all duration-700

/* The Main Cover */
w-full h-full object-cover transition-transform duration-[1.5s] group-hover:scale-110

/* Bottom Gradient Info */
absolute bottom-0 left-0 right-0 p-8 bg-gradient-to-t from-black/80 via-black/20 to-transparent
opacity-100 transition-opacity duration-500

/* Text */
font-serif text-2xl text-white tracking-tight /* Creator Name */
font-sans text-[10px] uppercase tracking-widest text-white/60 /* Location */
```

#### **The Elegant Search Bar**
```tailwind
/* Centered Pill */
w-full max-w-xl mx-auto bg-white/50 backdrop-blur-xl border border-gray-200 
rounded-full px-8 py-4 shadow-sm focus-within:shadow-xl focus-within:border-accent-primary 
transition-all
```

---

### **4. Edge Cases**
*   **No Results:** Show a "Personalized Recommendation" empty state: *"We couldn't find a exact match. How about these featured artists?"*
*   **No Portfolio Images:** If a creator has no portfolio, the tile shows a sophisticated "Studio Portrait" (Avatar) on a neutral silk-textured background.
*   **Low Bandwidth:** Tiles load a 10px blurred "L-Preview" (LQIP) first before the high-res cover arrives.

---

### **5. GitHub Issue: Implement Search & Discovery API (Backend)**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Use `ILIKE` for location/name search and GORM's `.Where()` for category arrays.
- **description**: Build the search engine for the marketplace.
- **requirements**:
    1. Endpoint: `GET /api/v1/public/creators` (Filterable).
    2. Support query params: `category`, `city`, `min_price`, `max_price`, `verified=true`.
    3. Logic: Only return creators who have `is_public: true` and at least 3 portfolio images.
    4. Implement "Weighted Sorting": Verified creators and those with recent activity appear first.
- **acceptance-criteria**:
    - API returns a list of creator profiles with their 4 most recent portfolio image URLs.
    - Searching "Colombo" returns only artists in that location.

---

### **6. GitHub Issue: Build Curated Discovery Grid UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `CSS Grid` with `grid-auto-flow: dense` to support the alternating tile sizes (Topic 8 visual language).
- **description**: Build the main marketplace discovery page.
- **requirements**:
    1. Implement the responsive Artist Grid with alternating tile sizes.
    2. Create the `ArtistTile` component with the "Slideshow Hover" effect.
    3. Build the `SidebarFilters` component (Sticky/Fixed).
    4. Implement the "Style Reveal" animation using `framer-motion`.
- **acceptance-criteria**:
    - The grid feels like a magazine spread, not a boring database.
    - Hovering over an artist tile triggers a subtle slideshow of their work.
    - Responsive: On mobile, sidebar moves to a "Filter" button that opens a bottom drawer.

---

### **7. GitHub Issue: Implement "Quick Inquire" Marketplace Shortcut**

- **suggested-assignee**: Junior
- **priority**: Medium
- **consideration**: Keep the form extremely short (3 fields) to maximize "Impulse Inquiries."
- **description**: Add a button on each Artist Tile to inquire immediately without leaving the search results.
- **requirements**:
    1. Add an "Inquire" icon/button to the `ArtistTile` (visible on hover).
    2. Trigger the `InquiryDialog` (Topic 6) in a modal overlay.
- **acceptance-criteria**:
    - Client can send an inquiry from the search grid in under 3 clicks.
    - User stays on the search page after sending the inquiry.

---

### **8. GitHub Issue: Location-Based Auto-Complete**

- **suggested-assignee**: Frontend
- **priority**: Low
- **consideration**: Use a fixed list of supported cities for V1 or integrate with Google Places API later.
- **description**: Provide a high-end dropdown for the location search.
- **requirements**:
    1. Update the search bar to show a list of cities as the user types.
    2. Style: Minimalist list with Serif city names.
- **acceptance-criteria**:
    - Typing "Col" suggests "Colombo, Sri Lanka."

---

**Next Deep Dive 11: The Public Portfolio (Artist Bio)**