---
name: frontend-performance-optimization
description: >
  Optimize frontend performance for fast, responsive web applications. Use this for bundle analysis,
  React performance tuning (memo, useMemo, code splitting, re-render prevention), image optimization
  (WebP, lazy loading, responsive images), render profiling with React DevTools and Chrome Performance tab,
  caching strategies (HTTP caching, service workers, CDN), Lighthouse audits, Core Web Vitals improvements
  (LCP, INP, CLS), webpack bundle analysis, tree shaking, code splitting, minification, compression,
  lazy loading, prefetching, critical rendering path optimization, or any task involving speeding up
  page load, reducing bundle size, eliminating render bottlenecks, improving Time to Interactive, or
  debugging performance issues. Applies to webpack, Vite, React, Next.js, and general frontend optimization workflows.
---

# Frontend Performance Optimization

## Overview

Frontend performance directly impacts user experience, conversion rates, and SEO. This skill helps diagnose bottlenecks and apply targeted optimizations across bundle size, rendering efficiency, asset delivery, and runtime behavior.

Performance work follows a hierarchy: fix architectural issues first (waterfalls, massive bundles, blocking resources), then optimize rendering (React re-renders, virtualization), then fine-tune (memoization, compression).

## Core Metrics (2026)

**Core Web Vitals** define user-perceived performance:

- **LCP (Largest Contentful Paint)**: Target < 2.5s. When main content becomes visible. Improved by code splitting, image optimization, SSR.
- **INP (Interaction to Next Paint)**: Target < 200ms. Responsiveness to user input. Improved by reducing JS work, memoization, web workers, `useTransition`.
- **CLS (Cumulative Layout Shift)**: Target < 0.1. Visual stability. Improved by explicit image dimensions, font preloading, reserved skeleton space.

INP replaced FID (First Input Delay) as a Core Web Vital in 2024. Prioritize INP over FID in audits.

## Measurement & Profiling

### Before optimizing, measure

**Lab tools (synthetic)**:
- **Lighthouse** (Chrome DevTools): Comprehensive audit with actionable recommendations. Run in production mode.
- **WebPageTest**: Advanced testing with filmstrips, waterfall charts, multi-location tests.
- **React DevTools Profiler**: Records component render times, identifies what triggered each render, highlights expensive components.
- **Chrome Performance tab**: Deep browser-level profiling—JS execution, layout, paint, compositing.

**Real-user monitoring (RUM)**:
- **Web Vitals JS library**: Capture real Core Web Vitals from production users.
- **CrUX (Chrome User Experience Report)**: Google's field data used for SEO rankings. Check Google Search Console.
- **Monitoring platforms**: Sentry, Datadog, New Relic, OneUptime for production performance tracking.

Use lab tools for controlled baselines; use RUM to understand real-world impact.

### Profiling workflow

1. Run Lighthouse to identify which Core Web Vital needs work.
2. Use React DevTools Profiler to find slowest components (if React).
3. Use Chrome Performance tab to drill into JS execution, layout thrashing, long tasks.
4. Apply the appropriate technique from this skill.
5. Re-measure to confirm improvement.

## Bundle Optimization

### Bundle analysis

**Visualize what's in your bundle** to find low-hanging fruit:

- **webpack-bundle-analyzer**: Interactive treemap showing module sizes. Add to webpack config:
  ```js
  const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
  module.exports = {
    plugins: [new BundleAnalyzerPlugin({ analyzerMode: 'static' })]
  };
  ```
- **Vite**: Use `vite-bundle-visualizer` or `rollup-plugin-visualizer`.
- **bundle-buddy**: Identifies duplicate modules across chunks.
- **source-map-explorer**: Analyze bundle composition from source maps.

Look for:
- Large libraries imported in full (e.g., lodash, moment.js with all locales)
- Duplicate dependencies across chunks
- Development builds (React dev mode) in production
- Unnecessary polyfills for modern browsers

### Code splitting & lazy loading

**Ship only what's needed for the current route**:

```js
// React lazy loading
const AdminPanel = React.lazy(() => import('./AdminPanel'));

<Suspense fallback={<Loader />}>
  <AdminPanel />
</Suspense>
```

