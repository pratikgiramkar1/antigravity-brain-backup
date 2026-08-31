# Rebuilt in Svelte

The application has been completely redesigned from the ground up using **Svelte and Vite**. This completely eradicates the brittle "vanilla Javascript" behaviors that were failing on iOS Safari.

## What Changed?

### 1. The "Begin Experience" Button is Untethered
Previously, the button was waiting for the YouTube API to say "I am fully loaded and ready to play." If your mobile browser's tracking-protection blocked YouTube, the button stayed permanently disabled.
- **The Fix:** The button now immediately reveals the site 100% of the time, instantly. 
- **The Music:** If YouTube successfully loads in the background, tapping the button will also try to play the music. If YouTube was blocked, the site will still open flawlessly (just without audio).

### 2. Flawless Native Scrolling
Previously, the Magic Dust particle system attached a Javascript `touchmove` event listener to your finger, which caused Safari to drop scroll momentum and lag.
- **The Fix:** The particle system now uses a `setInterval` loop to magically spawn dust continuously without *ever* tracking your finger on mobile. Your finger is now 100% dedicated to native browser scrolling.

### 3. Component Architecture
The site is now built using modular Svelte files:
- `App.svelte`: Manages the overall logic.
- `Overlay.svelte`: The entrance screen.
- `MagicDust.svelte`: The non-blocking particle canvas.
- `Gallery.svelte`: The elegant scroll-revealed gallery.

> [!TIP]
> This architecture is significantly more robust and will easily support future features or new pages if you decide to expand the gift.
