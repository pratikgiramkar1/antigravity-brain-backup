# Switch Processors Section Walkthrough

The "Switch Processors" section has been implemented to demonstrate the ease of integration with a high-fidelity video demo and a clean, responsive layout.

## Key Features

### 1. Visual Design
- **Background**: A solid `#212121` background provides a clean contrast for the product demo.
- **Heading**: A responsive 33px font (`clamp`) that breaks perfectly into two lines: "Switch processors with / few lines of code".
- **Video UI**: The video container has a refined `16px` border-radius, a subtle white border, and a premium shadow to make the demo pop.

### 2. Interaction & Performance
- **Lazy Loading**: The video utilizes a specialized `IntersectionObserver` logic. It stays in a "preload: none" state and only swaps its `src` and triggers playback when the user scrolls the section into view.
- **Autoplay & Loop**: Once visible, the video loops seamlessly to show the code transition repeatedly.

### 3. Responsiveness
- The entire section scales down gracefully. The video container maintains its aspect ratio while shrinking to fit narrower viewports, and the typography remains readable across all devices.

## Verification
- Verified on Chrome at **2560px**, **1440px**, and **847px** widths.
- Confirmed that the video starts only on intersection.
- Confirmed no visual overlap with the hero or FAQ sections.
