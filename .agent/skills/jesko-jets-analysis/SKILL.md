---
description: A deep dive into the engineering and design of the Jesko Jets website (jeskojets.com). Covers tech stack, layout patterns, animation strategies, and replication guide.
---

# Case Study: Jesko Jets (jeskojets.com)

**Overview**
The Jesko Jets website is a prime example of "Immersive Corporate" design—blending high-end luxury aesthetics with cutting-edge WebGL and scroll-driven animations. It was designed by **Ivan Chopei** and developed by the agency **The First The Last**.

Accolades: Awwwards Site of the Day (Jan 2026), Developer Award.

## 1. Technology Stack

Based on analysis of the site's behavior, performance, and agency pedigree, the following stack is used to achieve this "cinematic" feel:

### Core Framework

- **Framework**: **Nuxt.js (Vue.js)** or **Next.js (React)**.
  - *Why?* Seamless page transitions (SPA routing) are critical for the non-stop background music and persistent 3D canvas.
- **Rendering**: Static Site Generation (SSG) for SEO, with client-side hydration for the heavy interactive elements.

### Animation & Graphics (The "Secret Sauce")

- **GSAP (GreenSock Animation Platform)**: The backbone of the site's motion.
  - `ScrollTrigger`: Controls 90% of the site. Elements allow the user to "scrub" through the story (plane takeoff, service reveal) via scrolling.
  - `SplitText`: Used for the staggered, luxury typography reveals.
  - `Flip Plugin`: Likely used for smooth layout transitions between states.
- **WebGL / Three.js**:
  - Rendered in a strict `<canvas>` layer behind the DOM.
  - Uses **GLSL Shaders** for the clouds, atmospheric lighting, and possibly the plane model itself (unless pre-rendered video is used—see below).
- **Lenis (or Locomotive Scroll)**:
  - Essential for "luxury" scrolling. It smooths out the user's mouse wheel input, making the WebGL camera movements feel weighty and expensive, rather than jittery.

### Asset Management

- **Video Compression**: High-bitrate, transparency-enabled video (HEVC with Alpha for Safari, WebM for Chrome) is often used for the complex jet visuals if real-time 3D is too heavy for mobile.

## 2. Design Structure & "Vibe" System

The design follows the **"Liquid Luxury"** protocol (similar to iOS 26 conventions but darker).

### Visual Hierarchy

1. **The Stage (Background)**: The content is not "on" the page; it floats "above" a moving world. Aspects: Dark, atmospheric, heavily vignetted.
2. **Typography**:
    - **Headers**: Uppercase, wide tracking (letter-spacing), Sans-Serif (likely *Inter*, *Neue Montreal*, or a custom geometric sans).
    - **Body**: Minimal, high contrast (Pure White on Black).
3. **Navigation**:
    - **Floating Dock**: Usually bottom-centered or side-aligned to avoid cluttering the cinematic view.
    - **Glassmorphism**: Heavy use of backdrop-blur (50px+) on UI elements to separate them from the 3D background.

### Color Palette

- **Primary**: `Deep Void (#050505)`
- **Secondary**: `Carbon (#1A1A1A)`
- **Accent**: `Platinum (#E5E4E2)` or `Aviation Blue (Desaturated)`
- **Text**: `White (#FFFFFF)` with opacity adjustments (100% heading, 60% body).

## 3. Key UX Patterns (How it works)

### A. The "Scroll-Jacking" Narrative

The site does not scroll pixels; it scrolls *time*.

- **Mechanism**: The user's scroll depth (`0%` to `100%`) is mapped to a GSAP Timeline.
- **Effect**: As you scroll down, the plane turns, lighting shifts, and text fades in/out. The user feels like they are "directing" a movie.

### B. Preloader as Branding

- A heavy site needs a preloader. Jesko Jets uses this time to establish brand identity (animated logo, percentage counter). This masks the heavy accumulation of WebGL textures.

### C. Cursor Interaction

- Custom cursor (usually a small circle) that reacts magnetically to clickable elements. This increases the feeling of "flow."

## 4. How to Build a Jesko Jets Clone

If you want to replicate this, do not start with HTML/CSS. Start with the **Scene**.

### Phase 1: The Stage (Three.js / R3F)

1. Set up `react-three-fiber`.
2. Create a "Skybox" or Fog environment.
3. Load a `.gltf` model of a Jet.
4. Bind camera position to scroll offset.

### Phase 2: The UI Overlay (React/HTML)

1. Create a purely absolute-positioned HTML layer on top of the canvas.
2. Use CSS `pointer-events: none` on the container so the user can interact with the 3D scene (if interactive).
3. Re-enable `pointer-events: auto` on buttons/text.

### Phase 3: The Motion (GSAP)

1. Use `gsap.timeline()` synced to `ScrollTrigger`.
2. **Typography Reveal**: `y: 100%` -> `y: 0%` inside a `overflow: hidden` div (The "Masked Reveal" effect).

## 5. Directory Structure Suggestion

```text
/src
  /components
    /canvas           # The 3D World
      Scene.jsx
      JetModel.jsx
      Clouds.jsx
    /dom              # The UI Overlay
      Navigation.jsx
      HeroOverlay.jsx
      ServiceList.jsx
  /hooks
    useScroll.js      # Lenis integration
  /styles
    globals.css       # Tailwind + Custom fonts
```
