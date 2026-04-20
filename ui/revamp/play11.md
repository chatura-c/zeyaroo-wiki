This is **Deep Dive 11: The Public Portfolio (Artist Bio)**.

This is the creator's digital storefront. While the *Shared Gallery* is for a specific client, the *Public Portfolio* is for the **world**. It must be optimized for "Link-in-Bio" traffic (Instagram/TikTok) and designed to convert visitors into inquiries in seconds.

---

### **1. Wireframe Layout (Desktop & Mobile)**
A **Top-Down Visual Narrative** (Focus on the Creator's "Eye").

*   **The Signature Header (Top):**
    *   **Avatar:** Large, high-resolution portrait on the left.
    *   **Identity:** Name in `font-serif` (3xl), followed by the "Verified" shield.
    *   **Bio:** A single, elegant paragraph of text (max 300 chars).
    *   **Social Cluster:** Minimalist icons for Instagram, LinkedIn, and personal website.
*   **The Services Bar (Middle):**
    *   A horizontal scrolling set of tags: [Wedding] [Editorial] [Film Noir] [Commercial].
*   **The Curator's Grid (Center Stage):**
    *   The **Masonry Gallery** (Deep Dive 4), but populated with the creator's "Masterpieces" (from their Studio Project).
    *   Images feature a "Project Name" label on hover.
*   **The Trust Footer:**
    *   "Member since 2024" • "12 Projects Delivered" • "Based in Colombo."
*   **The Action Anchor (Sticky):**
    *   **Desktop:** A floating card on the right.
    *   **Mobile:** A fixed glassmorphic bar at the bottom: **[Send Inquiry]**.

---

### **2. Functional UX & Interactions**
*   **Smooth Entry:** The profile content should fade in from the bottom as the user scrolls (`opacity` and `translateY`).
*   **Image Lightbox:** Clicking any portfolio image opens the **Luxury Lightbox** (Topic 8) so the client can see the quality up close.
*   **The "Inquiry" Handshake:** Clicking the CTA opens a simplified, 3-step modal that captures: 
    1.  Event Type
    2.  Date 
    3.  Email.

---

### **3. Visual Polish (CSS/Tailwind Specs)**

#### **The Profile Header**
```tailwind
/* Background: Neutral Silk Texture or solid Off-White */
bg-surface-primary p-12 lg:p-24 flex flex-col items-center text-center

/* Avatar */
w-32 h-32 lg:w-48 lg:h-48 rounded-full object-cover border-4 border-white shadow-2xl mb-8
```

#### **The "Inquire" Sticky Bar (Mobile)**
```tailwind
/* High-key White Style */
fixed bottom-0 left-0 right-0 p-4 bg-white/70 backdrop-blur-xl border-t border-gray-100 z-50
/* Button */
w-full py-4 bg-gray-900 text-white font-semibold rounded-2xl shadow-lg active:scale-95 transition-transform
```

#### **Typography**
*   **Creator Name:** `font-serif text-5xl text-gray-900 tracking-tighter`
*   **Bio Text:** `font-sans text-lg leading-relaxed text-gray-600 max-w-xl mx-auto`

---

### **4. Edge Cases**
*   **Minimal Portfolio:** If a creator has < 3 images, the grid hides and the bio section expands to fill the screen to avoid a "lonely" look.
*   **No Location:** If the location is hidden, the Trust Footer skips the "Based in..." segment and centers the "Projects Delivered" stat.
*   **Creator "Away" Mode:** If the creator is not accepting inquiries, the CTA button changes to: *"Not currently accepting new projects"* and becomes a "Join Waitlist" email field.

---

### **5. GitHub Issue: Implement Public Portfolio API (Backend)**

- **suggested-assignee**: Backend
- **priority**: High
- **consideration**: Ensure that the "Portfolio Media" is pulled specifically from the project designated as the "Studio Project" (Topic 1).
- **description**: Build the public endpoint to fetch a creator's profile and their featured portfolio media.
- **requirements**:
    1. Endpoint: `GET /api/v1/public/creators/:id_or_slug`.
    2. Join `CreatorProfile` with the `User` and `Media` tables.
    3. Filter media by the `is_portfolio: true` flag or specific "Portfolio" album ID.
    4. Include "Trust Stats" (Member since, project count).
- **acceptance-criteria**:
    - The API returns all data needed for the header, bio, and masonry grid in a single request.
    - Profiles are accessible via both the UUID and the Vanity Slug (if set).

---

### **6. GitHub Issue: Build Responsive Public Portfolio UI**

- **suggested-assignee**: Frontend
- **priority**: High
- **consideration**: This page will be viewed mostly via Instagram's in-app browser. It must be extremely lightweight and fast.
- **description**: Implement the "Signature" layout for creator profiles.
- **requirements**:
    1. Build the `PortfolioHeader` with large typography and social links.
    2. Integrate the `MasonryGrid` component used in the Shared Gallery.
    3. Create the `ServiceTags` horizontal scroller.
    4. Implement the `StickyInquiryBar` for mobile users.
- **acceptance-criteria**:
    - Page looks like a high-end personal website.
    - Masonry grid correctly handles mixed image orientations.
    - Mobile bottom-bar remains visible and functional.

---

### **7. GitHub Issue: Simplified "Lead Capture" Modal**

- **suggested-assignee**: Frontend
- **priority**: Medium
- **consideration**: Use "Step-based" navigation (Step 1: What? -> Step 2: When? -> Step 3: Who?) to make the form feel shorter.
- **description**: Create a high-conversion inquiry form specifically for the Public Portfolio.
- **requirements**:
    1. Modal: `PublicInquiryModal.tsx`.
    2. Minimal inputs: Name, Email, Date, Brief Description.
    3. Success state: "Inquiry Sent. [Creator] will be in touch shortly."
- **acceptance-criteria**:
    - Clicking the sticky "Inquire" button triggers the modal.
    - Submitting the form creates a "New" lead in the creator's CRM (Deep Dive 8).

---

### **8. GitHub Issue: Implement "Hire Again" Logic for Clients**

- **suggested-assignee**: Fullstack
- **priority**: Low
- **consideration**: This is a retention feature for the platform's ecosystem.
- **description**: Track relationships between clients and creators to suggest them later in the Client Vault.
- **requirements**:
    1. Add a `CreatorHistory` section to the Client Vault (Deep Dive 6).
    2. Display the creator's avatar and a "Hire Again" link pointing to this portfolio page.
- **acceptance-criteria**:
    - Clients can easily find and re-hire creators they have previously released payments to.

---

**Next **Deep Dive 12: The Marketplace (Creator Discovery)**? 