**Route-based splitting** (most common): Load code per route only when navigated.

**Component-based splitting**: Defer heavy components (charts, editors, modals) until interaction or visibility.

Dynamic imports (`import()`) enable on-demand loading. Webpack/Vite automatically create separate chunks.

### Tree shaking

Eliminate unused code from final bundle:

- Use ES6 modules (`import`/`export`) not CommonJS—tree shaking requires static analysis.
- Enable production mode in bundler (webpack `mode: 'production'`).
- Mark side-effect-free packages in `package.json`: `"sideEffects": false`.
- Import only what you need: `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`.

Webpack 2026 roadmap includes **lazy barrel optimization** (inspired by Rspack): skips building unused re-exports in barrel files until needed, reducing build time and bundle size.

### Minification & compression

**Minification** removes whitespace, comments, shortens variable names:
- Webpack 4+ uses Terser by default in production mode.
- Vite uses esbuild for fast minification.

**Compression** reduces transfer size:
- **Gzip**: Widely supported, ~70% reduction. Use `compression-webpack-plugin` or enable on server/CDN.
- **Brotli**: Better compression than Gzip (~20% smaller), supported by modern browsers. Use `compression-webpack-plugin` with Brotli option.

Always compress text assets (JS, CSS, HTML, JSON). Serve compressed versions with `Content-Encoding` header.

### Avoid common bundle bloat

- **Lodash**: Use `lodash-es` or `babel-plugin-lodash` to import only used functions.
- **Moment.js**: Heavy with all locales. Use `date-fns` or `day.js` instead, or strip unused locales with webpack `ContextReplacementPlugin`.
- **Polyfills**: Use `@babel/preset-env` with `useBuiltIns: 'usage'` to include only needed polyfills. Don't ship ES5 polyfills to modern browsers—use module/nomodule pattern.
- **Development builds**: Ensure React, Vue, etc. use production builds. Check with React DevTools browser extension (icon is dark in production, red in dev).

## React Performance Tuning

### Understanding re-renders

React re-renders a component when:
1. Its state changes.
2. Its props change.
3. Its parent re-renders (even if props haven't changed).
4. Context value changes.

The third case is the biggest source of wasted work. Every optimization either prevents that cascade or makes each render cheaper.

### React.memo

Prevents re-render if props haven't changed (shallow comparison):

```js
const ExpensiveComponent = React.memo(({ data }) => {
  // Only re-renders if data changes
  return <div>{/* complex rendering */}</div>;
});
```

Use selectively—not every component. Best for:
- Components that render frequently with same props (list items, cards).
- Components with expensive rendering logic.

Don't overuse—memoization has a cost (comparison overhead).

### useMemo & useCallback

**useMemo**: Memoizes computed values to avoid recalculating on every render:

```js
const sortedData = useMemo(() => data.sort(sortFn), [data]);
```

**useCallback**: Memoizes function references to prevent child re-renders:

```js
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

Common mistake: overusing these hooks. "You can remove 90% of useMemo/useCallback usage, and your app might run just as well, or even faster." Only use when:
- Passing functions/objects to memoized children.
- Expensive computations (sorting, filtering large arrays).
- Dependencies in other hooks (useEffect, useMemo).

### useRef over useState

If a value doesn't need to trigger re-render (timers, DOM refs, previous values), use `useRef`. It updates without causing re-render, saving work.

### Avoid inline functions & objects in JSX

Inline functions/objects create new references on every render, breaking memoization:

```js
// Bad: new function every render, child re-renders
<Child onClick={() => doSomething()} />

// Good: stable reference
const handleClick = useCallback(() => doSomething(), []);
<Child onClick={handleClick} />
```

Same for objects: `<Child config={{ option: true }} />` creates new object every render.

### List keys

Always use stable, unique keys (IDs) for list items. Never use array indices—they break React's diffing logic and cause inefficient updates or bugs.

```js
{users.map(user => (
  <li key={user.id}>{user.name}</li> // Good
))}
```

### Virtualization for long lists

Rendering thousands of DOM nodes is expensive. **Windowing/virtualization** renders only visible items:

- **react-window** (lightweight) or **react-virtualized** (feature-rich): Render only visible rows, dramatically reducing render time and DOM node count.
- **FlashList** (React Native): 10× faster than FlatList by recycling components.

For lists with 100+ items, virtualization is often the highest-impact optimization.

### Concurrent features (React 18+)

**useTransition**: Mark non-urgent updates as low-priority, keeping UI responsive during heavy rendering:

```js
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setSearchResults(expensiveFilter(query)); // Low-priority
});
```

User input (typing) remains high-priority; background work (filtering 10k items) is interruptible.

**useDeferredValue**: Similar to useTransition but for values instead of state updates.

Concurrent Mode (enabled by default with `createRoot` in React 18) allows React to interrupt rendering, prioritize critical updates, and keep UI responsive.

### React 19 & Compiler (2026)

React 19 introduces the **React Compiler**: automatically handles memoization, reducing manual `useMemo`/`useCallback`. However, inefficient state placement can still bypass optimizations—keep state close to where it's used, avoid lifting state unnecessarily.

**Server Components (RSC)**: Reduce client-side JS by rendering non-interactive components on server. Streaming enables progressive hydration, improving perceived performance.

## Image Optimization

Images often constitute the largest portion of page weight.

### Modern formats

- **WebP**: ~30% smaller than JPEG/PNG, widely supported. Use with fallback:
  ```html
  <picture>
    <source srcset="image.webp" type="image/webp">
    <img src="image.jpg" alt="Description">
  </picture>
  ```
- **AVIF**: Better compression than WebP, but slower encoding and less browser support. Use for critical images with fallback.

### Responsive images

Serve appropriately sized images for different screen sizes:

```html
<img src="hero.jpg"
     srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1200.jpg 1200w"
     sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
     alt="Hero banner">
