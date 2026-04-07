---
name: ui-code-consistency
description: >
  Enforce UI and code consistency across design systems and codebases. Use this for design systems, component libraries, UI development, frontend architecture, style guides, pattern libraries, or whenever you need to maintain consistency in visual design, design tokens, spacing, typography, component naming, code style, or reusable patterns. Applies to React, Vue, Angular, web components, CSS, SASS, design tokens, Figma, Sketch, and any UI development workflow.
---

# UI and Code Consistency

This skill helps you build and maintain consistent UI and code across design systems, component libraries, and applications. Consistency reduces cognitive load, accelerates development, strengthens brand identity, and creates predictable user experiences.

## Core Principles

**Single source of truth**
Design tokens, components, and patterns should be defined once and referenced everywhere. Duplication creates divergence.

**Semantic over literal naming**
Name things by purpose, not appearance. `color-background-primary` survives a rebrand; `blue-500` does not. Semantic names make systems resilient to visual changes.

**Layered abstraction**
Separate raw values (primitives) from meaning (semantic tokens) from usage (component tokens). This enables theming, rebranding, and platform-specific variations without rewriting components.

**Consistency serves users, not aesthetics**
Consistency creates familiarity and reduces learning curves. Users transfer knowledge from one part of your product to another. Break consistency only when it genuinely adds value.

---

## Design Tokens

Design tokens are named variables storing design decisions—colors, spacing, typography, shadows, motion. They bridge design and code, ensuring both stay aligned.

### Three-Layer Token Architecture

**Primitive tokens** — Raw values, the base palette
```
blue-500: #0F62FE
gray-100: #F4F4F4
spacing-4: 16px
font-size-lg: 20px
radius-md: 8px
```

**Semantic tokens** — Assign meaning to primitives
```
color-background-primary → blue-500
color-text-muted → gray-100
spacing-layout-md → spacing-4
```

**Component tokens** (optional) — Component-specific overrides
```
button-primary-background → color-background-primary
button-primary-hover-background → blue-600
```

Components reference semantic or component tokens, never primitives directly. This allows global visual changes by updating token definitions, not components.

### Token Naming Conventions

Structured, multi-part names improve discoverability:

- **Two-part**: `spacing-md`, `radius-sm`
- **Three-part**: `color-background-primary`, `color-text-secondary`
- **Four-part**: `color-button-primary-hover`, `spacing-card-padding-sm`

Format: `[category]-[role]-[variant]-[state]`

Avoid raw value names like `blue-500` or `red-600` in semantic tokens. Use meaning-based names that describe intent.

### When to Avoid Over-Tokenization

Not every value needs a token. Component-specific tokens can bloat the system if overused. Create tokens for:
- Values used in multiple components
- Values that might change (theming, branding)
- Values tied to design decisions (spacing scale, color roles)

Skip tokens for one-off component-specific values unlikely to change or be reused.

---

## Spacing System

Consistent spacing creates visual rhythm and hierarchy. Define a spacing scale and apply it everywhere.

**Common scale**: 4px base unit (0.25rem)
```
spacing-xs: 4px
spacing-sm: 8px
spacing-md: 16px
spacing-lg: 24px
spacing-xl: 32px
spacing-2xl: 48px
```

**Usage**:
- Use the same token for related spacing (e.g., `spacing-md` for consistent gaps between form fields)
- Larger spacing separates unrelated sections; smaller spacing groups related items
- Avoid arbitrary values—stick to the scale

---

## Typography Scale

Define a limited set of text styles covering all use cases. Composite typography tokens bundle font family, size, weight, line height, and letter spacing.

```
text-heading-xl: { family: Inter, size: 32px, weight: 700, line-height: 1.2 }
text-heading-lg: { family: Inter, size: 24px, weight: 600, line-height: 1.3 }
text-body: { family: Inter, size: 16px, weight: 400, line-height: 1.5 }
text-body-sm: { family: Inter, size: 14px, weight: 400, line-height: 1.5 }
text-label: { family: Inter, size: 12px, weight: 500, line-height: 1.4 }
```

Limit the number of text styles—5 to 8 is typical. More than that creates inconsistency and decision fatigue.

---

## Component Naming

### File and Folder Naming

Use consistent casing across the codebase:
- **kebab-case** for CSS classes, file names: `button-primary.scss`, `card-header.jsx`
- **PascalCase** for React/Vue components: `ButtonPrimary.jsx`, `CardHeader.vue`
- **camelCase** for JavaScript variables and functions: `handleClick`, `isDisabled`

Match file names to component names:
```
ButtonPrimary.jsx → exports ButtonPrimary
button-primary.scss → contains .button-primary
```

### Component Naming Structure

**Descriptive, not cryptic**: `NavigationMenu` over `NavMenu` (unless abbreviation is universally clear)

**Hierarchical naming** for related components:
```
Card
CardHeader
CardBody
CardFooter
```

Or with slash-separated namespaces in design tools:
```
Card/Header
Card/Body
Card/Footer
```

**State-based variants**:
```
Button/Primary
Button/Secondary
Button/Disabled
```

**BEM for CSS** (if not using CSS-in-JS or scoped styles):
```css
.card { }
.card__header { }
.card__body { }
.card--featured { }
```

Double underscore (`__`) for elements, double hyphen (`--`) for modifiers.

### Prefixing to Avoid Conflicts

If integrating with third-party libraries or migrating from legacy systems, prefix component names:
```
ds-button (design system button)
legacy-button (old button)
```

