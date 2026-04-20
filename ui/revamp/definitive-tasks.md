To ensure these tasks are definitive and eliminate guesswork, I have mapped them directly to your existing file structure and provided specific technical specifications (hex codes, font names, and implementation logic).

---As a CPO and Architect, I believe **a Light (White) Theme is the superior choice for a luxury media brand.** 

While dark mode is "cool" for developers, **Light Mode** (think Apple, Chanel, or high-end bridal magazines like *Vogue Weddings*) feels cleaner, more expensive, and more "airy." Most importantly, **white is a neutral color temperature.** In dark mode, the UI's contrast can make a photographer's shadows look darker than they are. In a light "Gallery White" theme, the photos are the undisputed stars of the show.

Here is the **revised, definitive task list for Topic 8**, pivoted to a **"High-Key Gallery" (White) Theme**.

---

### **Task 1: Transition to "Gallery White" Palette**
- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Use "Off-White" instead of pure `#FFFFFF` for the background to reduce eye strain and make white images stand out.
- **description**: Replace the dark obsidian theme with a minimalist, high-key light theme.
- **requirements**:
    1. Update `phto-ui/tailwind.config.js`:
        - Set `surface-950` (Background) to `#FCFCFC`.
        - Set `surface-900` (Cards) to `#FFFFFF`.
        - Set `surface-800` (Hover/Secondary) to `#F5F5F5`.
        - Set `text-primary` to `#1A1A1A` (Near Black).
        - Set `text-muted` to `#737373`.
    2. Add a `border-subtle` class: `#E5E5E5`.
- **acceptance-criteria**:
    - The entire application background is off-white.
    - All text is dark gray/black on the light background.
    - Blue "ambient glows" are removed entirely.

---

### **Task 2: "Editorial" Typography & Letter Spacing**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: Pair the Serif font with slightly increased letter spacing (`tracking-wide`) for a premium feel.
- **description**: Implement "Instrument Serif" for headings to create an editorial/magazine vibe.
- **requirements**:
    1. Add Google Font to `phto-ui/index.html`: `Instrument+Serif`.
    2. Update `tailwind.config.js` font family: `serif: ['"Instrument Serif"', 'serif']`.
    3. Apply `font-serif tracking-tight` to all `h1` and `h2` elements in:
        - `src/pages/ProposalDetail.tsx`
        - `src/pages/CreatorProfile.tsx`
        - `src/components/ui/CardTitle.tsx`
- **acceptance-criteria**:
    - Headings have the high-contrast, elegant look of a luxury publication.

---

### **Task 3: Responsive Masonry Gallery implementation**
- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: Ensure that when an image is clicked, the transition to the full-screen lightbox is smooth.
- **description**: Replace the rigid square grid with a masonry layout to accommodate different photo orientations.
- **requirements**:
    1. Edit `phto-ui/src/pages/SharedGallery.tsx`.
    2. Use CSS Columns for the grid container:
       ```css
       .masonry-grid {
         column-count: 1; /* Mobile */
         column-gap: 1.5rem;
       }
       @screen sm { .masonry-grid { column-count: 2; } }
       @screen lg { .masonry-grid { column-count: 3; } }
       ```
    3. Items must use `break-inside: avoid` and `margin-bottom: 1.5rem`.
- **acceptance-criteria**:
    - Portrait photos take up more vertical space than landscape photos.
    - No gaps appear in the layout; photos flow naturally like an art gallery wall.

---

### **Task 4: High-Impact Hero Sections for Proposals**
- **suggested-assignee**: Fullstack
- **priority**: High
- **consideration**: Use a "Scrim" overlay (a light gradient from transparent to white) at the bottom so text doesn't get lost.
- **description**: Allow a large cinematic cover image to anchor the top of a project or proposal.
- **requirements**:
    1. **Backend**: Ensure `models.Proposal` has `CoverImageID`.
    2. **Frontend**: In `ProposalDetail.tsx`, create a top hero section:
       - `h-[60vh] w-full relative`.
       - Text (Title/Date) should be bottom-aligned, using `font-serif` and `text-4xl`.
       - Gradient Overlay: `bg-gradient-to-t from-[#FCFCFC] via-transparent to-transparent`.
- **acceptance-criteria**:
    - The proposal starts with a large, professional image that blends into the white page below.

---

