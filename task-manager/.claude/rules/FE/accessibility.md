---
name: frontend-accessibility-rules
description: >
  Mandatory accessibility standards for building inclusive, WCAG 2.2 AA-compliant
  frontend interfaces. Use this for any frontend work involving HTML, CSS, JavaScript,
  React, Vue, or other UI frameworks, especially when building interactive components,
  forms, navigation, modals, or dynamic content. Covers ARIA usage, keyboard navigation,
  semantic HTML, color contrast, screen reader compatibility, and forbidden practices.
  Apply when writing frontend code, reviewing pull requests, auditing interfaces, or
  when the user mentions accessibility, a11y, WCAG, ADA compliance, screen readers,
  keyboard users, or inclusive design.
---

# Frontend Accessibility Rules

Accessibility is not optional. WCAG 2.2 Level AA is the current standard (2026) for legal compliance (ADA Title II, European Accessibility Act, UK Public Sector Bodies regulations). These rules ensure your frontend code works for everyone, including the 1+ billion people worldwide with disabilities.

## Core Principle: Semantic HTML First

Native HTML elements have built-in accessibility. ARIA should only supplement HTML when native elements are insufficient.

**Always prefer:**
- `<button>` over `<div role="button">`
- `<a href="...">` over `<span onclick="...">`
- `<nav>`, `<main>`, `<header>`, `<footer>` over generic `<div>`s
- `<label>` + `<input>` over `aria-label` alone
- `<select>` over custom dropdowns (unless absolutely necessary)

**Why:** Native elements provide keyboard support, focus management, and screen reader compatibility automatically. Custom implementations require extensive ARIA and JavaScript to replicate basic functionality.

## ARIA: Use Sparingly, Use Correctly

### The First Rule of ARIA

"No ARIA is better than bad ARIA." Pages with ARIA average 41% more accessibility errors than pages without it (WebAIM Million, 2026). ARIA misuse creates worse experiences than no ARIA at all.

### When to Use ARIA

- Dynamic content updates (live regions)
- Custom interactive widgets with no HTML equivalent (tree views, complex tabs)
- Communicating state changes (`aria-expanded`, `aria-pressed`, `aria-checked`)
- Providing accessible names when visible text is insufficient

### When NOT to Use ARIA

