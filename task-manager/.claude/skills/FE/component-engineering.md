---
name: component-engineering
description: >
  Build production-ready UI components with reusable architecture, robust prop design,
  event handling, accessibility integration, empty/loading/error states, and TypeScript typing.
  Use this whenever building React components, component libraries, design systems, reusable UI elements,
  or when the user mentions components, props, events, accessibility (a11y), TypeScript interfaces,
  state patterns, component APIs, or asks about component best practices, architecture, or design patterns.
---

# Component Engineering

## Overview

This skill guides the creation of production-grade UI components that are reusable, accessible, type-safe, and maintainable. The focus is on designing component APIs that other developers will find intuitive, handling real-world states gracefully, and building accessibility into the foundation rather than bolting it on later.

## Core Principles

### Design for Composition Over Configuration

Components become more flexible when they compose rather than configure. Instead of adding props for every variation, design small components that nest and combine:

- Prefer `children` and slot patterns over boolean flags
- A `<Card><CardHeader /><CardBody /></Card>` beats `<Card hasHeader headerAlign="left">`
- Composition scales infinitely; prop explosions don't

### Make the Right Thing Easy

The component API should guide developers toward correct usage:

- Required accessibility props should be type-enforced, not optional
- Sensible defaults reduce cognitive load
- Common patterns should require less code than anti-patterns

### Fail Loudly in Development, Gracefully in Production

- TypeScript catches API misuse at compile time
- PropTypes or runtime validation catches edge cases in dev
- Production handles missing data without crashing

## Component API Design

### Prop Design Patterns

**Discriminated unions for mutually exclusive states:**

```typescript
type ButtonProps = 
  | { variant: 'link'; href: string; onClick?: never }
  | { variant: 'button'; onClick: () => void; href?: never };
```

This prevents impossible states like a link with an onClick but no href.

**Use `never` to enforce prop exclusivity:**

```typescript
type LabelProps = 
  | { ariaLabel: string; ariaLabelledBy?: never }
  | { ariaLabelledBy: string; ariaLabel?: never };
```

One or the other must be provided, but not both.

**Prefer specific types over `any` or `string`:**

```typescript
type Size = 'sm' | 'md' | 'lg';
type Variant = 'primary' | 'secondary' | 'danger';
```

This enables autocomplete and catches typos.

**Extract common patterns into utility types:**

```typescript
type WithChildren<T = {}> = T & { children: ReactNode };
type WithClassName<T = {}> = T & { className?: string };
type Polymorphic<T extends ElementType = 'div'> = {
  as?: T;
} & ComponentPropsWithoutRef<T>;
```

### Event Handler Naming

Be consistent and predictable:

- **Props (what parent passes in):** `onChange`, `onSelect`, `onSubmit`
- **Internal handlers:** `handleChange`, `handleSelect`, `handleSubmit`
- **Emit custom events with context:** `onChange(value, metadata)` not just `onChange(event)`

Avoid generic names like `onAction` or `callback`. Be specific.

### Prop Spreading and Rest Props

Allow consumers to pass through native HTML attributes:

```typescript
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary';
  size?: 'sm' | 'md' | 'lg';
}

const Button = ({ variant = 'primary', size = 'md', children, ...rest }: ButtonProps) => (
  <button className={cn(styles[variant], styles[size])} {...rest}>
    {children}
  </button>
);
```

This allows `disabled`, `aria-*`, `data-*`, and other standard props without explicitly defining each.

## State Patterns: Empty, Loading, Error

Real components handle more than the happy path. Design for these states from the start:

### Pattern 1: State Discriminated Union

```typescript
type DataState<T> = 
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function DataDisplay<T>({ state, render }: { state: DataState<T>; render: (data: T) => ReactNode }) {
  switch (state.status) {
    case 'idle': return <Empty message="No data loaded yet" />;
    case 'loading': return <Spinner />;
    case 'error': return <ErrorBanner error={state.error} />;
    case 'success': return <>{render(state.data)}</>;
  }
}
```

This pattern makes impossible states unrepresentable: you can't have both `data` and `error` simultaneously.

### Pattern 2: Render Props for Flexibility

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
  renderEmpty?: () => ReactNode;
  renderLoading?: () => ReactNode;
  isLoading?: boolean;
}
```

Let consumers customize each state while providing sensible defaults.

### Pattern 3: Compound Components for Complex States

```typescript
<Table>
  <Table.Loading />
  <Table.Error retry={handleRetry} />
  <Table.Empty action={<Button>Add Item</Button>} />
  <Table.Data>{/* rows */}</Table.Data>
</Table>
```

The parent component orchestrates which child renders based on state.

## Accessibility Integration

### Enforce Accessibility at the Type Level

Make inaccessible components impossible to create:

```typescript
type AccessibleButton = 
  | { children: ReactNode; 'aria-label'?: never }
  | { 'aria-label': string; children?: never };
