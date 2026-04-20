This is the definitive **Zeyaroo Visual Identity & UI Style Guide**. It is designed to be pasted into your project's `UI_GUIDE.md` or a Notion/Wiki page for the development team.

***

# Zeyaroo Visual Identity & UI Style Guide
**Project:** Premium Photography & Media Suite  
**Version:** 1.0  
**Design Philosophy:** "The Editorial Lens"

---

## 1. Design Philosophy
Our design language synthesizes high-end print journalism (**Vogue/Kinfolk**) with modern digital productivity (**Apple/Lightroom**).

### Core Pillars
*   **Negative Space as Luxury:** White space is a deliberate design element that allows photography to breathe.
*   **Dual-Context Mode:** 
    *   *Gallery White:* A bright, airy experience for Client-facing galleries and proposals.
    *   *Neutral Obsidian:* A deep, focused interface for Creator-facing management tools.
*   **Tactile Sophistication:** Interactions must feel weighted. Use hairline borders and glassmorphism instead of heavy drop shadows.

---

## 2. Color Palette
All grays must have **0% saturation** to ensure they do not clash with the color grading of professional photography.

### 2.1 Core Neutrals
| Name | Hex | Usage |
| :--- | :--- | :--- |
| **Gallery White** | `#FCFCFC` | Primary background (Light Mode). |
| **Pure White** | `#FFFFFF` | Cards & elevated surfaces (Light Mode). |
| **Soft Stone** | `#F0F0F0` | Hairline borders & secondary text (Light Mode). |
| **Deep Charcoal** | `#1A1A1A` | Primary text & buttons (Light Mode). |
| **Obsidian Black** | `#0A0A0A` | Dashboard background (Dark Mode). |
| **Studio Grey** | `#121212` | Cards & surfaces (Dark Mode). |
| **Border Dark** | `rgba(255,255,255,0.05)` | Hairline borders (Dark Mode). |

### 2.2 Accents
| Name | Hex | Usage |
| :--- | :--- | :--- |
| **Zeyaroo Accent** | `#1A1A1A` | Default (swappable per creator branding). |
| **Heritage Gold** | `#D4AF37` | Premium actions (e.g., Hand-over). |
| **Success Emerald**| `#10B981` | Paid milestones / Active status. |
| **Warning Amber** | `#F59E0B` | Disputes / Pending actions. |

---

## 3. Typography
**Rule:** "Serif for Soul, Sans for Function."

### 3.1 Typefaces
*   **Primary Display (Serif):** *Instrument Serif* (or *Playfair Display*).
    *   *Usage:* Headlines, Project Titles, Client Greetings.
*   **Interface Label (Sans):** *Inter* (or *Instrument Sans*).
    *   *Usage:* Data, Buttons, Navigation, System labels.

### 3.2 Type Scale
*   **Hero Display:** `96pt` / `-0.02em` Tracking / Serif.
*   **Section Header:** `32pt` / `-0.01em` Tracking / Serif.
*   **Body Copy:** `16pt` / `1.6` Line Height / Sans.
*   **Data Label:** `10pt` / `0.2em` Tracking / All-Caps / Sans Bold.

---

## 4. Components & Elements
We distinguish between **Editorial** (Client) and **Workspace** (Creator) contexts.

### 4.1 Buttons
*   **Editorial Style (Client-Facing):**
    *   `border-radius: 0px` (Sharp corners).
    *   `font-family: Sans` with `tracking-widest`.
    *   *Context:* "Accept Proposal", "View Gallery", "Pay Milestone".
*   **Workspace Style (Creator-Facing):**
    *   `border-radius: 12px` (Soft corners).
    *   `font-weight: Semi-bold`.
    *   *Context:* "Upload", "Add Album", "Settings".

### 4.2 Imagery (The Sacred Color Rule)
**Never apply color-altering filters (Grayscale/Sepia) to a Creator's work.**
*   **Idle State:** `opacity: 0.8`.
*   **Hover/Active State:** `opacity: 1.0`, `scale: 1.05`, `transition: 500ms`.

### 4.3 Borders & Glassmorphism
*   **Hairline Borders:** Always `1px`.
*   **Light Mode Glass:** `bg-white/70 backdrop-blur-md border border-white/40 ring-1 ring-black/5`.
*   **Dark Mode Glass:** `bg-black/40 backdrop-blur-md border border-white/10`.

---

## 5. Motion & Interaction

| Context | Transition Speed | Easing |
| :--- | :--- | :--- |
| **Editorial (Public)** | `1000ms` | `cubic-bezier(0.4, 0, 0.2, 1)` |
| **Workspace (Private)** | `300ms` | `ease-out` |

*   **Page Entry:** Fade-in with a `2%` upward slide.
*   **Button Press:** Use "Spring" physics (Stiffness 400, Damping 25).

---

## 6. Implementation Snippets

### 6.1 Tailwind Configuration
```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        surface: {
          primary: 'var(--bg-app)',
          card: 'var(--bg-card)',
        },
        text: {
          primary: 'var(--text-main)',
          muted: 'var(--text-muted)',
        },
        border: {
          subtle: 'var(--border-main)',
        }
      },
      fontFamily: {
        serif: ['"Instrument Serif"', 'serif'],
        sans: ['Inter', 'sans-serif'],
      },
      borderRadius: {
        'editorial': '0px',
        'workspace': '12px',
      }
    },
  },
}
```

### 6.2 CSS Variables
```css
/* index.css */
:root {
  --bg-app: #FCFCFC;
  --bg-card: #FFFFFF;
  --text-main: #1A1A1A;
  --text-muted: #737373;
  --border-main: #F0F0F0;
}

.dark {
  --bg-app: #0A0A0A;
  --bg-card: #121212;
  --text-main: #FCFCFC;
  --text-muted: #A3A3A3;
  --border-main: rgba(255,255,255,0.05);
}
```