---
name: responsive-ui-ruleset
description: >
  Build responsive UIs with mobile-first principles, fluid layouts, and modern CSS. Use this for any web UI work, including responsive design, layout systems, breakpoints, CSS Grid/Flexbox, fluid typography, mobile-first development, viewport configuration, touch targets, image optimization, or when building interfaces that must work across devices from phones to desktops. Applies to HTML/CSS projects, component libraries, design systems, or any frontend work requiring adaptive layouts.
---

# Responsive UI Development Ruleset

## Core Philosophy

Responsive design in 2026 is not about shrinking desktop layouts—it's about building fluid systems that adapt naturally across the full spectrum of devices. Start with constraints (mobile) and progressively enhance. Prioritize performance, accessibility, and intrinsic adaptability over rigid breakpoint-driven layouts.

## Mobile-First Foundation

**Always start with the smallest screen.** Write base styles for mobile devices first, then use `min-width` media queries to add complexity for larger viewports.

Why this matters: Mobile-first forces content prioritization, produces leaner CSS, and aligns with how 60%+ of users access the web. It prevents the common mistake of desktop-first designs that feel cramped and compromised on mobile.

```css
/* Base styles: mobile (320px+) */
.container {
  width: 100%;
  padding: 1rem;
}

/* Tablet and up */
@media (min-width: 768px) {
  .container {
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
  }
}

/* Desktop */
@media (min-width: 1024px) {
  .container {
    padding: 3rem;
  }
}
```

## Breakpoint Strategy

### Standard Breakpoints

Use these as starting points, not rigid rules:

- **320px**: Small phones (base, no query needed)
- **480px**: Large phones
- **768px**: Tablets, portrait mode
- **1024px**: Tablets landscape, small laptops
- **1280px**: Desktops
- **1920px**: Large monitors (use sparingly)

### Content-Driven Breakpoints

Breakpoints should respond to where *content breaks*, not arbitrary device sizes. Resize your browser and add breakpoints where the layout becomes awkward, text wraps poorly, or elements misalign.

Example: If a three-column grid breaks at 860px (not 768px), set your breakpoint at 860px.

### How Many Breakpoints?

Most projects need 2-3 breakpoints. More breakpoints = more maintenance. Prefer fluid layouts that adapt continuously over many fixed layouts.

## Fluid Layouts

### Use Relative Units

**Forbidden:** Fixed pixel widths on containers, typography, or spacing in most contexts.

```css
/* ❌ Breaks on smaller screens */
.container { width: 960px; }

/* ✅ Adapts fluidly */
.container { 
  width: 90%; 
  max-width: 960px; 
}
```

**Prefer:**
- `%`, `vw`, `vh` for layout dimensions
- `rem`, `em` for typography and spacing
- `fr` units in CSS Grid
- `clamp()`, `min()`, `max()` for fluid sizing

### CSS Grid & Flexbox as Default

These are your primary layout tools. They adapt intrinsically without media queries.

**Intrinsic Grid Example:**
```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1.5rem;
}
```

This creates a responsive grid that automatically adjusts column count based on available space—no breakpoints needed.

**Flexbox for Rows/Columns:**
```css
.flex-container {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}

.flex-item {
  flex: 1 1 300px; /* grow, shrink, base width */
}
```

## Fluid Typography

**Never use fixed pixel font sizes.** Text must scale across devices.

### Using clamp()

The modern standard for fluid typography:

```css
html {
  font-size: clamp(1rem, 2.5vw, 1.5rem);
}

h1 {
  font-size: clamp(2rem, 5vw, 4rem);
}

p {
  font-size: clamp(1rem, 1.5vw, 1.125rem);
  line-height: 1.6;
}
```

**Pattern:** `clamp(min, preferred, max)`
- `min`: Smallest size (mobile)
- `preferred`: Viewport-based scaling (use `vw`)
- `max`: Largest size (desktop)

### Modular Typography Scales

Define a base size and scale ratio that adjusts between mobile and desktop. Common scales:
- **1.25** (Major Third) — tighter, good for mobile
- **1.333** (Perfect Fourth) — balanced
- **1.618** (Golden Ratio) — dramatic, desktop-friendly

Use tighter scales on mobile, larger on desktop to prevent extreme size variance on small screens.

## Responsive Images & Media

### Always Set max-width

```css
img, video, iframe {
  max-width: 100%;
  height: auto;
}
```

### Use srcset for Performance

Serve appropriately sized images based on viewport:

```html
<img 
  src="image-800.jpg" 
  srcset="image-400.jpg 400w, 
          image-800.jpg 800w, 
          image-1200.jpg 1200w"
  sizes="(min-width: 1024px) 800px, 
         (min-width: 768px) 600px, 
         100vw"
  alt="Description">
```

### Art Direction with picture

Serve different crops for different contexts:

```html
<picture>
  <source media="(min-width: 800px)" srcset="hero-wide.webp">
  <img src="hero-square.webp" alt="Product photo">
</picture>
```

### Modern Formats

Use WebP or AVIF. Compress aggressively. Large unoptimized images kill mobile performance.

## Container Queries (2026 Standard)

**Use container queries for component-level responsiveness.** Media queries handle page-level layout; container queries handle component adaptation.

```css
.card-container {
  container-type: inline-size;
}

@container (min-width: 400px) {
  .card {
    display: flex;
    flex-direction: row;
  }
}
```

This makes components truly modular—they adapt based on *their* space, not the viewport.

## Touch & Interaction

### Minimum Touch Target Sizes

**44px × 44px minimum** for any tappable element (buttons, links, form fields). 48px is safer.