```

Either visible text or an aria-label must be provided.

### Common Patterns

**Focus management:**

```typescript
const Modal = ({ isOpen, onClose, children }: ModalProps) => {
  const previousFocus = useRef<HTMLElement | null>(null);
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      previousFocus.current = document.activeElement as HTMLElement;
      modalRef.current?.focus();
    } else {
      previousFocus.current?.focus();
    }
  }, [isOpen]);

  return (
    <div ref={modalRef} role="dialog" aria-modal="true" tabIndex={-1}>
      {children}
    </div>
  );
};
```

**Keyboard navigation:**

```typescript
const Tabs = ({ tabs, activeTab, onChange }: TabsProps) => {
  const handleKeyDown = (e: KeyboardEvent, index: number) => {
    if (e.key === 'ArrowRight') onChange(tabs[Math.min(index + 1, tabs.length - 1)]);
    if (e.key === 'ArrowLeft') onChange(tabs[Math.max(index - 1, 0)]);
    if (e.key === 'Home') onChange(tabs[0]);
    if (e.key === 'End') onChange(tabs[tabs.length - 1]);
  };

  return (
    <div role="tablist">
      {tabs.map((tab, i) => (
        <button
          key={tab.id}
          role="tab"
          aria-selected={tab.id === activeTab}
          tabIndex={tab.id === activeTab ? 0 : -1}
          onKeyDown={(e) => handleKeyDown(e, i)}
        >
          {tab.label}
        </button>
      ))}
    </div>
  );
};
```

**ARIA live regions for dynamic content:**

```typescript
const Toast = ({ message, type }: ToastProps) => (
  <div role="status" aria-live="polite" aria-atomic="true">
    {message}
  </div>
);
```

### Accessibility Checklist

- Semantic HTML elements (`<button>`, `<nav>`, `<main>`) over `<div>` with roles
- Keyboard navigable: all interactive elements reachable via Tab, operable via Enter/Space
- Focus visible: never `outline: none` without a custom focus indicator
- Color contrast: WCAG AA minimum (4.5:1 for text, 3:1 for UI components)
- Screen reader tested: use NVDA, JAWS, or VoiceOver to verify

## TypeScript Patterns

### Generic Components

```typescript
interface SelectProps<T> {
  options: T[];
  value: T;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string;
}

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select value={getValue(value)} onChange={(e) => {
      const selected = options.find(o => getValue(o) === e.target.value);
      if (selected) onChange(selected);
    }}>
      {options.map(option => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}
```

### Polymorphic Components

Allow consumers to change the underlying element:

```typescript
type PolymorphicProps<E extends ElementType> = {
  as?: E;
  children: ReactNode;
} & ComponentPropsWithoutRef<E>;

function Text<E extends ElementType = 'span'>({ as, children, ...props }: PolymorphicProps<E>) {
  const Component = as || 'span';
  return <Component {...props}>{children}</Component>;
}

// Usage:
<Text>Default span</Text>
<Text as="h1">Heading</Text>
<Text as="a" href="/">Link</Text>
```

### Strict Event Handlers

```typescript
type ChangeHandler<T> = (value: T, event: ChangeEvent<HTMLInputElement>) => void;

interface InputProps {
  value: string;
  onChange: ChangeHandler<string>;
}
```

Provide both the processed value and the raw event for flexibility.

## Testing Strategy

### What to Test

- **Rendering:** Does it render without crashing? Does it display the right content?
- **User interactions:** Click, type, focus, blur — does the component respond correctly?
- **Accessibility:** Are roles, labels, and keyboard navigation correct?
- **Edge cases:** Empty data, null props, extremely long text
- **Error boundaries:** Does it fail gracefully?

### Example with React Testing Library

```typescript
import { render, screen, fireEvent } from '@testing-library/react';

test('Button calls onClick when clicked', () => {
  const handleClick = jest.fn();
  render(<Button onClick={handleClick}>Click me</Button>);
  
  fireEvent.click(screen.getByRole('button', { name: /click me/i }));
  expect(handleClick).toHaveBeenCalledTimes(1);
});

test('Button is keyboard accessible', () => {
  const handleClick = jest.fn();
  render(<Button onClick={handleClick}>Click me</Button>);
  
  const button = screen.getByRole('button');
  button.focus();
  fireEvent.keyDown(button, { key: 'Enter' });
  expect(handleClick).toHaveBeenCalled();
});
```

## Documentation

Good components are self-documenting, but explicit documentation helps:

### JSDoc for Inline Documentation

```typescript
/**
 * A flexible button component supporting multiple variants and sizes.
 * 
 * @example
 * <Button variant="primary" size="lg" onClick={handleClick}>
 *   Submit
 * </Button>
 */
interface ButtonProps {
  /** Visual style variant */
  variant?: 'primary' | 'secondary' | 'danger';
  /** Size of the button */
  size?: 'sm' | 'md' | 'lg';
  /** Click handler */
  onClick?: () => void;
}
```

### Storybook for Visual Documentation

Create stories showing all states and variants:

```typescript
export default {
  title: 'Components/Button',
  component: Button,
};

export const Primary = () => <Button variant="primary">Primary</Button>;
export const Loading = () => <Button isLoading>Loading...</Button>;
export const Disabled = () => <Button disabled>Disabled</Button>;
```

## Common Pitfalls

### Avoid Prop Drilling

If you're passing the same prop through 3+ levels, consider Context or composition:

```typescript
const ThemeContext = createContext<Theme>(defaultTheme);

const Button = () => {
  const theme = useContext(ThemeContext);
  return <button style={{ color: theme.primary }}>Click</button>;
};
```

### Avoid Boolean Prop Explosions

Instead of `isLarge`, `isSmall`, `isMedium`, use:

```typescript
size?: 'sm' | 'md' | 'lg'
```

### Avoid Overly Generic Components

A `<DataDisplay>` that handles tables, lists, cards, and grids will be unmaintainable. Create focused components.

### Don't Sacrifice Accessibility for Aesthetics

If a design requires removing focus indicators or making text too small, push back. Accessibility isn't negotiable.

## Workflow

1. **Define the API first:** What props does this component need? What states must it handle?
2. **Type it strictly:** Use TypeScript to encode constraints and relationships.
3. **Build accessibility in:** Semantic HTML, ARIA, keyboard support from the start.
4. **Handle all states:** Empty, loading, error, success.
5. **Test with real interactions:** Click, type, navigate with keyboard, test with screen reader.
6. **Document with examples:** JSDoc, Storybook, or inline comments showing real usage.
7. **Refactor ruthlessly:** If a component grows beyond 200 lines or has more than 10 props, split it.
