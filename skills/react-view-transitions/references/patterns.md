# Patterns and Guidelines

## Searchable Grid with `useDeferredValue`

`useDeferredValue` makes filter updates a transition, activating `<ViewTransition>`:

```tsx
'use client';

import { useDeferredValue, useState, ViewTransition, Suspense } from 'react';

export default function SearchableGrid({ itemsPromise }) {
  const [search, setSearch] = useState('');
  const deferredSearch = useDeferredValue(search);

  return (
    <>
      <input value={search} onChange={(e) => setSearch(e.currentTarget.value)} />
      <ViewTransition>
        <Suspense fallback={<GridSkeleton />}>
          <ItemGrid itemsPromise={itemsPromise} search={deferredSearch} />
        </Suspense>
      </ViewTransition>
    </>
  );
}
```

Per-item `<ViewTransition name={...}>` inside a deferred list triggers cross-fades on every keystroke. Fix with `default="none"`:

```tsx
{filteredItems.map(item => (
  <ViewTransition key={item.id} name={`item-${item.id}`} share="morph" default="none">
    <ItemCard item={item} />
  </ViewTransition>
))}
```

## Card Expand/Collapse with `startTransition`

Toggle between grid and detail view with shared element morph:

```tsx
'use client';

import { useState, useRef, startTransition, ViewTransition } from 'react';

export default function ItemGrid({ items }) {
  const [expandedId, setExpandedId] = useState(null);
  const scrollRef = useRef(0);

  return expandedId ? (
    <ViewTransition enter="slide-in" name={`item-${expandedId}`}>
      <ItemDetail
        item={items.find(i => i.id === expandedId)}
        onClose={() => {
          startTransition(() => {
            setExpandedId(null);
            setTimeout(() => window.scrollTo({ behavior: 'smooth', top: scrollRef.current }), 100);
          });
        }}
      />
    </ViewTransition>
  ) : (
    <div className="grid grid-cols-3 gap-4">
      {items.map(item => (
        <ViewTransition key={item.id} name={`item-${item.id}`}>
          <ItemCard
            item={item}
            onSelect={() => {
              scrollRef.current = window.scrollY;
              startTransition(() => setExpandedId(item.id));
            }}
          />
        </ViewTransition>
      ))}
    </div>
  );
}
```

## Type-Safe Transition Helpers

Use `as const` arrays and derived types to prevent ID clashes:

```tsx
const transitionTypes = ['default', 'transition-to-detail', 'transition-to-list'] as const;
const animationTypes = ['auto', 'none', 'animate-slide-from-left', 'animate-slide-from-right'] as const;

type TransitionType = (typeof transitionTypes)[number];
type AnimationType = (typeof animationTypes)[number];
type TransitionMap = { default: AnimationType } & Partial<Record<Exclude<TransitionType, 'default'>, AnimationType>>;

export function HorizontalTransition({ children, enter, exit }: {
  children: React.ReactNode;
  enter: TransitionMap;
  exit: TransitionMap;
}) {
  return <ViewTransition enter={enter} exit={exit}>{children}</ViewTransition>;
}
```

## Cross-Fade Without Remount

Omit `key` to trigger an update (cross-fade) instead of exit + enter. Avoids Suspense remount/refetch:

```jsx
<ViewTransition>
  <TabPanel tab={activeTab} />
</ViewTransition>
```

Use `key` when content identity changes (state resets). Omit for cross-fades (tabs, panels, carousel).

## Isolate Elements from Parent Animations

The only way to pull an element out of the browser's animated `root` snapshot is to give it **its own `view-transition-name`**. There is no "exclude from root" primitive.

> **`view-transition-name: none` does NOT isolate anything.** `none` is the CSS *default* — setting it explicitly is a no-op, and the element stays part of the `root` capture. This is a common bug (e.g. a popover with `style={{ viewTransitionName: 'none' }}` still flickers). Isolation always requires a **real, unique name** plus CSS that neutralizes its animation.

### Persistent Layout Elements

Persistent elements (headers, navbars, sidebars) get captured in the page's transition snapshot. Fix with a real `viewTransitionName`:

```jsx
<nav style={{ viewTransitionName: "persistent-nav" }}>{/* ... */}</nav>
```

Then add the persistent element isolation CSS from `css-recipes.md`. For `backdrop-blur`/`backdrop-filter`, use the backdrop-blur workaround from `css-recipes.md`.

