---
name: frontend-clean-code-standards
description: >
  Enforce strict clean code rules for frontend development. Use this for any frontend code review,
  component creation, refactoring, file organization, naming decisions, props typing, folder structure design,
  or when establishing code standards. Applies to React, TypeScript, JavaScript, Vue, Angular, or any
  component-based frontend work. Covers naming conventions, component structure, props typing, folder
  architecture, readability standards, and anti-patterns to avoid.
---

# Frontend Clean Code Standards

This skill enforces absolute, non-negotiable clean code standards for frontend development. These rules exist because clean code is not aesthetic preference—it is risk management. Most engineering effort happens after initial release. Every confusing component, leaky abstraction, and hidden side effect becomes a tax paid repeatedly during onboarding, code reviews, incident fixes, and feature work.

## Naming Conventions

### Files and Folders

**Folders must use kebab-case:**
- `user-profile/`, `shopping-cart/`, `payment-gateway/`
- Never: `UserProfile/`, `shoppingCart/`, `Shopping_Cart/`

**Component files must use PascalCase:**
- `UserProfile.tsx`, `ShoppingCart.tsx`, `PaymentButton.tsx`
- Never: `userProfile.tsx`, `shopping-cart.tsx`

**Non-component files must use camelCase:**
- `formatDate.ts`, `useAuth.ts`, `apiClient.ts`
- Never: `FormatDate.ts`, `format-date.ts`

**Suffixes must indicate file purpose:**
- Types: `UserProfile.types.ts`
- Constants: `UserProfile.constants.ts`
- Utils: `UserProfile.utils.ts`
- Hooks: `useUserProfile.ts` (prefix `use`)
- Tests: `UserProfile.test.tsx` (co-located with source)
- Stores: `userProfile.store.ts` or `useUserProfileStore.ts`

### Variables, Functions, and Types

**Variables and functions must use camelCase:**
```typescript
const userList = [];
const isAuthenticated = true;
function validateEmail(email: string) { }
```

**Boolean variables must use is/has/should prefix:**
```typescript
const isLoading = false;
const hasError = true;
const shouldRender = false;
// Never: const loading, const error, const render
```

**Types and interfaces must use PascalCase:**
```typescript
type UserProfile = { };
interface ButtonProps { }
// Never: type userProfile, interface buttonProps
```

**Props types must end with `Props`:**
```typescript
type ButtonProps = { label: string };
type UserCardProps = { user: User };
// Never: type ButtonProperties, type Button
```

**Names must reveal intent immediately:**
```typescript
// Good
const filteredActiveUsers = users.filter(u => u.isActive);
const userCreatedAt = new Date();

// Bad
const data = users.filter(u => u.isActive);
const temp = new Date();
```

## Component Structure

### File Organization

**Component files must follow this exact order:**

1. Imports (grouped: external, internal, types, styles)
2. Types and interfaces
3. Constants (component-level)
4. Component definition
5. Exports

```typescript
// 1. Imports
import { useState, useEffect } from 'react';
import { Button } from '@/components/ui';
import type { User } from '@/types';
import styles from './UserCard.module.css';

// 2. Types
type UserCardProps = {
  user: User;
  onEdit: (id: string) => void;
};

// 3. Constants
const MAX_BIO_LENGTH = 200;

// 4. Component
export function UserCard({ user, onEdit }: UserCardProps) {
  const [isExpanded, setIsExpanded] = useState(false);
  
  return (
    <div className={styles.card}>
      {/* ... */}
    </div>
  );
}
```

### Component Size and Responsibility

**Components must do one thing:**
- A component handles one feature or one UI concern
- If a component does multiple unrelated things, split it
- If you cannot name a component without using "and", it is doing too much

**Components must be small:**
- Target: under 150 lines including types and logic
- Maximum: 250 lines (if exceeded, refactor)
- Extract helpers, hooks, and sub-components aggressively

**Blocks within conditionals must be one line:**
```typescript
// Good
if (isLoading) return <Spinner />;
if (error) return <ErrorMessage error={error} />;

// Acceptable
if (isLoading) {
  return <Spinner />;
}

// Bad - extract to component
if (isLoading) {
  return (
    <div>
      <Spinner />
      <p>Loading...</p>
      <Button>Cancel</Button>
    </div>
  );
}
```

### Props and State

**Props must be explicitly typed:**
```typescript
// Good
type ButtonProps = {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
};

function Button({ label, onClick, variant = 'primary' }: ButtonProps) { }

// Never use implicit any or loose types
// Bad: function Button(props) { }
// Bad: function Button({ label, onClick }: any) { }
```

**Optional props must have default values:**
```typescript
type CardProps = {
  title: string;
  subtitle?: string;
  elevation?: number;
};

function Card({ title, subtitle = '', elevation = 1 }: CardProps) { }
```

