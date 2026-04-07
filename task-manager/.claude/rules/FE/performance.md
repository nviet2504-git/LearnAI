---
name: frontend-performance-rules
description: >
  Enforce strict frontend performance rules covering bundling optimization
  (tree-shaking, code-splitting, minification), lazy loading (routes, components,
  images), memoization (React.memo, useMemo, useCallback), re-render control
  (avoiding unnecessary renders, stable references), asset optimization (image
  compression, font loading, CSS purging), and prohibited slow patterns (inline
  functions in JSX, massive Context providers, synchronous heavy computations).
  Use this skill when building or reviewing frontend code, React applications,
  web performance optimization, bundle analysis, Core Web Vitals improvement,
  or when the user mentions performance, speed, bundle size, re-renders,
  optimization, lazy loading, memoization, or wants to prevent slow patterns
  in their frontend codebase.
---

# Frontend Performance Rules

Enforce production-grade performance standards for modern frontend applications. These rules prevent common bottlenecks and ensure fast, scalable user experiences.

## Performance Philosophy

Performance is not an afterthought—it's a constraint. Every millisecond matters for Core Web Vitals, conversion rates, and user retention. The goal: ship only what users need, when they need it, in the smallest possible form.

**Measure first, optimize second.** Use React DevTools Profiler, Lighthouse, and bundle analyzers to identify actual bottlenecks before applying fixes.

## Bundling Optimization

### Tree-Shaking (Dead Code Elimination)

**Goal:** Remove unused exports from production bundles.

**Rules:**
- Use ES6 `import`/`export` syntax exclusively—tree-shaking fails with CommonJS (`require`/`module.exports`)
- Avoid Babel plugins that transform ESM to CommonJS (e.g., `@babel/plugin-transform-modules-commonjs`)
- Import specific exports, not entire libraries: `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`
- Mark side-effect-free packages in `package.json`: `"sideEffects": false` or `"sideEffects": ["*.css"]`
- Enable production mode in bundlers (Webpack, Vite, Rollup)—tree-shaking only runs in production

**Why:** Unused code inflates bundle size. A 500KB library where you use one function still ships 500KB unless tree-shaken. ES6 modules are statically analyzable; CommonJS is not.

### Code-Splitting

**Goal:** Split bundles into smaller chunks loaded on-demand.

**Rules:**
- **Route-based splitting:** Lazy-load route components via dynamic imports
  ```jsx
  const Dashboard = lazy(() => import('./Dashboard'));
  ```
- **Component-based splitting:** Defer heavy components (charts, editors, modals) until needed
- **Vendor splitting:** Separate third-party libraries into a stable vendor chunk to leverage browser caching
- **Async boundaries:** Wrap lazy components in `<Suspense>` with meaningful fallbacks
- **Avoid:** Splitting tiny modules (<20KB)—overhead of extra HTTP requests outweighs benefits

**Why:** Users shouldn't download the admin panel code if they never visit it. Smaller initial bundles = faster Time to Interactive (TTI).

### Minification & Compression

**Rules:**
- Enable Terser (Webpack) or esbuild minification in production
- Serve Brotli-compressed assets (better than gzip for text)
- Set bundle size budgets: warn at 244KB per chunk, error at 500KB
- Run `webpack-bundle-analyzer` or equivalent monthly to audit bundle composition

## Lazy Loading

### Routes

```jsx
const routes = [
  { path: '/', component: lazy(() => import('./Home')) },
  { path: '/admin', component: lazy(() => import('./Admin')) },
];
```

### Components

Defer non-critical UI:
```jsx
const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<Skeleton />}>
      <HeavyChart data={data} />
    </Suspense>
  );
}
```

### Images

- Use native lazy loading: `<img loading="lazy" />`
- For complex scenarios, use Intersection Observer or libraries like `react-lazy-load-image-component`
- Lazy-load offscreen images only—above-the-fold images must load immediately

**Why:** Lazy loading cuts initial payload. Images are often the largest assets; deferring offscreen images improves LCP (Largest Contentful Paint).

## Memoization & Re-render Control

### When to Memoize

Memoization prevents wasted re-computation and re-renders. Apply when:
- Component renders frequently with identical props
- Expensive calculations run on every render
- Callbacks passed to memoized children change identity

**Don't memoize blindly**—it adds complexity and memory overhead. Profile first.

### React.memo

Prevents component re-render if props haven't changed (shallow comparison).

```jsx
const ExpensiveChild = React.memo(({ data, onClick }) => {
  // Only re-renders if data or onClick changes
  return <div onClick={onClick}>{data.value}</div>;
});
```

**Pitfall:** Breaks if props are new objects/arrays/functions on every render. Combine with `useMemo`/`useCallback`.

### useMemo

Caches expensive computations.

```jsx
const sortedData = useMemo(
  () => data.sort((a, b) => a.value - b.value),
  [data]
);
```

**Use for:**
- Expensive transformations (sorting, filtering large arrays)
- Derived objects passed to memoized children

**Avoid for:**
- Cheap calculations (simple arithmetic, string concatenation)
- Values not used in dependencies of other hooks or passed as props

### useCallback

Stabilizes function identity.

```jsx
const handleClick = useCallback(
  (id) => { dispatch({ type: 'SELECT', id }); },
  [dispatch]
);
```

**Use when:**
- Passing callbacks to `React.memo` components
- Callbacks are dependencies of `useEffect` or other hooks

**Avoid:**
- Functions not passed as props or used in dependencies
- Functions defined outside components (already stable)

### Re-render Control Patterns