### **Task 5: Floating Glass "Upload Status" Tray**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: In light mode, glassmorphism needs a light background blur and a subtle shadow (`shadow-xl`).
- **description**: A persistent floating indicator that shows upload progress across all pages.
- **requirements**:
    1. Implement `src/components/UploadTray.tsx`.
    2. Style: `bg-white/70 backdrop-blur-md border border-gray-200 shadow-2xl rounded-2xl p-4`.
    3. Fixed Position: `bottom-8 right-8 w-80`.
    4. Features: ProgressBar and "X of Y files" text.
- **acceptance-criteria**:
    - Users can see their upload progress without staying on the Project page.

---

### **Task 6: "Luxury Minimalist" Empty States**
- **suggested-assignee**: Junior
- **priority**: Low
- **consideration**: Avoid "clutter." Use 1 icon, 1 Serif title, 1 short description.
- **description**: Standardize the look of pages when no data is present.
- **requirements**:
    1. Create `src/components/ui/EmptyState.tsx`.
    2. Use a thin-stroke (1px) icon set like *Lucide* or *HeroIcons*.
    3. Heading must use `font-serif text-2xl text-gray-900`.
- **acceptance-criteria**:
    - Pages with no items feel "intended" and professional, not empty.

---

### **Task 7: White-Themed Media Skeletons**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: Skeletons should be slightly off-white (`#F3F3F3`) to be visible against the white background.
- **description**: Visual placeholders to prevent layout jank during loading.
- **requirements**:
    1. Create `src/components/ui/MediaSkeleton.tsx`.
    2. Use `bg-gray-100 animate-pulse rounded-2xl`.
    3. Randomize the heights (e.g., `h-64`, `h-80`, `h-48`) to simulate the masonry look.
- **acceptance-criteria**:
    - Loading states look like the final gallery layout, preventing sudden movement.

---

### **Task 8: Luxury Watermark Overlay UI**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: The badge should look like a "Seal of Quality."
- **description**: Explicitly label preview images to drive milestone funding.
- **requirements**:
    1. In `SharedGallery.tsx`, add a top-right badge to watermarked images.
    2. Style: `bg-white/90 text-gray-900 text-[9px] font-bold tracking-tighter px-2 py-0.5 rounded-full border border-gray-200 shadow-sm`.
    3. Content: "PREVIEW ONLY".
- **acceptance-criteria**:
    - The client immediately understands that the low-quality image is temporary.

---

### **Task 9: Hairline Border & Soft Shadow Design System**
- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: In a white theme, borders should be very faint (`border-gray-100`). Use shadows to create depth instead.
- **description**: Standardize the "container" look across the app.
- **requirements**:
    1. Update `src/components/ui/Card.tsx`:
       - Remove heavy borders.
       - Use `bg-white shadow-[0_8px_30px_rgb(0,0,0,0.04)] rounded-3xl`.
       - Add a 1px border: `border-[#F0F0F0]`.
- **acceptance-criteria**:
    - UI elements feel "light" and floating, not "boxed in" by heavy lines.

---

### **Task 10: "Clean" Notification Bell & User Menu**
- **suggested-assignee**: Frontend
- **priority**: Low
- **consideration**: The active state of the bell should be a subtle dot in your "Brand Accent" color.
- **description**: Refactor the top-right navigation components for the light theme.
- **requirements**:
    1. Bell Icon: `text-gray-400 hover:text-gray-900`.
    2. Notification Dropdown: `bg-white shadow-2xl border border-gray-100 rounded-2xl`.
    3. User Avatar: 1px gray border to prevent "white-on-white" invisibility.
- **acceptance-criteria**:
    - Navigation is functional and disappears into the background until needed.

---

### **Task 11: Accent Color Implementation (Variable Buttons)**
- **suggested-assignee**: Fullstack
- **priority**: High
- **consideration**: Test that white text on the accent color is readable (Accessibility check).
- **description**: Set the stage for creator-specific branding.
- **requirements**:
    1. **Backend**: Add `PrimaryColor string` to `CreatorProfile` (default: `#1A1A1A`).
    2. **Frontend**: In `App.tsx` or `Layout.tsx`, inject a global CSS variable: 
       ```css
       :root { --brand-primary: ${profile.primary_code}; }
       ```
    3. Use `bg-[var(--brand-primary)]` for "Release Payment," "Fund," and "Accept" buttons.
- **acceptance-criteria**:
    - Public pages use the creator's color for primary actions.
    - The default is a classy, luxury black (`#1A1A1A`).

---

### **Verdict for the Developer:**
The "White Theme" is all about **restraint.** Use lots of white space, thin gray lines, and serif fonts. Let the photographs provide all the color. This looks 10x more expensive than a dark theme.