**`<ViewTransition default="none">` vs. a hand-named group.** Wrapping the element in `<ViewTransition default="none">` also lifts it out of `root` and disables its animation, with *no* CSS — prefer it when animation-off is all you need. But React assigns that boundary an **auto-generated** name you can't target from CSS, so you **cannot** add `z-index` layering or the backdrop-blur `::view-transition-old(...) { display: none }` workaround to it. When you need those (e.g. sticky chrome that must stay above transitioning content, or a blurred sidebar), keep the **hand-named** `viewTransitionName` + explicit CSS. This is inherent, not boilerplate you can delete.

### Floating Elements (popovers, menus, tooltips)

A floating element left **open** while a background transition runs is captured in the `root` snapshot and **flickers as the transition settles**. Give it a real `viewTransitionName` and isolate it:

```jsx
<SelectPopover style={{ viewTransitionName: 'select-popover' }}>{options}</SelectPopover>
```

```css
::view-transition-group(select-popover) { animation: none; z-index: 100; }
::view-transition-old(select-popover),
::view-transition-new(select-popover) { animation: none; }
```

A static name is safe as long as only one instance is mounted at a time (e.g. the menu uses `unmountOnHide`). If genuinely rendered in the browser **top layer** (native `popover`/`<dialog>`), the settle re-composite is a browser limitation React can't reach — but most portaled popovers are ordinary divs and the real-name fix resolves the flicker.

## Shared Controls / Text Between Skeleton and Content (fixing the reveal flicker)

**Why an element that appears in *both* the fallback and the resolved content flickers:** they're two different DOM nodes across the Suspense swap. Without a shared name the browser snapshots them separately and **cross-fades** (old fades out while new fades in) — with text or a solid bar of different size/content, that reads as a ghost/flicker. It is **not** a "pin it" problem — pinning (isolation) freezes the element; here you *want* continuity.

**Fix: morph the fallback shape into the content shape.** Give the matching element in fallback and content the **same `view-transition-name`**, and — critically for text/solid shapes — animate only the **group** (position/size) while disabling the old/new cross-fade:

```jsx
// Fallback
<input disabled placeholder="Search..." style={{ viewTransitionName: 'search-input' }} />
// Content
<input placeholder="Search..." style={{ viewTransitionName: 'search-input' }} />
```

```css
/* the group interpolates position + size = a smooth morph */
::view-transition-group(search-input) { animation-duration: 220ms; }
/* no cross-fade of the two snapshots = no ghost/flicker */
::view-transition-old(search-input),
::view-transition-new(search-input) { animation: none; }
```

So yes — you *can* "morph the fallback too": the fallback element and the content element sharing one name is what makes the fallback shape animate into the content shape instead of flickering. Blanket-avoiding it ("never put the same element in both") works but forfeits the nicest reveal; the shared-name + group-only morph is the better default.

Don't put manual `viewTransitionName` on the root DOM node inside `<ViewTransition>` — React's auto-generated name overrides it.

## Sliding Indicator (tabs / segmented control)

A single active-indicator (underline, pill) that **slides** to the selected tab is the same shared-element mechanism: render the indicator **only under the active tab**, give every tab's indicator the **same `name`**, and it morphs from the old position to the new one on change. Animate the group (the slide); disable old/new so the solid bar doesn't cross-fade.

```tsx
'use client';
import { useOptimistic, useTransition, ViewTransition } from 'react';

export function Tabs({ tabs, active, indicatorName = 'tab-indicator' }) {
  const [optimisticActive, setOptimisticActive] = useOptimistic(active);
  const [, startTransition] = useTransition();
  return (
    <nav>
      {tabs.map(t => {
        const isCommitted = active === t.value;              // where the bar actually is
        return (
          <Link key={t.value} href={t.href} scroll={false}
            aria-current={optimisticActive === t.value ? 'page' : undefined}
            onNavigate={() => startTransition(() => setOptimisticActive(t.value))}>
            <span>{t.label}</span>
            {isCommitted && (
              <ViewTransition name={indicatorName} share="tab-underline">
                <span className="active-underline" aria-hidden />
              </ViewTransition>
            )}
          </Link>
        );
      })}
    </nav>
  );
}
```

```css
::view-transition-group(.tab-underline) {
  animation-duration: 220ms;
  animation-timing-function: cubic-bezier(0.5, 0, 0.2, 1);
}
::view-transition-old(.tab-underline),
::view-transition-new(.tab-underline) {
  animation: none;   /* don't fade the two bars — just let the group slide */
  height: 100%;      /* both snapshots fill so it reads as one solid moving bar */
}
```

Key details: the indicator is keyed to the **committed** `active` (so the bar sits where navigation actually landed), while `useOptimistic` drives the *label* state instantly; give each mount point a distinct `indicatorName` so two tab strips on one page don't fight over one name.

## Reusable Animated Collapse

