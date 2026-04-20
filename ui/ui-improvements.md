# UI Improvements

## Typography
- [ ] Load Cormorant Garamond (or similar serif) for display headings — currently all font aliases (cormorant, serif, display) just map to Inter, so the editorial feel promised by the design system never lands
- [ ] Use a different weight/style pairing within Inter as a fallback if serif loading is deferred

## Dashboard Stat Cards
- [ ] Differentiate the three cards — earnings, storage, leads are all identical style with no visual hierarchy
- [ ] Make one the hero metric (e.g. total earned)
- [ ] Circular progress charts are too thin and barely visible — increase stroke weight

## Project Cards
- [ ] Dark gradient overlay is too heavy — image behind is invisible
- [ ] Fallback for no-image state should be intentional (pattern, texture, or color field) rather than a plain grey gradient
- [ ] "ACTIVE" label + title text is small and hard to scan — bump size and contrast

## Milestone Progress Card
- [ ] Stat chips (Completed / In Progress / Pending / Total) have no visual hierarchy — Total should anchor as the primary number
- [ ] Give each status a more distinct visual treatment

## Status Badges
- [ ] "In Progress" amber badge is too saturated vs the rest of the muted palette — desaturate or soften
- [ ] Review all status badge colors for consistency with the surface palette

## Sidebar
- [ ] Active icon state needs stronger visual treatment — left border alone is too subtle, especially in obsidian/dark mode
- [ ] Consider active icon fill or background pill

## Tab Navigation (Media / Chapters / Brief / Shares)
- [ ] Tab underline indicator is too thin and low-contrast
- [ ] Increase tab bar height and use slightly bolder weight for active tab label

## Page-level Hierarchy
- [ ] Dashboard greeting ("Good morning, April.") has no typographic contrast — use serif/display font here for a premium moment

## Grain Overlay
- [ ] Current opacity is 0.045 — nearly invisible; bump to 0.06–0.08 for the tactile film grain feel the code intends

## Primary CTA Button
- [ ] "+ New Project" uses plain black — should use gold accent to tie brand language together

## Theme — Too White / Monotone
- [ ] surface-primary (#F9F9F7) and surface-card (white) are nearly identical — no depth between layers
- [ ] Consider a warm tint on surface-muted (very subtle warm gray, not cool) to add dimension
- [ ] Introduce a second accent color or a richer use of gold beyond just badges/borders
- [ ] Try shifting the primary background very slightly warm (e.g. #F7F5F0) to feel less clinical
- [ ] Use gold as a subtle tint on active/hover states throughout — not just on specific components
- [ ] Add depth via shadow variation — currently shadows are very minimal; layering them creates perceived richness without adding color
- [ ] Consider a subtle warm-tinted surface for the sidebar (slightly darker + warm) to separate it from content area
