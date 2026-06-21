# FlatList & List Optimization

Slow or choppy long lists are the #1 RN performance complaint. `FlatList` is built on
`VirtualizedList`, which only renders items near the viewport. The job of tuning is to
keep the **fill rate** high (no blank gaps) while keeping **memory** and **JS work per
batch** low. These goals pull against each other — every prop below is a trade-off.

## Key terms

- **Viewport:** the visible area (rendered to pixels).
- **Window:** the larger area around the viewport where items are kept mounted.
- **Blank areas:** empty space shown when virtualization can't render items fast enough
  for how fast the user scrolls.
- **Responsiveness:** how quickly touches/presses get handled — hurt by long JS render
  batches.

## First, the highest-impact item-level fixes

These matter more than prop tuning in most apps:

### 1. `getItemLayout` — the single best win for fixed-size rows

If every row has the same height (or width, for horizontal lists), provide
`getItemLayout`. It removes async layout measurement entirely, so the list can jump to
any offset instantly and scrolling stays smooth.

```tsx
const ITEM_HEIGHT = 72;

const getItemLayout = (_: unknown, index: number) => ({
  length: ITEM_HEIGHT,
  offset: ITEM_HEIGHT * index,
  index,
});

<FlatList data={data} renderItem={renderItem} getItemLayout={getItemLayout} />
```

If rows are dynamically sized and you genuinely need the performance, the docs'
blunt advice is to push back on the design — ask whether fixed heights are possible.
Otherwise consider FlashList, which estimates sizes.

### 2. Memoize the item component

Wrap items in `React.memo()` so a row only re-renders when *its* props change, not when
the parent list re-renders.

```tsx
import { memo } from 'react';
import { View, Text } from 'react-native';

const MyListItem = memo(
  ({ title }: { title: string }) => (
    <View>
      <Text>{title}</Text>
    </View>
  ),
  (prev, next) => prev.title === next.title, // optional custom comparator
);

export default MyListItem;
```

### 3. Hoist `renderItem` + `useCallback` — never inline it

An inline arrow `renderItem={({item}) => <Row .../>}` is a brand-new function every
render, defeating memoization downstream. Define it once.

```tsx
const renderItem = useCallback(
  ({ item }) => <MyListItem title={item.title} />,
  [],
);

return <FlatList data={items} renderItem={renderItem} keyExtractor={keyExtractor} />;
```

### 4. Stable `keyExtractor`

Provide a stable, unique key. It's used for caching and as the React key to track
re-ordering. Avoid using the array index as the key for lists that mutate/reorder.

```tsx
const keyExtractor = useCallback((item) => item.id, []);
```

### 5. Keep items light and simple

- **Basic components:** less nesting and logic per row = faster render. If a row
  component is reused all over the app, make a dedicated stripped-down variant for big
  lists.
- **Light images:** every list image is a `new Image()`; the sooner it loads the sooner
  the JS thread frees up. Use thumbnails/cropped versions sized for the row, and a
  caching image lib (e.g. a maintained fork of `react-native-fast-image`) rather than a
  full-resolution source.

## Then tune the virtualization props

Defaults are reasonable; change these only when a trace or visible blanking justifies
it. Each has a real cost.

| Prop | Default | Increase it → | Decrease it → |
| --- | --- | --- | --- |
| `windowSize` (units of viewport height; 21 = 10 above + 10 below + 1) | 21 | fewer blanks while scrolling, **more memory** | less memory, **more blank areas** |
| `maxToRenderPerBatch` (items per scroll batch) | 10 | fewer blanks (higher fill rate), **longer JS blocks → worse responsiveness** | snappier touch handling, more blanks |
| `updateCellsBatchingPeriod` (ms between batches) | 50 | fewer, larger batches | more frequent, smaller batches |
| `initialNumToRender` (first paint count) | 10 | covers the screen on first render | faster initial render but risk of initial blank if too low to fill viewport |
| `removeClippedSubviews` (detach off-screen views) | `true` on Android, else `false` | reduces main-thread work | — |

Notes & caveats:

- `removeClippedSubviews` **detaches** off-screen native views (less main-thread
  draw/traversal work) but does **not** deallocate them, so memory savings are minimal.
  It can cause missing content / blanking bugs, mainly on iOS and especially with
  transforms or absolute positioning. Test carefully before enabling on iOS.
- Combine `maxToRenderPerBatch` + `updateCellsBatchingPeriod` deliberately: e.g. *more*
  items per batch *less* frequently, or *fewer* items *more* frequently — pick based on
  whether you're fighting blanks (raise fill) or fighting unresponsiveness (smaller,
  rarer batches).
- Set `initialNumToRender` to roughly the number of rows that fill one screen on your
  smallest target device.

## When to reach for a third-party list

For very large or heterogeneous lists, the official docs point to community libraries
optimized beyond `FlatList`:

- **FlashList** (Shopify) — recycles views, estimates item sizes, usually a drop-in
  with big gains for heavy lists. Provide `estimatedItemSize`.
- **Legend List** — another performance-focused alternative.

These help most when rows are complex or sizes vary; for simple fixed-height rows, a
well-configured `FlatList` with `getItemLayout` is often enough.

## Review checklist for any list

- [ ] `getItemLayout` provided (or fixed heights / FlashList `estimatedItemSize`)?
- [ ] Item component wrapped in `memo()`?
- [ ] `renderItem` hoisted + `useCallback`, not inline?
- [ ] Stable `keyExtractor` (not array index for mutable data)?
- [ ] Rows lightweight (minimal nesting, thumbnail images, cached image lib)?
- [ ] Virtualization props tuned only where a trace/visible blanking justified it, with
      the memory/responsiveness trade-off acknowledged?
- [ ] Not rendering a huge data set via `.map()` inside a `ScrollView`?