**Prohibited:**
- Inline object/array literals in props: `<Child config={{ theme: 'dark' }} />`—creates new reference every render
- Inline functions in JSX: `<button onClick={() => doThing()} />`—breaks memoization
- Massive Context providers wrapping entire app—every context change triggers all consumers

**Preferred:**
- Hoist stable objects outside components or memoize them
- Split Context by concern (auth, theme, data)—limits re-render scope
- Use state management libraries (Zustand, Jotai) with selective subscriptions

**Why:** React re-renders a component and all children when state/props change. Unnecessary re-renders compound exponentially in deep trees. Memoization + stable references break the cascade.

## Asset Optimization

### Images

- **Format:** Use WebP or AVIF with JPEG fallback
- **Compression:** Optimize with tools like Squoosh, ImageOptim
- **Responsive:** Serve appropriately sized images via `srcset` or `<picture>`
- **Dimensions:** Always set width/height to prevent layout shift (CLS)
- **Avoid:** Base64 inline images (bloats HTML, non-cacheable)

### Fonts

- **Subset fonts:** Include only needed characters/weights
- **Preload critical fonts:** `<link rel="preload" as="font" href="font.woff2" crossorigin />`
- **font-display:** Use `swap` to show fallback text immediately
- **Limit font families:** Each font = extra HTTP request

### CSS

- **Purge unused CSS:** Use PurgeCSS or Tailwind's built-in purging
- **Critical CSS:** Inline above-the-fold styles, defer the rest
- **Avoid:** Global `*` selectors (hard to tree-shake), deep nesting (increases specificity and file size)

## Prohibited Slow Patterns

### ❌ Inline Functions in JSX (High-Frequency Renders)

```jsx
// Bad: Creates new function every render
<button onClick={() => handleClick(id)}>Click</button>

// Good: Stable reference
const onClick = useCallback(() => handleClick(id), [id]);
<button onClick={onClick}>Click</button>
```

### ❌ Synchronous Heavy Computations in Render

```jsx
// Bad: Blocks render
function Component({ data }) {
  const result = expensiveSync(data); // 200ms computation
  return <div>{result}</div>;
}

// Good: Memoize or move to Web Worker
const result = useMemo(() => expensiveSync(data), [data]);
```

### ❌ Mutating Props/State

```jsx
// Bad: Breaks memoization
const items = props.items;
items.push(newItem); // Mutation!

// Good: Immutable update
const items = [...props.items, newItem];
```

### ❌ Large Context Providers with Frequent Updates

```jsx
// Bad: Every state change re-renders all consumers
<AppContext.Provider value={{ user, theme, data, settings }}>
  <App />
</AppContext.Provider>

// Good: Split by update frequency
<UserContext.Provider value={user}>
  <ThemeContext.Provider value={theme}>
    <App />
  </ThemeContext.Provider>
</UserContext.Provider>
```

### ❌ Missing Dependency Arrays

```jsx
// Bad: Runs every render
useEffect(() => { fetchData(); }); // No dependency array

// Good: Runs only when id changes
useEffect(() => { fetchData(id); }, [id]);
```

### ❌ Importing Entire Libraries

```jsx
// Bad: Imports all of lodash (~70KB)
import _ from 'lodash';

// Good: Imports only debounce (~2KB)
import debounce from 'lodash-es/debounce';
```

## Measurement & Monitoring

**Before optimizing:**
1. Profile with React DevTools Profiler—record interaction, identify wasted renders
2. Run Lighthouse—check LCP, FID, CLS scores
3. Analyze bundles—use `webpack-bundle-analyzer` or `source-map-explorer`

**After optimizing:**
- Verify improvements in production (not dev mode—React runs slower in dev)
- Monitor Core Web Vitals in production with Real User Monitoring (RUM)

**Targets (2026 standards):**
- LCP: <1.2s (good), <2.5s (acceptable)
- FID/INP: <100ms (good), <200ms (acceptable)
- CLS: <0.1 (good), <0.25 (acceptable)
- Initial bundle: <200KB (compressed)

## Decision Framework

**Bundling:**
- Route visited by <30% of users? → Lazy-load it
- Library >50KB with partial usage? → Import specific exports or find lighter alternative
- Bundle >500KB? → Split vendors, analyze with bundle analyzer

**Memoization:**
- Component re-renders >10 times with same props? → `React.memo`
- Calculation takes >5ms? → `useMemo`
- Callback passed to memoized child? → `useCallback`
- Simple, fast operations? → Skip memoization

**Assets:**
- Image >100KB? → Compress, use WebP, implement lazy loading
- Font >50KB? → Subset or switch to system fonts
- CSS >50KB after minification? → Purge unused styles

## React Compiler (React 19+)

The React Compiler automatically memoizes components and values, reducing manual `useMemo`/`useCallback` boilerplate. It works best with:
- Pure functional components
- Immutable updates
- Minimal side effects

**Limitations:** Can't optimize impure components, external mutations, or architectural issues (e.g., massive Context providers). Still apply splitting, lazy loading, and asset optimization manually.

## Summary Checklist

- [ ] ES6 imports/exports for tree-shaking
- [ ] Route-based code-splitting with `lazy()` + `<Suspense>`
- [ ] Lazy-load images with `loading="lazy"`
- [ ] `React.memo` for expensive components with stable props
- [ ] `useMemo` for expensive computations
- [ ] `useCallback` for callbacks passed to memoized children
- [ ] No inline objects/functions in JSX (high-frequency renders)
- [ ] Split Context providers by concern
- [ ] Optimized images (WebP, compressed, responsive)
- [ ] Purged CSS, subsetted fonts
- [ ] Bundle size <200KB initial (compressed)
- [ ] Lighthouse score: LCP <1.2s, CLS <0.1

Performance is a feature. Ship fast code.