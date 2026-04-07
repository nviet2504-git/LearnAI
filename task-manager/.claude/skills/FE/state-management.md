---
name: react-state-management-expert
description: >
  Expert guidance for architecting state in React applications. Use this for any state management decisions, questions, or implementations including choosing between Redux, Zustand, Recoil, Jotai, or Context API; deciding global vs local state placement; separating server state from UI state; optimizing re-renders; managing derived state; handling async state; form state patterns; and performance optimization. Covers TanStack Query for server state, when to use Context vs external libraries, selector patterns, atomic state design, and avoiding common pitfalls like prop drilling, unnecessary re-renders, or mixing state concerns. Essential for React apps of any size dealing with state architecture, refactoring legacy state code, or scaling state patterns.
---

# React State Management Expert

## Core Principle: Match State to Its Nature

State management isn't about picking one library—it's about recognizing that different kinds of state need different tools. Mixing server data, UI toggles, and form inputs into one global store creates stale data, scattered loading flags, and performance issues.

### The Four State Categories

**Server State** — Data owned by the backend (user lists, posts, dashboard metrics)  
Needs: caching, refetching, invalidation, background updates  
**Tool:** TanStack Query (React Query), SWR  
**Never:** Redux, Zustand, Context for API responses

**Global UI State** — Client-only data shared across features (auth status, theme, sidebar open/closed)  
Needs: simple reads/writes, persistence, devtools  
**Tool:** Zustand (small/medium apps), Redux Toolkit (large/complex apps), Context (rarely changing data)

**Local UI State** — Component-specific ephemeral data (accordion expanded, modal visible)  
Needs: minimal overhead, colocation with component  
**Tool:** `useState`, `useReducer`

**Form State** — Temporary interactive data with validation, dirty tracking, field-level errors  
Needs: uncontrolled performance, validation hooks, submission handling  
**Tool:** React Hook Form, Formik

## Decision Tree: Choosing Your State Tool

### Is this data from an API?

**Yes** → Use TanStack Query or SWR. Do not store API responses in Redux/Zustand/Context. Server state libraries handle caching, staleness, and synchronization automatically. Your global store should only hold client-side UI flags (like `isAuthenticated`), not the user object itself.

**No** → Continue below.

### Does this state change frequently (multiple times per second)?

**Yes** → Use local `useState` or `useReducer`. Examples: text input, slider position, animation frame. Avoid global stores for high-frequency updates—they trigger broad re-renders.

**No** → Continue below.

### Is this state needed across multiple unrelated features?

**Yes** → Use Zustand (if team prefers simplicity) or Redux Toolkit (if you need strict patterns, time-travel debugging, or large team coordination). Jotai/Recoil work well for atomic, derived state patterns.

**No** → Use local state or lift to nearest common parent. Avoid premature globalization.

### Is this a form with 5+ fields or complex validation?

**Yes** → Use React Hook Form. It uses refs (uncontrolled) by default, minimizing re-renders. Integrates easily with validation libraries like Zod.

**No** → `useState` is sufficient.

## Library Quick Reference

### TanStack Query (React Query)

**When:** Any data from an API  
**Why:** Automatic caching, background refetching, request deduplication, optimistic updates, infinite scroll  
**Pattern:**

```tsx
const { data, isLoading, error } = useQuery({
  queryKey: ['users', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000, // 5 minutes
});
```

Set `staleTime` based on how often data changes. Short for live dashboards, long for static content.

### Zustand

**When:** Simple global UI state (theme, auth flags, sidebar state)  
**Why:** Minimal API (~1KB), no providers, works outside React, built-in devtools/persistence middleware  
**Pattern:**

```tsx
import { create } from 'zustand';

const useStore = create<State>((set) => ({
  theme: 'dark',
  sidebarOpen: false,
  setTheme: (theme) => set({ theme }),
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));

// In component - use selectors to avoid unnecessary re-renders
const theme = useStore((s) => s.theme);
const toggleSidebar = useStore((s) => s.toggleSidebar);
```

**Key:** Always use selectors. `useStore()` subscribes to entire store.

### Redux Toolkit

**When:** Large apps, multiple teams, strict architectural requirements, time-travel debugging  
**Why:** Predictable state flow, excellent devtools, massive ecosystem, enforced patterns  
**Pattern:**

```tsx
const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, isAuthenticated: false },
  reducers: {
    login: (state, action) => {
      state.isAuthenticated = true; // Immer handles immutability
    },
    logout: (state) => {
      state.isAuthenticated = false;
      state.user = null;
    },
  },
});
```

**Avoid** for new small/medium projects—Zustand is simpler. Redux shines when you need structure at scale.

### Jotai / Recoil

**When:** Complex derived state, fine-grained reactivity, atomic state patterns  
**Why:** Bottom-up composition, automatic dependency tracking, only re-renders components using changed atoms  
**Pattern (Jotai):**

```tsx
const countAtom = atom(0);
const doubleAtom = atom((get) => get(countAtom) * 2);

// Component only re-renders when doubleAtom changes
const double = useAtomValue(doubleAtom);
```

**Trade-off:** More concepts to learn (atoms, selectors). Best when you have many small, interdependent pieces of state.

### Context API

**When:** Rarely changing global data (theme, locale, auth status) with few consumers  
**Why:** Built-in, zero dependencies  
**Avoid when:** State changes frequently or has many consumers—Context triggers re-renders of all children when value changes, even with memoization.

**Pattern:**

```tsx
const ThemeContext = createContext<Theme>('light');

// Wrap once at root
<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>

// Consume anywhere
const theme = useContext(ThemeContext);
```

