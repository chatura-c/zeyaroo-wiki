This is **Deep Dive 1: The Creator Command Center (Dashboard)**.

The Dashboard is no longer a collection of boxes; it is a **high-end studio desk**. It should feel calm, organized, and prioritize the creator's best imagery.

---

### **1. Wireframe Layout (Desktop)**
We use a **12-column grid** with a fixed minimalist sidebar.

*   **Sidebar (Cols 1-2):** Slim icons and text labels. Deeply inset borders.
*   **Main Stage (Cols 3-12):**
    *   **Header:** Greets the user ("Good morning, [Name]") in `font-serif`. Features a global "Quick Search" (Cmd+K).
    *   **Top Row (Stats):** Three large, glassmorphic cards:
        *   **Revenue Ring:** A circular thin-stroke chart showing "Funds in Escrow" vs "Total Earned."
        *   **Storage Ring:** A circular thin-stroke chart showing GB used.
        *   **Inquiry Pulse:** A number indicating active leads.
    *   **Middle Row (The Work):** A horizontal "Film Strip" of Active Projects.
        *   *Visual Hack:* Instead of 1 cover photo, show a **"Stack"** of 3 photos overlapping slightly to represent the project depth.
    *   **Bottom Row (Split):**
        *   **Left (2/3):** "Upcoming Milestones" list.
        *   **Right (1/3):** "Studio Activity" (Client interactions).

---

### **2. Functional UX & Interactions**
*   **The "Stack" Hover:** When hovering over a project stack, the images should slightly fan out (animate `rotate` and `translateX`).
*   **The Stats Pulse:** When the dashboard loads, the circular rings should draw themselves from 0% to X% over 1.2 seconds (`transition-all duration-1000`).
*   **One-Click Action:** Every project card has a tiny "+" button visible only on hover to "Quick Upload" without leaving the dashboard.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Page Background**
```tailwind
bg-surface-primary /* #FCFCFC / #0A0A0A */
```

#### **The Project Stack Card**
```tailwind
/* Container */
relative w-64 h-80 group cursor-pointer

/* Image 3 (Bottom) */
absolute bottom-4 left-4 rotate-[-6deg] opacity-40 blur-[1px]

/* Image 2 (Middle) */
absolute bottom-2 left-2 rotate-[-3deg] opacity-80

/* Image 1 (Top) */
relative w-full h-72 rounded-2xl object-cover shadow-xl group-hover:rotate-0 transition-transform duration-500
```

#### **Typography**
*   **Greeting:** `font-serif text-4xl tracking-tight text-text-primary`
*   **Stats Labels:** `font-sans text-[10px] uppercase tracking-[0.2em] text-text-muted`
*   **Data Numbers:** `font-sans font-light text-3xl text-text-primary`

---

### **4. Edge Cases**
*   **Zero State:** If no projects exist, show a single empty "Stack" silhouette with a dashed border and a Serif title: *"Your first gallery is waiting to be born."*
*   **Vertical vs Horizontal Covers:** The "Stack" logic automatically crops to a **4:5 aspect ratio** (the "Instagram Portrait" standard) to ensure consistency regardless of the original photo shape.
*   **Mobile View:** The sidebar moves to a bottom navigation bar. Stats become a swipeable carousel to save vertical space.

---

### **5. GitHub Issue: Implement Creator Command Center UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use `Framer Motion` for the stack fan-out animation. Ensure the "Storage Ring" uses a SVG path-dasharray for the animation.
- **description**: Build the main dashboard page using the "Gallery-First" luxury design language. Replace the current stat cards with high-impact visual components.
- **requirements**:
    1. Implement the 12-column grid layout with a sticky sidebar.
    2. Create the `ProjectStack` component with fanning hover animation.
    3. Implement `AnimatedMetricRing` component for Revenue and Storage.
    4. Integrate `Instrument Serif` for all primary headlines.
    5. Support for Theme Switching via CSS Variables.
- **acceptance-criteria**:
    - Dashboard renders correctly in both Light and Dark modes.
    - Hovering over a project stack triggers a smooth 3-image fan animation.
    - Metrics animate on page mount.
    - Responsive: On screens < 768px, sidebar becomes a bottom-nav.

---

**Next Deep Dive?**
Shall we move to **Deep Dive 2: The Project Workspace (Media Management & Culling)**? This is where creators spend 90% of their time.