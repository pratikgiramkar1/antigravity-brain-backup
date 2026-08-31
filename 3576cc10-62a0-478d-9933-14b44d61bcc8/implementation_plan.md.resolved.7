# Replace Prism Image with Scattering SVG

## Goal
Replace the old `prism-crystal.png` with the new, full-width `lightscattering.svg` which includes the white beam entering and the scattered rainbow light exiting the prism. The scattered light needs to properly flow behind the 3 stat cards on desktop and be smartly cropped via `transform` on screens 960px and smaller to avoid visual clutter when the cards wrap to a single column.

## Proposed Changes

### [src/pages/prism.astro](file:///Users/pratik.giramkar/orca-docs/src/pages/prism.astro)

#### [MODIFY] `src/pages/prism.astro`
- **Asset Import**: Import `lightscattering.svg` and remove unused `prism-crystal.png` import.
- **HTML structure**: Change the `<img src={prismCrystalSrc} ...>` to use the new scattered light SVG.
- **Desktop CSS Adjustments (>.stats-crystal)**:
  - Remove `max-width: 380px` constraint.
  - Make the image much larger (e.g. `width: 200%` or specific large dimensions) and use negative margins or translations so the prism part aligns under the title but the beam extends all the way to the right behind the cards.
  - Adjust `.stats-right` (the cards) to have a higher `z-index` (e.g., `position: relative; z-index: 10`) so they sit on top of the colorful scattered light beam.
- **Mobile CSS Adjustments (<960px)**:
  - Add a media query block for `.stats-crystal`.
  - Apply `transform: scale(2) translateX(...)` (or similar properties) to "chop off" the empty scattered light area on the right, keeping the main prism and the beginning of the light burst visible without making the section excessively tall or creating horizontal scroll.
  - Ensure `.stats-band` has `overflow: hidden` to avoid horizontal scrolling from the large scaled SVG.

## Open Questions

- Does the new SVG have the white beam entering from the left edge? Depending on the internal padding of the SVG, we might need to fine-tune `left` or `transform: translate()` values.

## Verification Plan
1. Check the desktop view to ensure the prism is under the text and the scattered light passes correctly behind the cards.
2. Check the view at <960px to confirm the `transform` scales the prism up beautifully and chops the right side off cleanly, preventing layout congestion.