### **Task 12: Implement Theme Engine (Context & Persistence)**
- **suggested-assignee**: Fullstack
- **priority**: High
- **consideration**: Use the `class` strategy for Tailwind. This allows us to force a specific theme on public pages (e.g., Creator wants a white gallery) regardless of the visitor's system settings.
- **description**: Create a centralized Theme Provider to manage the transition between "Gallery White" and "Obsidian Dark."
- **requirements**:
    1. Create `phto-ui/src/contexts/ThemeContext.tsx`.
    2. Logic: Initialize state from `localStorage` (key: `zeyaroo-theme`). If empty, detect system preference using `window.matchMedia`.
    3. Effect: Toggle the `.dark` class on the `document.documentElement` (the `<html>` tag).
    4. Provide a `toggleTheme` function to the rest of the app.
- **acceptance-criteria**:
    - Refreshing the page remembers the chosen theme.
    - Manually changing the `<html>` class to `dark` updates the UI instantly.

---

### **Task 13: Abstract Design Tokens to CSS Variables**
- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: This is the "Definitive" part. Developers should now use `bg-surface-primary` instead of `bg-white` or `bg-surface-950`.
- **description**: Map the Tailwind configuration to CSS variables defined in the global CSS file.
- **requirements**:
    1. Update `phto-ui/src/index.css` to define tokens:
       ```css
       :root { /* Light Theme */
         --bg-app: #FCFCFC;
         --bg-card: #FFFFFF;
         --text-main: #1A1A1A;
         --border-main: #F0F0F0;
       }
       .dark { /* Dark Theme */
         --bg-app: #0A0A0A;
         --bg-card: #121212;
         --text-main: #FCFCFC;
         --border-main: #1F1F1F;
       }
       ```
    2. Update `tailwind.config.js` to reference these variables (e.g., `colors: { surface: { primary: 'var(--bg-app)' } }`).
- **acceptance-criteria**:
    - Changing the variable in `index.css` updates the color across both themes.

---

### **Task 14: Minimalist Theme Toggle UI**
- **suggested-assignee**: Junior
- **priority**: Low
- **consideration**: Use Lucide-React icons `Sun` and `Moon`. Keep the toggle subtle—place it in the user profile dropdown.
- **description**: Add a visual control for the user to switch themes.
- **requirements**:
    1. Update `phto-ui/src/components/Layout.tsx`.
    2. Add a toggle button in the navigation or the profile dropdown menu.
    3. Use a "Switch" component from HeadlessUI for a premium feel.
- **acceptance-criteria**:
    - Clicking the toggle flips the UI and updates `localStorage`.

---

### **Task 15: Per-User Theme Preference (Backend Sync)**
- **suggested-assignee**: Backend
- **priority**: Medium
- **consideration**: This ensures that if a creator logs in on a new device, their "Studio" preference is already set.
- **description**: Add a persistent theme preference to the user's database record.
- **requirements**:
    1. **Models**: Add `PreferredTheme string` (light/dark/system) to `models.User`.
    2. **Handlers**: Update the `UpdateUser` endpoint to accept this field.
    3. **Frontend**: On login/app-mount, if the user is authenticated, sync the `ThemeContext` state with the value from the API.
- **acceptance-criteria**:
    - Changing the theme on Desktop and then logging in on Mobile (browser) shows the same theme preference.

---

### **Task 16: "Force Theme" Logic for Public Pages**
- **suggested-assignee**: Senior
- **priority**: Medium
- **consideration**: This is a key business feature. A creator might design their brand for "Dark Mode" and doesn't want a client's "Light Mode" system setting to ruin the artistic look of the gallery.
- **description**: Implement logic to override the visitor's theme with the Creator's brand preference on public-facing pages.
- **requirements**:
    1. **Backend**: Add `PublicTheme string` (light/dark) to `models.CreatorProfile`.
    2. **Frontend**: In `SharedGallery.tsx` and `PublicProposal.tsx`, use a `useEffect` to ignore the user's `localStorage` theme and apply the `profile.public_theme` class to a local wrapper div or the `html` tag temporarily.
- **acceptance-criteria**:
    - If Creator A sets their public theme to "Dark," any client visiting their link sees the "Obsidian" theme, even if the client's own computer is set to Light mode.

---

### **Summary of Theme Logic for Developers:**
*   **Default State:** All components should use Tailwind's `dark:` prefix or the new variable names.
*   **Variable Usage:** 
    *   Instead of `text-gray-900`, use `text-text-primary`.
    *   Instead of `bg-white`, use `bg-surface-900`.
    *   Instead of `border-gray-200`, use `border-subtle`.