```css
button, a, input, select {
  min-height: 44px;
  min-width: 44px;
  padding: 0.75rem 1.5rem;
}
```

### Spacing Between Interactive Elements

Minimum **8px gap** between tappable items to prevent mis-taps.

### Hover States Are Desktop-Only

Never rely on `:hover` for critical functionality. Touch devices don't hover. Use `:focus` and `:active` for universal interaction feedback.

## Viewport Configuration

**Always include this in your HTML `<head>`:**

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Without this, mobile browsers render at desktop width and scale down—breaking your responsive design.

## Forbidden Patterns

### ❌ Desktop-First Design

Starting with desktop and "shrinking down" leads to bloated mobile experiences. Always start mobile.

### ❌ Fixed Pixel Widths on Containers

```css
/* Never do this */
.container { width: 1200px; }
```

Use `max-width` with percentage or viewport units instead.

### ❌ Hiding Content with display: none on Mobile

Hiding content doesn't remove it—it still loads, slowing performance. If content isn't valuable on mobile, question whether it's valuable at all. Reorganize and prioritize instead.

### ❌ Too Many Breakpoints

Every breakpoint is a new layout to maintain. Prefer intrinsic layouts (Grid with `auto-fit`, Flexbox with `flex-wrap`) that adapt without breakpoints.

### ❌ Non-Responsive Navigation

Complex desktop navigation crammed onto mobile is unusable. Simplify: hamburger menus, bottom nav bars, or progressive disclosure patterns.

### ❌ Tiny Text on Mobile

Body text below 16px on mobile is hard to read. Use `clamp()` to ensure readability.

### ❌ Ignoring Orientation Changes

Test both portrait and landscape. Layouts that work in portrait often break in landscape.

### ❌ Loading Desktop-Sized Images on Mobile

Serving a 3MB image on a phone wastes bandwidth and kills performance. Use `srcset` or `picture`.

### ❌ Assuming Consistent Breakpoint Behavior

Devices between your breakpoints exist. Design for fluidity, not fixed sizes.

## Performance Requirements

- **Target load time:** Under 2 seconds on 3G networks
- **Lighthouse Mobile Score:** 90+ (Performance, Accessibility)
- **Cumulative Layout Shift (CLS):** < 0.1
- **Largest Contentful Paint (LCP):** < 2.5s

Responsive design isn't just visual—it's performant. Optimize images, minimize CSS, defer non-critical JS.

## Testing Protocol

### Test on Real Devices

Emulators are useful but incomplete. Test on:
- At least one iPhone (Safari)
- At least one Android phone (Chrome)
- One tablet (iPad or Android)
- Multiple desktop browsers (Chrome, Firefox, Safari, Edge)

### Test Across Widths

Don't just test at breakpoints. Resize your browser continuously from 320px to 1920px and watch for layout breaks.

### Test Orientation

Rotate devices. Landscape often reveals layout issues.

### Test with Real Content

Layouts break with long names, missing images, or unexpected text lengths. Test with realistic, variable content.

### Accessibility Testing

- Keyboard navigation works at all sizes
- Screen reader compatibility
- Color contrast meets WCAG AA (4.5:1 for text)
- Zoom to 200% without horizontal scroll

## Modern CSS Techniques

### Logical Properties

Use `margin-inline`, `padding-block`, `inset-inline-start` for better internationalization and logical spacing.

### Prefer prefers-* Media Queries

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; }
}

@media (prefers-color-scheme: dark) {
  /* Dark mode styles */
}
```

### Use CSS Custom Properties for Theming

```css
:root {
  --spacing-sm: clamp(0.5rem, 2vw, 1rem);
  --spacing-md: clamp(1rem, 3vw, 2rem);
  --spacing-lg: clamp(2rem, 5vw, 4rem);
}

.section {
  padding: var(--spacing-md);
}
```

## Decision Framework

When building a responsive component, ask:

1. **What's the mobile experience?** Start here.
2. **Can this adapt intrinsically?** Use Grid/Flexbox before media queries.
3. **Where does the layout break?** Set breakpoints based on content, not devices.
4. **Is this performant on 3G?** Optimize images, minimize assets.
5. **Does it work with touch?** Ensure adequate target sizes and spacing.
6. **Is it accessible?** Test keyboard nav, screen readers, zoom.

## Example: Responsive Card Grid

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 300px), 1fr));
  gap: clamp(1rem, 3vw, 2rem);
  padding: clamp(1rem, 4vw, 3rem);
}

.card {
  container-type: inline-size;
  background: white;
  border-radius: 8px;
  padding: 1.5rem;
}

@container (min-width: 400px) {
  .card {
    display: flex;
    gap: 1rem;
  }
  
  .card img {
    width: 40%;
  }
}

.card h3 {
  font-size: clamp(1.25rem, 2.5vw, 1.75rem);
  margin-bottom: 0.5rem;
}

.card p {
  font-size: clamp(0.875rem, 1.5vw, 1rem);
  line-height: 1.6;
}
```

This grid:
- Adapts column count automatically
- Uses fluid spacing and typography
- Leverages container queries for card layout
- Works from 320px to 4K without breakpoints

## Summary

- **Mobile-first always:** Start small, enhance up
- **Fluid by default:** Relative units, Grid, Flexbox, `clamp()`
- **Content-driven breakpoints:** 2-3 breakpoints where content actually breaks
- **Performance matters:** Optimize images, target <2s load on 3G
- **Touch-friendly:** 44px+ targets, adequate spacing
- **Test everywhere:** Real devices, all orientations, variable content
- **Avoid anti-patterns:** No fixed widths, no desktop-first, no hiding content