**Children must be explicitly typed:**
```typescript
import type { ReactNode } from 'react';

type ContainerProps = {
  children: ReactNode;
};

function Container({ children }: ContainerProps) {
  return <div>{children}</div>;
}
```

**Destructure props at function signature:**
```typescript
// Good
function UserCard({ user, onEdit }: UserCardProps) { }

// Bad
function UserCard(props: UserCardProps) {
  const { user, onEdit } = props;
}
```

**State must have explicit types when not inferred:**
```typescript
// Good - inferred
const [count, setCount] = useState(0);
const [name, setName] = useState('');

// Good - explicit when needed
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);

// Bad - no type safety
const [data, setData] = useState();
```

## TypeScript Standards

### Type Definitions

**Use `type` for props and simple shapes:**
```typescript
type ButtonProps = {
  label: string;
  onClick: () => void;
};
```

**Use `interface` for extensible contracts:**
```typescript
interface BaseEntity {
  id: string;
  createdAt: Date;
}

interface User extends BaseEntity {
  name: string;
  email: string;
}
```

**Never use `any`:**
```typescript
// Bad
function process(data: any) { }

// Good - use unknown and narrow
function process(data: unknown) {
  if (typeof data === 'string') {
    // data is string here
  }
}

// Good - use generics
function process<T>(data: T) { }
```

**Prefer union types over enums for string constants:**
```typescript
// Good
type Status = 'idle' | 'loading' | 'success' | 'error';

// Acceptable but verbose
enum Status {
  Idle = 'idle',
  Loading = 'loading',
  Success = 'success',
  Error = 'error',
}
```

### Props Typing Patterns

**Spread native element props when wrapping:**
```typescript
import type { ComponentPropsWithoutRef } from 'react';

type ButtonProps = ComponentPropsWithoutRef<'button'> & {
  variant?: 'primary' | 'secondary';
};

function Button({ variant = 'primary', ...props }: ButtonProps) {
  return <button {...props} className={variant} />;
}
```

**Use discriminated unions for conditional props:**
```typescript
type ButtonProps =
  | { variant: 'link'; href: string }
  | { variant: 'button'; onClick: () => void };

function Button(props: ButtonProps) {
  if (props.variant === 'link') {
    return <a href={props.href}>Link</a>;
  }
  return <button onClick={props.onClick}>Button</button>;
}
```

**Generic components must constrain types:**
```typescript
type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => ReactNode;
};

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map(renderItem)}</ul>;
}
```

## Folder Structure

### Recommended Architecture

**Feature-based structure scales better than type-based:**

```
src/
├── app/                    # Next.js app router OR entry points
├── components/
│   ├── ui/                 # Generic primitives (Button, Input, Card)
│   └── layout/             # Layout components (Header, Footer, Sidebar)
├── features/               # Feature-specific modules
│   ├── auth/
│   │   ├── components/     # Auth-specific components
│   │   ├── hooks/          # useAuth, useLogin
│   │   ├── api/            # Auth API calls
│   │   └── types/          # Auth types
│   └── checkout/
│       ├── components/
│       ├── hooks/
│       └── utils/
├── hooks/                  # Global hooks (useMediaQuery, useDebounce)
├── lib/                    # External service wrappers (apiClient, analytics)
├── types/                  # Global types
├── utils/                  # Pure utilities (formatDate, parseJSON)
└── styles/                 # Global styles, theme, design tokens
```

**Why feature-based wins:**
- Changes to checkout touch only `features/checkout/`
- New developers understand features, not technical categories
- Features can be extracted to separate packages easily
- Reduces cross-folder navigation by 70%

### Folder Rules

**Component folders must be self-contained:**
```
user-profile/
├── UserProfile.tsx
├── UserProfile.types.ts
├── UserProfile.test.tsx
├── UserProfile.module.css
└── components/             # Sub-components used ONLY by UserProfile
    └── UserAvatar.tsx
```

**One level of nesting maximum for sub-components:**
- `components/user-profile/components/UserAvatar.tsx` ✓
- `components/user-profile/components/avatar/AvatarImage.tsx` ✗ (too deep, flatten or extract)

**If used by 2+ components, promote to shared:**
- Move from `features/auth/components/` to `components/ui/`
- Move from `features/auth/hooks/` to `hooks/`

**Public API pattern for complex components:**
```typescript
// user-profile/index.ts
export { UserProfile } from './UserProfile';
export type { UserProfileProps } from './UserProfile.types';

// Other files (types, utils, hooks) are NOT exported
// Forces clean boundaries
```

## Code Readability Standards

### Function and Variable Clarity

**Functions must have single responsibility:**
```typescript
// Bad - does three things
function processUser(user: User) {
  validateUser(user);
  const total = calculateTotal(user.orders);
  sendEmail(user.email, total);
}

// Good - each function does one thing
function processUser(user: User) {
  validateUser(user);
  const total = calculateOrderTotal(user.orders);
  notifyUserOfTotal(user.email, total);
}
```

