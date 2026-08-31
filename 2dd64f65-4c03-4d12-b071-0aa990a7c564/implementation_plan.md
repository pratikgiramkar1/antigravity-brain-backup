# Rebuild Rakhi Experience in Svelte

The vanilla Javascript implementation has suffered from mobile browser quirks (specifically iOS Safari's aggressive autoplay blocking and touch-event scroll hijacking). To provide a flawless, modern, and robust experience, we will rebuild the entire application from scratch using **Svelte**.

## User Review Required

> [!IMPORTANT]  
> Please review this plan. Once approved, I will wipe the slate clean and generate the new Svelte application. The new app will be created in a folder named `svelte-rakhi`. You will need to link your Vercel project to this new folder for future deployments.

## Open Questions

> [!WARNING]  
> The "Begin Experience" button was blocked because your mobile browser's ad-blocker or tracking-protection blocked the YouTube API from loading, leaving the button disabled forever. 
> 
> **Question for you:** For the Svelte rewrite, I will completely untether the "Begin Experience" button from the music player so it is **never** blocked again. If the YouTube music fails to load in the background, you will still be able to enter the site smoothly. Are you okay with this fallback behavior?

## Proposed Changes

### 1. Project Initialization
- Use Vite to scaffold a new Svelte Single Page Application (`npm create vite@latest svelte-rakhi -- --template svelte`).
- Install necessary dependencies.

### 2. Asset Migration
- Copy all existing images from `assets/images/` to the new Svelte `public/images/` directory so they are served correctly.

### 3. Component Architecture (Svelte)

We will break the vanilla HTML into modular Svelte components for flawless state management:

#### [NEW] `src/App.svelte`
The main orchestrator. It will hold the `hasStarted` state. If `false`, it shows the Overlay. If `true`, it mounts the main content.

#### [NEW] `src/components/Overlay.svelte`
The launch screen. 
- The "Begin Experience" button will **always** be clickable instantly. No more waiting or blocking.

#### [NEW] `src/components/MagicDust.svelte`
A dedicated Svelte component for the canvas particles.
- It will exclusively use a passive `setInterval` for ambient background dust and a passive `mousemove` for desktop. 
- **Zero** touch event hijacking to guarantee buttery smooth native mobile scrolling.

#### [NEW] `src/components/MusicPlayer.svelte`
A headless component managing the YouTube IFrame API.
- Exposes a reactive `$isPlaying` store.
- If it fails to load, it will fail silently in the background without breaking the user interface.
- Contains the 🎵 SVG button that reactively adds the slash when paused.

#### [NEW] `src/components/Hero.svelte` & `src/components/Gallery.svelte`
The main visual content.
- Translating all vanilla CSS into scoped Svelte `<style>` blocks.
- Using `IntersectionObserver` via Svelte actions (`use:reveal`) for the fade-up animations.

### 4. Styling (CSS)
- Migrate `style.css` global variables and fonts to `src/app.css`.
- Ensure `overflow-x: hidden` is applied securely without breaking iOS momentum scrolling.

## Verification Plan

### Manual Verification
1. I will run the local Vite dev server (`npm run dev`).
2. I will verify that the "Begin Experience" button works instantly.
3. I will confirm the Music SVG toggle updates correctly using Svelte reactivity.
4. I will deploy a test build to ensure no Vercel-specific quirks remain.