**Pitfall:** Wrapping `value` in an object creates new reference every render. Use `useMemo`.

## Preventing Unnecessary Re-renders

### Selector Pattern (Zustand, Redux)

Subscribe only to the slice of state you need:

```tsx
// Bad - re-renders on any store change
const store = useStore();

// Good - re-renders only when theme changes
const theme = useStore((s) => s.theme);
```

### Atomic Subscriptions (Jotai, Recoil)

Components automatically subscribe only to atoms they read:

```tsx
const count = useAtomValue(countAtom); // Only re-renders when countAtom changes
```

### Memoization

Use `useMemo` for expensive derived state, `useCallback` for functions passed to optimized children:

```tsx
const filteredItems = useMemo(
  () => items.filter((item) => item.active),
  [items]
);
```

### Code Splitting State

Don't put everything in one store. Create feature-specific stores:

```tsx
// ❌ One giant store
const useAppStore = create(() => ({ auth, cart, ui, settings, ... }));

// ✅ Separate concerns
const useAuthStore = create(...);
const useCartStore = create(...);
```

## Server State vs UI State: The Critical Separation

Most "global state" in modern apps is actually server state. Storing API responses in Redux/Zustand creates:

- **Stale data** — No automatic refetching when data changes on server  
- **Cache duplication** — Same data fetched multiple times  
- **Boilerplate** — Manual loading/error states, race condition handling  
- **Invalidation complexity** — When to refetch? On mount? On focus? Manual triggers?

TanStack Query solves all of this:

```tsx
// ❌ Old pattern: API data in Redux
const user = useSelector((state) => state.user);
useEffect(() => {
  dispatch(fetchUser(userId));
}, [userId]);

// ✅ Modern pattern: TanStack Query
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => api.getUser(userId),
});
```

**What stays in global UI state:**  
`isAuthenticated`, `theme`, `sidebarOpen`, `activeModal`, user preferences

**What moves to TanStack Query:**  
User profile, posts, comments, dashboard data, search results, any API response

## Common Pitfalls

### Pitfall: Storing Derived State

Don't store computed values—derive them:

```tsx
// ❌ Storing derived state
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);

// Must remember to update both
const addItem = (item) => {
  setItems([...items, item]);
  setTotal(total + item.price); // Easy to forget or desync
};

// ✅ Derive on render
const [items, setItems] = useState([]);
const total = useMemo(
  () => items.reduce((sum, item) => sum + item.price, 0),
  [items]
);
```

### Pitfall: Premature Global State

Start local, lift when needed:

```tsx
// ❌ Immediately global
const useModalStore = create((set) => ({
  isOpen: false,
  toggle: () => set((s) => ({ isOpen: !s.isOpen })),
}));

// ✅ Start local, only globalize if multiple features need it
const [isOpen, setIsOpen] = useState(false);
```

### Pitfall: Context for Frequent Updates

Context triggers re-renders of all consumers. For frequent changes, use Zustand/Jotai:

```tsx
// ❌ Slider position in Context - all consumers re-render constantly
const [volume, setVolume] = useState(50);
<VolumeContext.Provider value={{ volume, setVolume }}>

// ✅ Local state or Zustand with selector
const volume = useStore((s) => s.volume);
```

### Pitfall: Mixing Server and Client State

Keep them separate:

```tsx
// ❌ API response in Zustand
const useStore = create((set) => ({
  users: [],
  fetchUsers: async () => {
    const users = await api.getUsers();
    set({ users });
  },
}));

// ✅ Server state in TanStack Query, UI flags in Zustand
const { data: users } = useQuery({ queryKey: ['users'], queryFn: api.getUsers });
const sidebarOpen = useStore((s) => s.sidebarOpen);
```

## Practical Combinations

**Small app (blog, portfolio):**  
`useState` + TanStack Query

**Medium app (dashboard, SaaS):**  
Zustand (UI state) + TanStack Query (server state) + React Hook Form (forms)

**Large app (enterprise, multi-team):**  
Redux Toolkit (complex UI state) + TanStack Query (server state) + React Hook Form (forms)

**Atomic state patterns:**  
Jotai or Recoil + TanStack Query

## Migration Strategy

Refactoring legacy Redux with API data:

1. **Identify server state** — Any data from `fetch`, `axios`, etc.  
2. **Move to TanStack Query** — Replace Redux actions/reducers with `useQuery`/`useMutation`  
3. **Keep UI state in Redux** — Or migrate to Zustand if Redux feels heavy  
4. **Remove boilerplate** — Delete loading/error reducers, TanStack Query handles them

Typically removes 60-80% of Redux code.

## Performance Checklist

- [ ] Server state uses TanStack Query, not global store  
- [ ] Selectors used for Zustand/Redux subscriptions  
- [ ] No derived state stored—computed with `useMemo`  
- [ ] Frequent updates stay local (not in Context or global store)  
- [ ] Forms use React Hook Form for 5+ fields  
- [ ] Code-split stores by feature, not one giant store  
- [ ] `staleTime` configured based on data freshness needs  

## When to Reach for Each Tool

**Fetching user profile, posts, or any API data?**  
→ TanStack Query

**Need theme, auth status, or sidebar state across app?**  
→ Zustand (simple) or Redux Toolkit (complex/large team)

**Building a form with validation?**  
→ React Hook Form

**Component-specific toggle or input?**  
→ `useState`

**Complex derived state with many dependencies?**  
→ Jotai or Recoil

**Passing static config down tree?**  
→ Context API

The best state architecture uses the smallest, most specific tool for each job. Don't force everything into one pattern.