- On elements that already have semantic meaning (`<button>` doesn't need `role="button"`)
- To replace visible labels (violates WCAG 2.5.3 Label in Name)
- On non-interactive elements without a role
- To "fix" poorly structured HTML

### Critical ARIA Mistakes to Avoid

**1. Mismatched visible and accessible names**
```html
<!-- WRONG: Screen reader says "Submit form" but visual users see "Send" -->
<button aria-label="Submit form">Send</button>

<!-- RIGHT: Accessible name includes visible text -->
<button>Send application</button>
```

**2. ARIA on elements without roles**
```html
<!-- WRONG: aria-label ignored by most screen readers -->
<div aria-label="Product description">...</div>

<!-- RIGHT: Add role or use semantic element -->
<section aria-label="Product description">...</section>
```

**3. Broken ID references**
```html
<!-- WRONG: #email-hint doesn't exist, input has no description -->
<input aria-describedby="email-hint" />

<!-- RIGHT: ID exists and is in DOM -->
<input aria-describedby="email-hint" />
<span id="email-hint">Must be a valid email address</span>
```

**4. Redundant ARIA**
```html
<!-- WRONG: Unnecessary duplication -->
<button aria-label="Search">Search</button>

<!-- RIGHT: Button text is sufficient -->
<button>Search</button>
```

**5. Missing parent-child relationships**
```html
<!-- WRONG: role="option" requires parent role="listbox" -->
<ul role="listbox">
  <li>Option 1</li> <!-- Missing role="option" -->
</ul>

<!-- RIGHT: Complete relationship -->
<ul role="listbox">
  <li role="option">Option 1</li>
</ul>
```

**6. Custom attributes**
```html
<!-- WRONG: aria-tooltip is not a valid ARIA attribute -->
<button aria-tooltip="Save changes">Save</button>

<!-- RIGHT: Use defined ARIA attributes -->
<button aria-label="Save changes">Save</button>
```

### ARIA Attribute Reference

**Labels and descriptions:**
- `aria-label`: Provides accessible name when no visible text exists (icon buttons)
- `aria-labelledby`: References existing visible text by ID (preferred over aria-label)
- `aria-describedby`: Adds supplementary description (form hints, error messages)

**State and properties:**
- `aria-expanded`: "true"/"false" for collapsible sections, dropdowns
- `aria-pressed`: "true"/"false"/"mixed" for toggle buttons
- `aria-checked`: "true"/"false"/"mixed" for custom checkboxes
- `aria-disabled`: "true"/"false" when element is disabled but visible
- `aria-hidden`: "true" removes element from accessibility tree (use carefully)
- `aria-live`: "polite"/"assertive"/"off" for dynamic content updates
- `aria-current`: "page"/"step"/"location"/"date"/"time" for current item in navigation

**Relationships:**
- `aria-controls`: References ID of element controlled by this element
- `aria-owns`: Defines parent-child relationship when DOM structure doesn't

## Keyboard Navigation: Non-Negotiable

All interactive elements must be fully operable via keyboard. This is WCAG 2.1.1 (Level A) — the most fundamental requirement.

### Standard Navigation Patterns

- **Tab**: Move forward through focusable elements
- **Shift+Tab**: Move backward
- **Enter**: Activate links and buttons
- **Space**: Activate buttons, toggle checkboxes, scroll page
- **Arrow keys**: Navigate within components (menus, tabs, radio groups, sliders)
- **Escape**: Close modals, cancel operations
- **Home/End**: Jump to first/last item in lists or content

### Focus Management Rules

**1. Logical focus order**

Focus order must follow visual layout (left-to-right, top-to-bottom in LTR languages). Never use `tabindex` values greater than 0 — they create unpredictable, confusing navigation.

```html
<!-- WRONG: Positive tabindex breaks natural order -->
<button tabindex="3">Third</button>
<button tabindex="1">First</button>
<button tabindex="2">Second</button>

<!-- RIGHT: Natural DOM order, or tabindex="0" for custom elements -->
<button>First</button>
<button>Second</button>
<button>Third</button>
```

**2. Visible focus indicators**

Never remove focus outlines without providing an alternative. WCAG 2.4.7 (Level AA) requires visible focus indicators.

```css
/* FORBIDDEN: Removes focus indicator entirely */
*:focus {
  outline: none;
}

/* RIGHT: Replace default outline with custom indicator */
button:focus {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

/* BETTER: Use :focus-visible for keyboard-only indicators */
button:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}
```

**3. Skip links**

Provide "Skip to main content" link as the first focusable element on pages with repeated navigation.

```html
<a href="#main" class="skip-link">Skip to main content</a>
<nav>...</nav>
<main id="main">...</main>

<style>
.skip-link {
  position: absolute;
  left: -9999px;
  z-index: 999;
}
.skip-link:focus {
  left: 0;
  top: 0;
  background: #000;
  color: #fff;
  padding: 1rem;
}
</style>
```

**4. No keyboard traps**

Users must always be able to move focus away from any element using only keyboard. Exception: Modal dialogs intentionally trap focus but must allow Escape to close.

**5. Custom interactive elements**

If you build custom widgets (dropdowns, tabs, sliders), you must implement expected keyboard behaviors:

- **Custom dropdowns**: Arrow keys navigate options, Enter/Space selects, Escape closes
- **Tabs**: Arrow keys switch tabs, Tab moves to panel content
- **Modals**: Focus moves into dialog on open, Tab cycles through dialog controls only, Escape closes and returns focus to trigger
- **Sliders**: Arrow keys adjust value, Home/End jump to min/max

Refer to WAI-ARIA Authoring Practices Guide for complete patterns.

## Semantic HTML Structure

Proper HTML structure enables screen readers to navigate efficiently and understand content hierarchy.

### Landmarks

Use HTML5 sectioning elements to define page regions:

```html
<header>
  <nav aria-label="Main navigation">...</nav>
</header>
<main>
  <section aria-labelledby="products-heading">
    <h2 id="products-heading">Our Products</h2>
    ...
  </section>
</main>
<aside aria-label="Related links">...</aside>
<footer>...</footer>
```

**Multiple landmarks of same type:** Give each a unique label.

```html
<nav aria-label="Primary navigation">...</nav>
<nav aria-label="Footer links">...</nav>
<nav aria-label="Breadcrumb">...</nav>
```

### Headings

Headings create document outline for screen reader navigation. Never skip levels.

```html
<!-- WRONG: Skips from h1 to h3 -->
<h1>Page Title</h1>
<h3>Subsection</h3>

<!-- RIGHT: Logical hierarchy -->
<h1>Page Title</h1>
<h2>Section</h2>
<h3>Subsection</h3>
```

Don't use headings for styling. Use CSS to style any heading level as needed.

### Lists

Use `<ul>`, `<ol>`, `<dl>` for lists. Screen readers announce item count and position.

```html
<!-- WRONG: Fake list -->
<div>
  <div>• Item 1</div>
  <div>• Item 2</div>
</div>

<!-- RIGHT: Semantic list -->
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
</ul>
```

### Links vs. Buttons

- **Links (`<a>`)**: Navigate to another page or location (`href` required)
- **Buttons (`<button>`)**: Trigger actions, submit forms, toggle UI state

```html
<!-- WRONG: Button that navigates -->
<button onclick="location.href='/about'">About Us</button>

<!-- RIGHT: Link for navigation -->
<a href="/about">About Us</a>

<!-- WRONG: Link that triggers action -->
<a href="#" onclick="openModal()">Open Dialog</a>

<!-- RIGHT: Button for action -->
<button onclick="openModal()">Open Dialog</button>
```

### Forms

Every input must have an associated label. Placeholders are not labels.

```html
<!-- WRONG: No label, placeholder disappears on input -->
<input type="email" placeholder="Email address" />

<!-- RIGHT: Visible label -->
<label for="email">Email address</label>
<input type="email" id="email" />

<!-- ACCEPTABLE: Visually hidden label (use sparingly) -->
<label for="search" class="visually-hidden">Search</label>
<input type="search" id="search" placeholder="Search..." />
```

**Error messages:** Associate with inputs using `aria-describedby`.

```html
<label for="password">Password</label>
<input type="password" id="password" aria-describedby="password-error" aria-invalid="true" />
<span id="password-error" role="alert">Password must be at least 8 characters</span>
```

**Required fields:** Use `required` attribute and indicate visually.

```html
<label for="name">Name <span aria-label="required">*</span></label>
<input type="text" id="name" required />
```

## Color and Contrast

WCAG 2.2 Level AA requires:
- **Normal text**: 4.5:1 contrast ratio
- **Large text** (18pt+ or 14pt+ bold): 3:1 contrast ratio
- **UI components and graphics**: 3:1 contrast ratio

**Never rely on color alone** to convey information (WCAG 1.4.1).

```html
<!-- WRONG: Only color indicates error -->
<span style="color: red;">Invalid email</span>

<!-- RIGHT: Icon + text + color -->
<span style="color: red;">
  <svg aria-hidden="true">...</svg>
  Error: Invalid email
</span>
```

Use tools like WebAIM Contrast Checker, Stark, or browser DevTools to verify contrast.

## Screen Reader Considerations

### Alternative Text

All meaningful images must have `alt` text. Decorative images should have empty `alt=""`.

```html
<!-- Informative image -->
<img src="chart.png" alt="Sales increased 40% in Q1 2026" />

<!-- Decorative image -->
<img src="divider.png" alt="" />

<!-- Functional image (button/link) -->
<a href="/home">
  <img src="logo.png" alt="Acme Corp Home" />
</a>
```

**Complex images:** Provide long description via `aria-describedby` or adjacent text.

### Hidden Content

Understand the difference:

- `display: none` / `visibility: hidden`: Hidden from everyone, including screen readers
- `aria-hidden="true"`: Hidden from screen readers only, still visible
- `.visually-hidden` class: Hidden visually, still announced by screen readers

```css
/* Visually hidden but available to screen readers */
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

### Live Regions

Announce dynamic content changes without moving focus.

```html
<!-- Polite: Waits for screen reader to finish current announcement -->
<div aria-live="polite" aria-atomic="true">
  Item added to cart
</div>

<!-- Assertive: Interrupts immediately (use sparingly) -->
<div aria-live="assertive" role="alert">
  Error: Payment failed
</div>
```

## Forbidden Practices

These patterns always fail accessibility:

1. **Removing focus outlines** without replacement (`outline: none` without alternative)
2. **Positive tabindex values** (`tabindex="1"`, `tabindex="2"`, etc.)
3. **Click-only interactions** on non-interactive elements (`<div onclick="...">` without keyboard support)
4. **Hover-only content** (tooltips, menus) with no keyboard/focus alternative
5. **Auto-playing media** without pause control
6. **Time limits** without ability to extend/disable
7. **Keyboard traps** (except intentional focus traps in modals with Escape to close)
8. **Empty links or buttons** (`<a href="#">` with no text, `<button></button>`)
9. **Using placeholders as labels** without visible label text
10. **Color-only information** ("Click the red button" without other indicator)
11. **Inaccessible CAPTCHAs** without alternative (use reCAPTCHA v3 or similar)
12. **Overlay widgets** that claim to fix accessibility automatically (often make things worse)

## Testing Checklist

Before considering any interface complete:

1. **Keyboard test**: Unplug mouse, navigate entire interface with Tab, Enter, Space, Arrow keys
2. **Screen reader test**: Test with NVDA (Windows), JAWS (Windows), or VoiceOver (macOS/iOS)
3. **Contrast test**: Verify all text and UI components meet 4.5:1 or 3:1 ratios
4. **Zoom test**: Zoom to 200%, verify content remains usable without horizontal scroll
5. **Automated scan**: Run axe DevTools, WAVE, or Lighthouse accessibility audit
6. **HTML validation**: Validate markup at validator.w3.org

## Framework-Specific Notes

### React

- Use `htmlFor` instead of `for` on labels
- Manage focus with `useRef` and `.focus()` after dynamic changes
- Use fragments to avoid unnecessary wrapper divs
- Libraries: `@reach/ui`, `@radix-ui`, `@headlessui` provide accessible primitives

### Vue

- Use `ref` to manage focus
- `v-if` removes from DOM (good), `v-show` uses `display: none` (still in DOM)
- Vue 3 Teleport useful for accessible modals

### Angular

- Use `@angular/cdk/a11y` for focus management and keyboard utilities
- `cdkTrapFocus` for modal focus traps
- LiveAnnouncer service for screen reader announcements

## Resources

- **WCAG 2.2**: w3.org/WAI/WCAG22/quickref/
- **ARIA Authoring Practices**: w3.org/WAI/ARIA/apg/
- **WebAIM**: webaim.org (articles, contrast checker, screen reader survey)
- **Deque University**: dequeuniversity.com (comprehensive guides)
- **Testing tools**: axe DevTools, WAVE, Lighthouse, NVDA, JAWS, VoiceOver

## Remember

Accessibility is not a feature you add at the end. It's a fundamental aspect of quality frontend engineering. When you build accessible interfaces, you create better experiences for everyone — people with disabilities, people using keyboards, people on mobile devices, people with slow connections, and people using assistive technologies you've never heard of.

Start with semantic HTML, add ARIA only when necessary, ensure keyboard operability, test with real assistive technologies, and never ship an interface you can't navigate without a mouse.