**Extract magic numbers to named constants:**
```typescript
// Bad
if (user.age > 18 && user.score >= 750) { }

// Good
const MINIMUM_AGE = 18;
const CREDIT_SCORE_THRESHOLD = 750;

if (user.age > MINIMUM_AGE && user.score >= CREDIT_SCORE_THRESHOLD) { }
```

**Avoid deep nesting—extract or return early:**
```typescript
// Bad
function process(user: User) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        // do work
      }
    }
  }
}

// Good
function process(user: User | null) {
  if (!user) return;
  if (!user.isActive) return;
  if (!user.hasPermission) return;
  
  // do work
}
```

### Comments and Documentation

**Code must be self-documenting:**
- Good names eliminate most comment needs
- If you need a comment to explain what code does, rename variables/functions instead

**When comments are needed:**
```typescript
// Good - explains WHY, not WHAT
// Use debounce to prevent API rate limiting (max 10 req/sec)
const debouncedSearch = useDebounce(searchTerm, 300);

// Bad - repeats what code already says
// Set loading to true
setLoading(true);
```

**JSDoc for public APIs only:**
```typescript
/**
 * Formats a date according to user's locale.
 * Falls back to ISO format if locale is unavailable.
 */
export function formatDate(date: Date, locale?: string): string { }
```

### Consistency

**Line length must not exceed 120 characters:**
- Configure Prettier: `"printWidth": 120`
- Enforced by ESLint: `max-len`

**Use Prettier for formatting:**
- No manual spacing, indentation, or bracket placement debates
- Commit formatted code only

**Use ESLint for code quality:**
- Enable `@typescript-eslint/recommended`
- Enable `react-hooks/recommended`
- Enable `no-console` for production

## Anti-Patterns to Avoid

### Forbidden Patterns

**Never use `useEffect` for derived state:**
```typescript
// Bad
const [count, setCount] = useState(0);
const [doubleCount, setDoubleCount] = useState(0);

useEffect(() => {
  setDoubleCount(count * 2);
}, [count]);

// Good
const [count, setCount] = useState(0);
const doubleCount = count * 2;
```

**Never mutate props:**
```typescript
// Bad
function UserCard({ user }: { user: User }) {
  user.name = user.name.toUpperCase(); // MUTATION
  return <div>{user.name}</div>;
}

// Good
function UserCard({ user }: { user: User }) {
  const displayName = user.name.toUpperCase();
  return <div>{displayName}</div>;
}
```

**Never use index as key in lists:**
```typescript
// Bad
{items.map((item, index) => <Item key={index} {...item} />)}

// Good
{items.map(item => <Item key={item.id} {...item} />)}
```

**Never define components inside components:**
```typescript
// Bad
function Parent() {
  function Child() { return <div>Child</div>; }
  return <Child />;
}

// Good
function Child() { return <div>Child</div>; }

function Parent() {
  return <Child />;
}
```

**Never use inline object/array literals in props:**
```typescript
// Bad - creates new reference every render
<Component style={{ margin: 10 }} items={[1, 2, 3]} />

// Good
const STYLE = { margin: 10 };
const ITEMS = [1, 2, 3];

<Component style={STYLE} items={ITEMS} />
```

**Never use `var`:**
```typescript
// Bad
var count = 0;

// Good
const count = 0;
let mutableCount = 0;
```

**Never use `==` or `!=`:**
```typescript
// Bad
if (value == null) { }

// Good
if (value === null || value === undefined) { }
// Or
if (value == null) { } // Only exception: checking both null and undefined
```

### Performance Anti-Patterns

**Never fetch in loops:**
```typescript
// Bad
for (const id of userIds) {
  const user = await fetchUser(id);
}

// Good
const users = await fetchUsers(userIds);
```

**Never create functions in render:**
```typescript
// Bad
function List({ items }: { items: Item[] }) {
  return items.map(item => {
    const handler = () => console.log(item.id);
    return <Item onClick={handler} />;
  });
}

// Good
function List({ items }: { items: Item[] }) {
  const handleClick = (id: string) => console.log(id);
  return items.map(item => (
    <Item onClick={() => handleClick(item.id)} />
  ));
}
```

## Enforcement

These standards are not suggestions. Enforce them with:

- **ESLint**: Catch violations at write-time
- **Prettier**: Eliminate formatting debates
- **TypeScript strict mode**: `strict: true` in `tsconfig.json`
- **Pre-commit hooks**: Husky + lint-staged to block non-compliant commits
- **Code review**: Reject PRs that violate these rules

Clean code is a discipline. It reflects thoughtful engineering and respect for future maintainers. These rules exist because they have been proven to reduce bugs, accelerate onboarding, and enable safe refactoring at scale.