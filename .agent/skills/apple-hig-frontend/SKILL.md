---
name: apple-hig-frontend
description: Expert guide to implementing Apple's Human Interface Guidelines (HIG) in Frontend (React/Tailwind & SwiftUI). Covers philosophy, visual design, layout, materials, and interaction patterns.
---

# Apple Human Interface Guidelines (Frontend Implementation)

This skill translates Apple's design philosophy into actionable code for Web (React, Tailwind CSS) and Native (SwiftUI) development.

## 1. Core Philosophy (The "Why")

* **Clarity**: Content is paramount. Text is legible at every size, icons are precise and lucid, decorations are subtle and appropriate.
* **Deference**: The interface recedes. Fluid motion and a crisp, beautiful interface help people understand and interact with content while never competing with it.
* **Depth**: Distinct visual layers and meaningful motion convey hierarchy and vitality.

## 2. Visual Design System

### A. Layout & Geometry

* **Fluidity**: Interfaces should adapt to any screen size (Responsive Design).
* **Safe Areas**: Respect the notch, home indicator, and rounded corners.
  * *Web*: Use `env(safe-area-inset-top)`, `env(safe-area-inset-bottom)`, etc.
  * *SwiftUI*: `safeAreaInset()`
* **Touch Targets**: Minimum **44x44 pt/px** for interactive elements.
* **Margins**:
  * Standard View: 16pt (Compact/Mobile), 20pt (Regular/Tablet).

### B. Typography (SF Pro)

Apple uses the System Font (SF Pro).

* **Hierarchy**: Use weight and size to denote hierarchy, not just color.
* **Leading**: Default line-height is usually roughly 1.2-1.4x the font size.
* **Tracking**: Apple adjusts letter-spacing (tracking) dynamically. Larger headings have tighter tracking; smaller captions have wider tracking.

**Web Implementation (Tailwind):**

```css
/* Add to your CSS/Tailwind config */
font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
```

### C. Color & Dark Mode

Do not hardcode hex values. Use **Semantic Colors**.

| Semantic Role | Light Mode Value (Approx) | Dark Mode Value (Approx) | Usage |
| :--- | :--- | :--- | :--- |
| **System Background** | `#FFFFFF` | `#000000` | Main view background |
| **Secondary Background** | `#F2F2F7` | `#1C1C1E` | Grouped table views, cards |
| **Label (Primary)** | `#000000` | `#FFFFFF` | Primary text |
| **Secondary Label** | `#3C3C43` (60%) | `#EBEBF5` (60%) | Subtitles, description |
| **Tertiary Label** | `#3C3C43` (30%) | `#EBEBF5` (30%) | Placeholders, disabled text |
| **Separator** | `#3C3C43` (29%) | `#545458` (65%) | Thin borders between items |
| **System Blue** | `#007AFF` | `#0A84FF` | Actionable items, links |

### D. Materials (Glassmorphism / Vibrancy)

"Materials" are translucent layers that blur the content behind them (Backdrop Blur). This creates a sense of depth and context.

**Material Types**:

1. **Ultra Thin**: Very translucent (HUDs, transient views).
2. **Thin**: Spotlight search, lighter interactions.
3. **Regular**: Default navigation bars, toolbars.
4. **Thick**: Popovers, modal sheets.

**Web Implementation (Tailwind):**
To achieve the "Apple Glass" look:

1. **Blur**: High blur value.
2. **Saturation**: Boost saturation to maintain color vibrancy of background.
3. **Color**: Semi-transparent white (Light) or black (Dark).

```html
<!-- Light Mode Glass -->
<div class="bg-white/70 backdrop-blur-xl backdrop-saturate-150 border-b border-white/20">
  <!-- Content -->
</div>

<!-- Dark Mode Glass -->
<div class="bg-black/70 backdrop-blur-xl backdrop-saturate-150 border-b border-white/10">
  <!-- Content -->
</div>
```

**SwiftUI Implementation:**

```swift
.background(.ultraThinMaterial)
```

## 3. UI Patterns & Components

### A. Navigation Bar

* **Large Title**: Collapses to inline title on scroll.
* **Translucency**: Background blurs content scrolling underneath.
* **Actions**: "Back" on left (with label), primary action on right.

### B. Cards & Grouped Views

* **Corner Radius**: Matches the curvature of the device or "Continuous Curve" (Squircle).
  * *Web*: Use `rounded-2xl` or `rounded-3xl` (approx 20px-32px).
* **Shadows**: extremely diffuse, soft ambient shadows.
  * *Web*: `box-shadow: 0 4px 24px rgba(0,0,0,0.06);`

### C. Modals & Sheets

* **Page Sheet**: Card that slides up, covering mostly everything but top margin.
* **Interaction**: Drag down to dismiss.

## 4. Interaction & Motion

### A. Physics-Based Animation

Apple animations feel natural. They use spring physics (mass, stiffness, damping) rather than rigid easing curves (linear, ease-in-out).

**Web (Framer Motion / Tailwind):**
Avoid `transition-all duration-300 ease` for complex movement. Use springs.

```javascript
// Framer Motion ideal spring
transition={{ type: "spring", stiffness: 300, damping: 30 }}
```

### B. Gestures

* **Direct Manipulation**: Swipe pages, pinch photos.
* **Feedback**: Active states should be instant.
  * *Buttons*: Dim opacity on press (don't shrink/scale unless necessary). `active:opacity-50`.

## 5. Implementation Checklist

1. [ ] **Grid & Layout**: Are margins consistent (16px/20px)? Are safe areas respected?
2. [ ] **Typography**: Is SF Pro (or equivalent) used? Is hierarchy established via weight/size?
3. [ ] **Color**: does it support Dark Mode automatically? Are you using semantic colors?
4. [ ] **Touch**: Are all clickable things at least 44x44px?
5. [ ] **Haptics** (Native): Are you generating feedback for significant actions?
6. [ ] **Accessibilty**: Does Dynamic Type scale layouts? Is contrast sufficient?

## 6. Code Snippets

### React / Tailwind (iOS-style List Item)

```jsx
<div className="bg-white dark:bg-[#1C1C1E] rounded-xl overflow-hidden">
  <div className="flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-800 last:border-0 active:bg-gray-50 dark:active:bg-gray-800 transition-colors cursor-pointer">
    <div className="flex items-center gap-3">
      <div className="w-8 h-8 rounded-lg bg-blue-500 flex items-center justify-center text-white">
        <Icon name="star" />
      </div>
      <span className="text-[17px] font-medium text-black dark:text-white">Favorites</span>
    </div>
    <ChevronRight className="text-gray-400 w-4 h-4" />
  </div>
</div>
```

### CSS Variables (Semantic Colors)

```css
:root {
  --system-background: #FFFFFF;
  --system-secondary-background: #F2F2F7;
  --system-blue: #007AFF;
}

@media (prefers-color-scheme: dark) {
  :root {
    --system-background: #000000;
    --system-secondary-background: #1C1C1E;
    --system-blue: #0A84FF;
  }
}
```