```jsx
function AnimatedCollapse({ open, children }) {
  if (!open) return null;
  return (
    <ViewTransition enter="expand-in" exit="collapse-out">
      {children}
    </ViewTransition>
  );
}

// Usage: toggle with startTransition
<button onClick={() => startTransition(() => setOpen(o => !o))}>Toggle</button>
<AnimatedCollapse open={open}><SectionContent /></AnimatedCollapse>
```

## Preserve State with Activity

```jsx
<Activity mode={isVisible ? 'visible' : 'hidden'}>
  <ViewTransition enter="slide-in" exit="slide-out">
    <Sidebar />
  </ViewTransition>
</Activity>
```

## Exclude Elements with `useOptimistic`

`useOptimistic` values update before the transition snapshot, excluding them from animation. Use for controls (labels); use committed state for animated content:

```tsx
const [sort, setSort] = useState('newest');
const [optimisticSort, setOptimisticSort] = useOptimistic(sort);

function cycleSort() {
  const nextSort = getNextSort(optimisticSort);
  startTransition(() => {
    setOptimisticSort(nextSort);  // before snapshot — no animation
    setSort(nextSort);            // between snapshots — animates
  });
}

<button>Sort: {LABELS[optimisticSort]}</button>
{items.sort(comparators[sort]).map(item => (
  <ViewTransition key={item.id}><ItemCard item={item} /></ViewTransition>
))}
```

---

## View Transition Events

Imperative control via `onEnter`, `onExit`, `onUpdate`, `onShare`. Always return a cleanup function. `onShare` takes precedence over `onEnter`/`onExit`.

```jsx
<ViewTransition
  onEnter={(instance, types) => {
    const anim = instance.new.animate(
      [{ transform: 'scale(0.8)', opacity: 0 }, { transform: 'scale(1)', opacity: 1 }],
      { duration: 300, easing: 'ease-out' }
    );
    return () => anim.cancel();
  }}
>
  <Component />
</ViewTransition>
```

The `instance` object: `instance.old`, `instance.new`, `instance.group`, `instance.imagePair`, `instance.name`.

The `types` array (second argument) lets you vary animation based on transition type.

---

## Animation Timing

| Interaction | Duration |
|------------|----------|
| Direct toggle (expand/collapse) | 100–200ms |
| Route transition (slide) | 150–250ms |
| Suspense reveal (skeleton → content) | 200–400ms |
| Shared element morph | 300–500ms |

---

## Troubleshooting

**VT not activating:** Ensure `<ViewTransition>` comes before any DOM node. Ensure state update is inside `startTransition`.

**"Two ViewTransition components with the same name":** Names must be globally unique. Use IDs: `name={`hero-${item.id}`}`.

**Scrolling hangs / feels stuck while a transition animates (esp. a Suspense reveal):** the browser's `::view-transition` overlay is `position: fixed` over the viewport, and the old/new snapshots don't scroll with the page — so scrolling appears frozen until the animation settles. This is a **browser/CSS View Transitions limitation**, not something React can hide (skipping the transition on scroll snaps abruptly to the end). Mitigate by keeping reveal durations short (200–400ms). For genuinely scroll-driven UI, use **gesture transitions** (`useSwipeTransition` / `startGestureTransition`) — those are scrubbed by scroll position so scrolling *drives* the transition instead of fighting it.

**Open popover/menu flickers when a background transition settles:** the floating element is captured in the `root` snapshot. Give it a real `view-transition-name` + isolation CSS — see "Floating Elements" above. (`viewTransitionName: 'none'` does not fix it — that's the CSS default.)

**`router.back()` and browser back/forward skip animation:** Use `router.push()` with an explicit URL instead. See SKILL.md "router.back() and Browser Back Button."

**`flushSync` skips animations:** Use `startTransition` instead.

**Only updates animate (no enter/exit):** Without `<Suspense>`, React treats swaps as updates. Conditionally render the VT itself, or wrap in `<Suspense>`.

**Layout VT prevents page VTs from animating:** Nested VTs never fire enter/exit inside a parent VT. If your layout has a VT wrapping `{children}`, page-level enter/exit will silently not work. Remove the layout VT.

**List reorder not animating with `useOptimistic`:** Optimistic values resolve before snapshot. Use committed state for list order.

**TS error "Property 'default' is missing":** Type-keyed objects require a `default` key.

**Hash fragments cause scroll jumps:** Navigate without hash; scroll programmatically after navigation.

**Backdrop-blur flickers:** Use the backdrop-blur workaround from `css-recipes.md`.

**`border-radius` lost during transitions:** Apply `border-radius` directly to the captured element.

**Skeleton controls slide away:** Give matching controls the same `viewTransitionName`.

**Batching:** Multiple updates during animation are batched. A→B→C→D becomes B→D.