This prevents naming collisions and clarifies component origin.

---

## Reusable Components

### Component API Consistency

Component props/attributes should follow consistent patterns:

**Boolean props**: Use `is-`, `has-`, `should-` prefixes
```jsx
<Button isDisabled />
<Card hasHeader />
<Modal shouldCloseOnEscape />
```

**Event handlers**: Use `on-` prefix
```jsx
<Button onClick={handleClick} />
<Input onChange={handleChange} />
```

**Size/variant props**: Use consistent names
```jsx
<Button size="large" variant="primary" />
<Input size="medium" variant="outlined" />
```

### Component States

Every interactive component should define states:
- **Default**: Resting state
- **Hover**: Mouse over
- **Active/Pressed**: Click or tap
- **Focus**: Keyboard focus (critical for accessibility)
- **Disabled**: Non-interactive
- **Loading**: Async operation in progress
- **Error**: Validation or system error

Document and design all states. Inconsistent state handling confuses users.

### Composition Over Duplication

Build complex components from simpler ones. If you find yourself creating `ButtonLarge`, `ButtonSmall`, `ButtonPrimary`, `ButtonSecondary`—you're duplicating. Instead:

```jsx
<Button size="large" variant="primary" />
<Button size="small" variant="secondary" />
```

One component, multiple configurations.

---

## Coding Style Consistency

### Linting and Formatting

Use automated tools to enforce style:
- **ESLint** for JavaScript/TypeScript
- **Stylelint** for CSS/SCSS
- **Prettier** for formatting

Commit config files (`.eslintrc`, `.prettierrc`, `stylelint.config.js`) to the repo so all contributors use the same rules.

### CSS/SCSS Conventions

**Organize rules consistently**:
1. Positioning (`position`, `top`, `left`, `z-index`)
2. Box model (`display`, `width`, `height`, `margin`, `padding`, `border`)
3. Typography (`font-family`, `font-size`, `line-height`, `color`)
4. Visual (`background`, `box-shadow`, `opacity`)
5. Misc (`cursor`, `transition`, `animation`)

**Use design tokens**:
```scss
.button-primary {
  background: var(--color-background-primary);
  padding: var(--spacing-md) var(--spacing-lg);
  border-radius: var(--radius-md);
  font-size: var(--text-body-size);
}
```

**Avoid magic numbers**:
```scss
/* Bad */
.card { padding: 23px; }

/* Good */
.card { padding: var(--spacing-lg); }
```

### JavaScript/TypeScript Conventions

**Consistent prop destructuring**:
```jsx
// Consistent
const Button = ({ variant, size, children, onClick }) => { ... }

// Inconsistent
const Button = (props) => {
  const variant = props.variant;
  ...
}
```

**Consistent import order**:
1. External libraries (`react`, `lodash`)
2. Internal utilities (`@/utils`)
3. Components (`@/components`)
4. Styles (`./Button.scss`)

**Consistent file structure**:
```
components/
  Button/
    Button.jsx
    Button.module.scss
    Button.test.jsx
    index.js  // re-exports Button
```

---

## Documentation and Governance

### Document the "Why"

Don't just list rules—explain reasoning:
- Why semantic token names? (Resilience to visual changes)
- Why limit text styles? (Reduces decision fatigue, maintains hierarchy)
- Why spacing scale? (Creates visual rhythm, speeds development)

When teams understand the reasoning, they apply principles intelligently rather than following rules rigidly.

### Living Style Guide

Maintain a living style guide or component library documentation (Storybook, Styleguidist, Zeroheight, etc.) showing:
- All components with all states
- Token reference (colors, spacing, typography)
- Usage guidelines (when to use which component)
- Code examples

Keep it up to date—an outdated style guide is worse than none.

### Consistency Audits

Periodically audit the codebase:
- Are new components following naming conventions?
- Are design tokens being used, or are magic values creeping in?
- Are deprecated patterns still in use?

Automated linting catches some issues; manual audits catch the rest.

### Governance Without Rigidity

Establish a design system team or working group to:
- Review and approve new components
- Maintain token libraries
- Update documentation
- Resolve inconsistencies

But allow flexibility: if a use case genuinely doesn't fit the system, adapt the system rather than forcing a bad fit. Consistency serves users and teams, not the other way around.

---

## When to Break Consistency

Consistency is a tool, not a law. Break it when:
- A unique context requires a unique solution (e.g., a marketing landing page vs. a product dashboard)
- Consistency would harm usability (e.g., using a destructive red button in a confirmation dialog, even if primary buttons are blue elsewhere)
- Platform conventions override internal conventions (e.g., iOS vs. Android UI patterns)

When you break consistency, document why. This prevents the exception from becoming an unintentional pattern.

---

## Quick Checklist

- [ ] Design tokens defined in three layers (primitive, semantic, component)
- [ ] Token names describe purpose, not appearance
- [ ] Spacing scale defined and applied consistently
- [ ] Typography scale limited to 5-8 styles
- [ ] Component naming follows consistent casing (kebab-case for files, PascalCase for components)
- [ ] Component props/attributes follow consistent patterns
- [ ] All component states designed and documented
- [ ] Linting and formatting tools configured and committed
- [ ] CSS organized consistently (positioning, box model, typography, visual)
- [ ] Living style guide maintained and up to date
- [ ] Periodic consistency audits scheduled
- [ ] Design system governance established with clear decision-making process