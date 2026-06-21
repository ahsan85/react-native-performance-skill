# React Render Optimization (JS-thread)

When profiling shows the **JS thread** overrunning frame boundaries (see
`profiling.md`, signature #1), the cause is usually too much rendering work: components
re-rendering when they shouldn't, or rendering something expensive. The JS thread is
where React reconciliation, your logic, and touch handling all compete — every wasted
re-render steals from the 16ms budget.

## Diagnose first

- Use the **Perf Monitor** (Dev Menu) to confirm it's JS FPS dropping, not UI FPS.
- Use the **React DevTools Profiler** to see which components render, how often, and
  why. Look for components re-rendering on unrelated state changes — that's a
  fan-out problem.
- A classic trace tell (from the official profiling doc): an event/handler firing many
  times per frame, or one state set on a high component re-rendering a large subtree.

## The core fixes

### 1. Memoize components that re-render needlessly

`React.memo()` skips a re-render when props are shallow-equal. Apply to leaf and
mid-tree components that get the same props but re-render because a parent did.

```tsx
const Row = memo(function Row({ title, onPress }: Props) {
  return <Pressable onPress={onPress}><Text>{title}</Text></Pressable>;
});
```

Memoization only helps if the props are actually stable — which requires the next two.

### 2. Stabilize callbacks with `useCallback`

A new function identity each render breaks `memo` on every child receiving it. Wrap
handlers passed as props:

```tsx
const onPress = useCallback(() => select(id), [id]);
```

### 3. Stabilize derived values / objects with `useMemo`

Inline object/array literals (`style={{...}}`, `data={[...]}`) are new references each
render and defeat memoization downstream. Memoize expensive computations and the
objects you pass as props:

```tsx
const sorted = useMemo(() => items.slice().sort(compare), [items]);
const containerStyle = useMemo(() => ({ padding: 16, gap }), [gap]);
```

Don't `useMemo`/`useCallback` *everything* — each has a small cost and clutters code.
Apply where a child is memoized, a value is expensive to compute, or a reference must
stay stable across renders.

### 4. Tame Context fan-out

Every consumer of a Context re-renders when the Context value changes. Two pitfalls:

- Passing a **fresh object** as the provider value each render (`value={{ user, setUser }}`)
  re-renders all consumers every time — memoize it with `useMemo`.
- Putting **too much** in one Context so unrelated updates re-render everyone. Split
  contexts by update frequency (e.g. separate rarely-changing config from
  frequently-changing state), or colocate state lower in the tree.

### 5. Keep state local and lift only when necessary

State set high in the tree re-renders everything below it. Keep state as close to where
it's used as possible so updates have a small blast radius. A single high-up `setState`
that re-renders a complex subtree can blow the entire frame budget.

### 6. Avoid expensive work in render

Render should be cheap and pure. Move heavy computation into `useMemo`, effects, or
worker/async paths. Don't sort/filter/format large arrays inline in JSX on every render.

## Lists are a special case

For list-row re-render problems, the list-specific patterns (memoized items, hoisted
`renderItem` + `useCallback`, stable `keyExtractor`) live in
`references/flatlist-optimization.md`. The principles are the same; the list amplifies
them across many rows.

## When re-renders aren't the problem

If memoization doesn't move the needle, the JS thread may be busy with non-render work:
synchronous parsing/computation, over-firing event handlers (debounce/throttle them), or
chatty bridge/native-module calls. Re-profile to see what's actually running on the JS
thread before adding more `memo`.

## Checklist

- [ ] Confirmed via Perf Monitor / React DevTools Profiler that JS-thread re-renders are
      the bottleneck?
- [ ] Components that get stable props wrapped in `memo()`?
- [ ] Callbacks passed as props wrapped in `useCallback`?
- [ ] Expensive values / passed objects wrapped in `useMemo` (no inline literals into
      memoized children)?
- [ ] Context provider values memoized; large contexts split by update frequency?
- [ ] State kept as local as possible (small re-render blast radius)?
- [ ] No heavy synchronous computation inside render?