```

Never load 2000×2000 image for 200×200 thumbnail. Use server-side resizing or CDN with dynamic transformation (Cloudinary, Imgix).

### Lazy loading

Defer loading offscreen images:

```html
<img src="below-fold.jpg" loading="lazy" alt="Description">
```

Native `loading="lazy"` is widely supported. For above-the-fold images, use `loading="eager"` or omit (default).

### Priority hints

For critical above-the-fold images, use `fetchpriority="high"` to prioritize loading:

```html
<img src="hero.jpg" fetchpriority="high" alt="Hero">
```

Or preload in `<head>`:

```html
<link rel="preload" as="image" href="hero.jpg">
```

### Progressive loading

Load tiny blurred placeholder first, then transition to full-resolution image. Improves perceived performance—users see content immediately.

### Image optimization tools

- **Webpack**: `image-webpack-loader` for compression.
- **Vite**: `vite-plugin-imagemin`.
- **Standalone**: TinyPNG, Squoosh, sharp (Node.js).

## Caching Strategies

### HTTP caching

Leverage browser cache to avoid re-downloading unchanged resources:

- **Cache-Control headers**: `Cache-Control: public, max-age=31536000, immutable` for versioned assets (JS, CSS with hashes in filename).
- **ETag/Last-Modified**: For resources without versioning, use conditional requests.

Use content hashing in filenames (`bundle.a1b2c3.js`) to cache aggressively—new versions get new filenames, old versions cached forever.

### Service Workers

Intercept network requests, cache assets, enable offline functionality:

- **Workbox** (Google): Simplifies service worker setup with caching strategies (cache-first, network-first, stale-while-revalidate).
- Cache static assets (JS, CSS, fonts) on install; serve from cache, update in background.

Service workers enable Progressive Web Apps (PWAs) with offline support and instant loading.

### CDN (Content Delivery Network)

Serve assets from edge servers near users, reducing latency:

- Cloudflare, Fastly, AWS CloudFront, Vercel Edge Network.
- Enable compression (Gzip/Brotli) at CDN level.
- Use CDN for static assets (images, JS, CSS, fonts).

## Critical Rendering Path Optimization

The browser must download HTML, parse it, fetch CSS/JS, construct DOM/CSSOM, render pixels. Optimize this sequence:

### Minimize render-blocking resources

**CSS blocks rendering**—inline critical CSS in `<head>` for above-the-fold content, load non-critical CSS asynchronously:

```html
<style>/* Critical CSS */</style>
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
```

**JS blocks parsing**—use `async` or `defer`:

- `async`: Loads script without blocking parsing, executes as soon as downloaded (order not guaranteed).
- `defer`: Loads script without blocking parsing, executes after DOM parsing completes (order guaranteed).

Use `defer` for scripts that depend on DOM or execution order.

### Preload, prefetch, preconnect

- **Preload**: Fetch critical resources early: `<link rel="preload" href="font.woff2" as="font" crossorigin>`.
- **Prefetch**: Hint browser to fetch resources for next navigation: `<link rel="prefetch" href="next-page.js">`.
- **Preconnect**: Establish early connection to third-party domains: `<link rel="preconnect" href="https://api.example.com">`.

### Reduce DOM size

Large DOM trees slow rendering and layout. Keep DOM shallow and small—avoid deeply nested structures, remove hidden/unused elements.

## Lighthouse Workflow

1. **Run Lighthouse** in production mode (Chrome DevTools > Lighthouse tab). Select Performance, Best Practices, SEO.
2. **Review scores**: 0-49 (poor), 50-89 (needs improvement), 90-100 (good).
3. **Check opportunities**: Lighthouse lists issues ordered by impact (e.g., "Reduce unused JavaScript", "Properly size images").
4. **Check diagnostics**: Additional insights (e.g., "Avoid enormous network payloads", "Minimize main-thread work").
5. **Fix high-impact issues first**: Eliminate waterfalls, reduce bundle size, optimize images.
6. **Re-run Lighthouse**: Confirm improvements.

Lighthouse scores are synthetic (lab data). Validate with real-user metrics (CrUX, RUM) to ensure improvements translate to production.

## Debouncing & Throttling

Limit how often expensive operations execute during frequent events (typing, scrolling, resizing):

**Debounce**: Execute after user stops triggering event for X ms (good for search input):

```js
const handleSearch = debounce((query) => {
  fetch(`/api/search?q=${query}`);
}, 300);
```

**Throttle**: Execute at most once every X ms (good for scroll/resize handlers):

```js
const handleScroll = throttle(() => {
  updateScrollPosition();
}, 100);
```

Use lodash or implement manually. Prevents excessive API calls, re-renders, layout calculations.

## Common Pitfalls

- **Optimizing too early**: Measure first. Don't add memoization everywhere—it has overhead.
- **Ignoring network waterfalls**: If a request waterfall adds 600ms, micro-optimizations won't help. Fix architectural issues first.
- **Lighthouse score obsession**: A 94 Lighthouse score means nothing if real users experience jank. Prioritize real-user metrics (INP, LCP from CrUX).
- **Polyfill bloat**: Don't ship ES5 polyfills to modern browsers. Use differential loading (module/nomodule).
- **Unoptimized images**: High-res images are often the biggest bottleneck. Compress, resize, lazy load.

## Quick Wins Checklist

- [ ] Run Lighthouse, identify top 3 issues.
- [ ] Enable production builds (React, webpack mode).
- [ ] Analyze bundle with webpack-bundle-analyzer, remove large unused libraries.
- [ ] Enable code splitting (route-based minimum).
- [ ] Compress images (WebP, lazy loading, responsive images).
- [ ] Enable Gzip/Brotli compression.
- [ ] Add `Cache-Control` headers for static assets.
- [ ] Use `defer` for non-critical scripts.
- [ ] Profile React components with DevTools Profiler, memoize expensive components.
- [ ] Virtualize long lists (100+ items).
- [ ] Monitor Core Web Vitals in production (CrUX, Web Vitals JS).

## When to Dive Deeper

This skill covers 80% of frontend performance work. For advanced scenarios:

- **Server-side rendering (SSR)**: Next.js, Remix for faster initial paint.
- **Streaming SSR**: Progressive hydration with React 18 Server Components.
- **Web Workers**: Offload heavy computation (parsing, data processing) to background thread.
- **HTTP/2 & HTTP/3**: Multiplexing, server push, reduced latency.
- **Advanced caching**: IndexedDB for large client-side datasets, stale-while-revalidate patterns.

Performance is a continuous process, not a one-time task. Set up monitoring, performance budgets, and automated checks (Lighthouse CI) to catch regressions before they ship.