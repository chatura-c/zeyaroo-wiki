To ensure the **Zeyaroo** redesign remains maintainable and high-end, developers must follow strict "Clean Code" standards. This prevents the "Visual Drift" that happens when multiple developers add arbitrary padding or hardcoded hex colors.

Add this section to your **`CONTRIBUTING.md`** or **`DEVELOPER_GUIDE.md`**.

***

# Zeyaroo Frontend Development Standards

## 1. Zero-Hardcoding Policy (Styling)
**Never** use arbitrary hex codes or Tailwind square-bracket notation (e.g., `bg-[#FCFCFC]`) in components.

*   **Semantic Classes:** Use the semantic names defined in `tailwind.config.js`.
    *   ✅ Good: `<div className="bg-surface-primary text-text-main">`
    *   ❌ Bad: `<div className="bg-[#FCFCFC] text-gray-900">`
*   **Spacing:** Follow the 8px grid. Use standard Tailwind scales (`p-4`, `m-8`, `gap-2`). If you need a custom value, define it in the config as a design token.

## 2. Context-Aware Components
Since we have two visual languages (**Editorial** and **Workspace**), shared components must be flexible.

*   **The "Context" Prop:** High-level components (Buttons, Inputs, Cards) should accept a `variant` prop.
    ```tsx
    interface ButtonProps {
      variant: 'editorial' | 'workspace';
      // ...
    }
    
    // Logic: Applies rounded-none for editorial, rounded-workspace for toolset
    const className = variant === 'editorial' ? 'rounded-none' : 'rounded-workspace';
    ```

## 3. Typography Architecture
Do not apply font families or sizes manually to every tag. Use a consistent hierarchy.

*   **Component-Based Heads:** Create a `Typography` component or use Tailwind Typography patterns.
    *   `font-serif`: Reserved **strictly** for H1, H2, and Quote elements.
    *   `font-sans`: Used for all UI labels, navigation, and body text.
*   **Tracking:** Always use `tracking-tight` for large Serif headings and `tracking-widest` + `uppercase` for small Sans-Serif labels.

## 4. Media & Performance Best Practices
Zeyaroo is a media-heavy app. Poor media handling kills the "Luxury" feel via layout shifts.

*   **Aspect Ratio Wrappers:** Always wrap images in a container with a defined aspect ratio to prevent Layout Shift (CLS).
    ```tsx
    <div className="aspect-[4/5] relative overflow-hidden bg-surface-muted">
       <img src={url} loading="lazy" className="object-cover" />
    </div>
    ```
*   **LQIP (Low-Quality Image Placeholders):** Always use a blurred placeholder or a specific color average (Topic 12) while the high-res image loads.
*   **Suspense & Skeletons:** Every data-fetching component must have a corresponding Skeleton state. A blank screen is a bug.

## 5. React & State Patterns
*   **Zustand for UI State:** Use small, specialized stores for UI-only state (e.g., `useUploadStore`, `useThemeStore`).
*   **React Query for Server State:** Never use `useEffect` to fetch data. Use the custom hooks in `src/api/`.
*   **Early Returns:** Keep components clean by returning loading/error states early.
    ```tsx
    if (isLoading) return <ProjectSkeleton />;
    if (error) return <ErrorMessage error={error} />;
    return <ActualUI data={data} />;
    ```

## 6. Layout Composition
*   **The 80/20 Spacing Rule:** When in doubt, add more padding. Luxury is defined by the space between elements.
*   **Consistent Modals:** Use the standard `Dialog` component from HeadlessUI. Ensure all modals utilize `backdrop-blur-md` for the overlay.
*   **Sticky Elements:** Navigation and Action bars should use `sticky` positioning with a high `z-index` and glassmorphism.

## 7. DOM & Accessibility
*   **Interactive Elements:** Ensure every clickable element has a `hover:`, `active:`, and `focus-visible:` state.
*   **Contrast:** Even in Dark Mode (Obsidian), ensure text-to-background contrast passes WCAG AA standards.
*   **Semantic HTML:** Use `<main>`, `<section>`, `<nav>`, and `<aside>` instead of a sea of `<div>` tags. This helps both SEO and AI crawlers (GEO).

## 8. Git & Task Flow
*   **Atomic Commits:** One task from the backlog = one pull request.
*   **Component Documentation:** If you create a new UI pattern, add it to the `src/components/ui/README.md`.

***

### Summary Checklist for PR Reviewers:
1.  Are there any hex codes in the code? (If yes, reject).
2.  Does the page shift when images load? (If yes, add aspect-ratio wrappers).
3.  Is the Serif font used for a button or a small label? (If yes, change to Sans).
4.  Does it work in both Light and Dark mode using CSS variables?