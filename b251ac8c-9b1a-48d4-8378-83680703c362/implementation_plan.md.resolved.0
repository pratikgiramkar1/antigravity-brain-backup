# Prism Page Redesign Implementation Plan

This document outlines the approach to complete the redesign of the `/prism` page based on your Figma reference and instructions. The goal is to create a dynamic, premium, highly polished web app representation of the design.

## User Review Required

> [!IMPORTANT]
> The changes represent a significant update to the visual structure of the page. Please review the *Open Questions* section before I begin, as your answers will guide the exact styling, especially since I do not have direct access to your Figma file.

## Proposed Changes

### 1. Brand-proof-band & Marquee Animation
- **Positioning Fix:** Adjust `margin-top` and stacking contexts (z-index) in `.brand-proof-band` so it seamlessly overlaps the hero video without blocking the CTA buttons.
- **Marquee:** Convert the static `.brand-strip` grid into an infinite CSS-based horizontal marquee animation (duplicated logos scrolling seamlessly).
- **Dark Logos:** Use purely CSS-based filters (e.g. `filter: brightness(0) opacity(0.7);` combined with a `mix-blend-mode` if needed) to turn the current images into "dark logos" to contrast against the light `#efefef` background of this section. 

### 2. Stats Section Light Effect
- **Styling:** Introduce a soft ambient glow (radial gradient) behind the stats cards. We will elevate the cards using soft Drop Shadows and subtle borders (glassmorphism) to enhance the "light effect".

### 3. "How to get started?" Section Redesign
- **Styling:** Revamp the code panel and tabs to look ultra-modern. We'll introduce a dark glassy aesthetic for the code window with macOS-like window controls, vibrant active-tab indicators, and a clean, high-contrast monospace typographic treatment.

### 4. "Switch Processors" New Section
- **Implementation:** Add a completely new section highlighting the ease of switching payment processors.
- **Design:** Using a split layout with engaging visuals (e.g., animated abstract nodes representing processors) on one side and compelling copy on the other.

### 5. FAQ Section Refinement
- **Styling:** Redesign the `<details>` elements into a custom, smooth-animated accordion. Features will include chevron rotation, distinct hover background highlights, and proper typography spacing that matches the premium aesthetic.

### 6. Custom Footer
- **Implementation:** Build a sleek, minimal dark footer specific to the Prism page (`bg-[#161616]`) with appropriate brand links and repository links, replacing or overriding any default layout footers if necessary.

### 7. Unused SVG Cleanup
- **Maintenance:** Scan and remove unused SVG and asset imports at the top of `src/pages/prism.astro` to clean up the bundle.

---

### Src / Pages

#### [MODIFY] src/pages/prism.astro
This will be the primary file updated to include all the new components, semantic markup, and the required inline CSS/JS adjustments. 

---

## Open Questions

> [!WARNING]
> Since I cannot view the Figma file directly, I need a bit of guidance on a few specifics to ensure I match your vision:

1. **Dark Logos:** Are you okay with me using CSS filters (e.g. `brightness(0)`) to make the current image logos dark, or do you plan to upload/import specific dark PNG/SVG assets for the marquee?
2. **Switch Processors Section:** Do you have specific copy (text) or a particular visual metaphor in mind for this section, or should I use my best judgment to create a premium placeholder design?
3. **How to get started Redesign:** Are there any specific visual queues from Figma for the code block (e.g., solid dark background vs translucent glass)?
4. **Custom Footer:** What links or layout structure does the custom footer have in Figma? Should it just contain the basic copyright and GitHub/Docs links, or something more complex?

## Verification Plan

### Manual Verification
- After implementation, I will rely on you to verify the design visually against Figma on the local server (`http://localhost:4321/prism`).
- I will ensure responsiveness by simulating mobile and tablet viewports and ensuring the marquee, code blocks, and layouts stack appropriately without overflowing.
