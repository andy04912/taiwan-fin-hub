# Taiwan Fin Hub — Global App Shell Redesign Plan

## Design Read

Reading this as: a trust-first personal finance product for daily desktop and mobile use, with a calm financial-console language, leaning toward a full-screen application shell, dense but readable information hierarchy, one restrained cyan/steel accent, and purposeful motion rather than decorative effects.

## Taste Dials

- `DESIGN_VARIANCE: 6` — enough asymmetry and hierarchy to avoid template-like dashboards while preserving financial trust.
- `MOTION_INTENSITY: 8` — visible page, navigation, list and state transitions, capped for responsiveness and reduced-motion support.
- `VISUAL_DENSITY: 7` — desktop console density with touch-safe mobile layouts.

## Product Architecture

- Desktop: fixed primary sidebar + fixed top command bar + one internal scroll viewport.
- Tablet: compact top navigation with optional slide-over menu.
- Mobile/PWA: sticky top bar + hamburger drawer + four-item bottom navigation.
- Detail pages: predictable back navigation inside the shell, never a separate document-style page.
- Settings: preserve the existing secondary settings shell inside the global shell.
- Motion: page enter/exit, active-nav indicator, drawer transitions, staggered content reveal, tactile press states, loading shimmer, number/status emphasis.
- Accessibility: keyboard navigation, visible focus, 44px touch targets, no color-only states, `prefers-reduced-motion` fallback.

## UI UX Pro Max Recommendation

## Design System: Taiwan Fin Hub

### Design Dials
- **Variance:** 6/10 — Balanced / Modern
- **Motion:** 8/10 — Complex
- **Density:** 7/10 — Standard

### Pattern
- **Name:** Real-Time / Operations Landing
- **Conversion Focus:** For ops/security/iot products. Demo or sandbox link. Trust signals.
- **CTA Placement:** Primary CTA in nav + After metrics
- **Color Strategy:** Dark or neutral. Status colors (green/amber/red). Data-dense but scannable.
- **Sections:** 1. Hero (product + live preview or status), 2. Key metrics/indicators, 3. How it works, 4. CTA (Start trial / Contact)

### Style
- **Name:** Data-Dense Dashboard
- **Mode Support:** Light ✓ Full | Dark ✓ Full
- **Keywords:** Multiple charts/widgets, data tables, KPI cards, minimal padding, grid layout, space-efficient, maximum data visibility
- **Best For:** Business intelligence dashboards, financial analytics, enterprise reporting, operational dashboards, data warehousing
- **Performance:** ⚡ Excellent | **Accessibility:** ✓ WCAG AA

### Colors
| Role | Hex | CSS Variable |
|------|-----|--------------|
| Primary | `#1E40AF` | `--color-primary` |
| On Primary | `#FFFFFF` | `--color-on-primary` |
| Secondary | `#3B82F6` | `--color-secondary` |
| Accent/CTA | `#059669` | `--color-accent` |
| Background | `#0F172A` | `--color-background` |
| Foreground | `#FFFFFF` | `--color-foreground` |
| Muted | `#101A34` | `--color-muted` |
| Border | `rgba(255,255,255,0.08)` | `--color-border` |
| Destructive | `#DC2626` | `--color-destructive` |
| Ring | `#1E40AF` | `--color-ring` |

*Notes: Trust blue + profit green on dark*

### Typography
- **Heading:** Fira Code
- **Body:** Fira Sans
- **Mood:** dashboard, data, analytics, code, technical, precise
- **Best For:** Dashboards, analytics, data visualization, admin panels
- **Google Fonts:** https://fonts.googleapis.com/css2?family=Fira+Code:wght@400;500;600;700&family=Fira+Sans:wght@300;400;500;600;700&display=swap
- **CSS Import:**
```css
@import url('https://fonts.googleapis.com/css2?family=Fira+Code:wght@400;500;600;700&family=Fira+Sans:wght@300;400;500;600;700&display=swap');
```

### Key Effects
Hover tooltips, chart zoom on click, row highlighting on hover, smooth filter animations, data loading spinners

### Motion
**Page Transition** (Complex) — Trigger: route change | Duration: 500-800ms | Easing: `expo.inOut`
```js
const state = Flip.getState('.hero-image'); navigate(); Flip.from(state, { duration: 0.6, ease: 'expo.inOut', absolute: true, zIndex: 100 });
```
*Framework notes: Requires the GSAP Flip plugin; the 'from' and 'to' route must render the same element with a shared data-flip-id*
- ✅ Verify the shared element exists in both DOM states before calling Flip.from to avoid a silent no-op
- ❌ Don't use shared-element transitions across more than one element pair per navigation; compounding Flips are hard to time correctly

### Avoid (Anti-patterns)
- Ornate design
- No filtering

### Pre-Delivery Checklist
- [ ] No emojis as icons (use SVG: Heroicons/Lucide)
- [ ] cursor-pointer on all clickable elements
- [ ] Hover states with smooth transitions (150-300ms)
- [ ] Light mode: text contrast 4.5:1 minimum
- [ ] Focus states visible for keyboard nav
- [ ] prefers-reduced-motion respected
- [ ] Responsive: 375px, 768px, 1024px, 1440px



## Svelte App Shell Guidance

## UI Pro Max Stack Guidelines
**Stack:** svelte | **Query:** responsive app shell sidebar drawer sticky navigation transitions accessibility
**Source:** stacks/svelte.csv | **Found:** 3 results

### Result 1
- **Category:** SvelteKit
- **Guideline:** Use $app/stores for app state
- **Description:** $page $navigating $updated
- **Do:** $page for current page data
- **Don't:** Manual URL parsing
- **Code Good:** import { page } from '$app/stores'
- **Code Bad:** window.location.pathname
- **Severity:** Medium
- **Docs URL:** https://kit.svelte.dev/docs/modules#$app-stores

### Result 2
- **Category:** Transitions
- **Guideline:** Use built-in transitions
- **Description:** Svelte transition directives
- **Do:** transition:fade for simple effects
- **Don't:** Manual CSS transitions
- **Code Good:** <div transition:fade>
- **Code Bad:** <div class:fade={visible}>
- **Severity:** Low
- **Docs URL:** https://svelte.dev/docs/element-directives#transition-fn

### Result 3
- **Category:** Transitions
- **Guideline:** Add local modifier
- **Description:** Prevent ancestor trigger
- **Do:** transition:fade|local
- **Don't:** Global transitions for lists
- **Code Good:** <div transition:slide|local>
- **Code Bad:** <div transition:slide>
- **Severity:** Medium
- **Docs URL:** 

