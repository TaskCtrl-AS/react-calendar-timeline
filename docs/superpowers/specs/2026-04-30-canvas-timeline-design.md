# @taskctrl/canvas-timeline — Design Spec

## Overview

A new high-performance canvas-based timeline library to replace the DOM-based `@taskctrl/react-calendar-timeline` fork. Targets 1000+ groups and 5000+ items at 60fps.

**Approach:** Hybrid canvas — canvas for the content area (items, rows, grid), DOM for headers, sidebar, and interactive overlays (tooltips, context menus).

**Architecture:** Layered canvases — three stacked canvases for independent redraw cycles.

**Migration:** Separate package (`@taskctrl/canvas-timeline`), timelines migrated one at a time.

---

## 1. Component Architecture

```
<CanvasTimeline>
  +-- <TimelineHeaders>        (DOM)
  |   +-- <SidebarHeader>      (DOM)
  |   +-- <DateHeader>         (DOM)
  |   +-- <CustomHeader>       (DOM — CustomMonthHeader works here)
  +-- <Sidebar>                (DOM — virtualized group list)
  |   +-- renders only visible groups
  +-- <CanvasContent>          (Canvas layers container)
  |   +-- <canvas> grid        Layer 0: row backgrounds, grid lines, day/row styles
  |   +-- <canvas> items       Layer 1: items, group items, dependency arrows
  |   +-- <canvas> overlay     Layer 2: drag ghost, snap guide, cursor line
  +-- <OverlayPortal>          (DOM — tooltips, context menus positioned absolutely)
```

Headers and sidebar stay as DOM — they're small, fixed-position, and benefit from CSS/accessibility. The canvas layers handle all rendering that scales with data count.

Sidebar uses a lightweight virtual list — only renders DOM nodes for groups visible in the viewport plus a small overscan buffer. Scroll position syncs with the canvas content.

---

## 2. Canvas Layer System

### Layer 0 — Grid
**Redraws on:** zoom, resize, `dayStyle`/`rowStyle` prop change.

Renders:
- Row background bands (alternating or per-group via `rowStyle` callback)
- Vertical grid lines (day/week/month boundaries)
- Horizontal row separators
- Day column fills (via `dayStyle` callback — weekends, holidays, freeze periods)

**Scroll behavior:** CSS `transform: translate()` — no redraw on pan. Redraws only when zoom changes or new grid regions enter the buffer zone.

### Layer 1 — Items
**Redraws on:** zoom, data change, scroll into new buffer region.

Renders:
- All visible items via user-provided `itemRenderer` draw function
- Group items (trains, control area groups) via `groupRenderer` draw function
- Dependency arrows (bezier curves between connected items)
- Selection highlight on selected items

**Spatial index:** Items indexed by time range using an interval tree. On scroll/zoom, query for visible items — O(log n + k). No full-array filtering.

**Buffer zone:** Pre-renders items 1.5x viewport width on each side. Scroll within this zone is a CSS translate. Exiting the buffer triggers a partial redraw.

### Layer 2 — Overlay
**Redraws on:** every animation frame during interaction only.

Renders:
- Drag ghost (semi-transparent item copy following cursor)
- Snap guide line (vertical dashed line at target day)
- Cursor crosshair line
- Rubber-band selection rectangle (future)

Cleared when idle. Only shows cursor line when `showCursorLine: true`.

### DPI Handling
All canvases use `window.devicePixelRatio` for backing store size. CSS dimensions set to logical pixels. Crisp rendering on Retina displays.

### Coordinate System
A shared `ViewState` manages mapping between:
- **Data coordinates:** time (ms) on X, group index on Y
- **Canvas coordinates:** pixel position

All layers and sidebar read from the same `ViewState` for synchronized scroll/zoom.

---

## 3. Visual Design Language

Matches the genius-feature-construction2 app's existing design system.

### Items
- **3px border radius** (matching existing `rounded-xs`)
- **No drop shadow** — flat style matching current items
- **Train colors as fill** — `aero: #65aae4`, `violet: rgba(166,160,254,1)`, `cyan: rgba(38,198,218,1)`, etc.
- **Status gradient:** half-status items (`half_red`, `half_green`, `half_yellow`) rendered as 50/50 horizontal gradient
- **Clean zone indicator:** 3px left-edge bar in red (`#EF5350`) or yellow (`#FBBF24`)
- **Selected state:** 2px ring in `#a3a3a3` with 1px offset
- **Filtered-out items:** 40% opacity
- **Text:** 12px medium weight, `#374151` (gray-700), truncated with ellipsis
- **Hover:** brightness reduction
- **Icons:** Checkout checkmark and danger icons as vector paths on canvas

### Group Items
- **Control area groups:** Gray bar (`#d1d5db`) with green progress fill (`#31c48d`), fraction text, red/yellow blocking badges as filled circles
- **Train items:** Solid train color fill with title text, 2px black border when selected

### Grid & Rows
- Row backgrounds: `#FFFFFF` alternating with `#F7F7F7`
- Grid lines: `#E5E5E5` at 0.5px
- Row separators: `#E5E5E5`

### Markers
- **Today line:** 6px wide `#FD7171`
- **Milestone markers:** 4-6px `#3B82F6` lines
- **Cursor line:** 1px `#269bf7`

### Headers & Sidebar
- Background: `#F9FAFB`
- Borders: `#E5E7EB`
- Text: `#6c737f`
- Active/selected accent: `#269bf7` (blue-root)

### Day Column Styling
- Weekends: `rgba(0,0,0,0.03)`
- Holidays: `rgba(239,83,80,0.08)`
- Freeze periods: `rgba(59,130,246,0.06)`

### Theme Object
All colors exposed as configuration:

```typescript
interface TimelineTheme {
  primary: string              // '#269bf7'
  trainColors: Record<string, string>
  status: { red: string; yellow: string; green: string }
  grid: { line: string; rowAlt: string; weekend: string }
  item: { radius: number; text: string; selectedRing: string }
  marker: { today: string; milestone: string; cursor: string }
  sidebar: { bg: string; border: string; text: string }
  header: { bg: string; border: string; text: string }
}
```

---

## 4. API Surface

### Core Props

```typescript
interface CanvasTimelineProps {
  // Data
  groups: Group[]
  items: Item[]

  // Time bounds
  defaultTimeStart: number
  defaultTimeEnd: number
  visibleTimeStart?: number
  visibleTimeEnd?: number

  // Layout
  sidebarWidth: number
  lineHeight: number
  itemHeightRatio: number
  stackItems: boolean
  buffer?: number                // default: 3

  // Constraints
  canMove: boolean
  canResize: boolean
  canChangeGroup: boolean
  dragSnap: number
  minZoom: number
  maxZoom: number

  // Styling
  theme?: Partial<TimelineTheme>
  dayStyle?: (date: Date) => DayStyle | null
  rowStyle?: (group: Group) => RowStyle | null
  showCursorLine?: boolean

  // Rendering
  itemRenderer: CanvasItemRenderer
  groupRenderer?: CanvasGroupItemRenderer
  sidebarGroupRenderer: (group: Group) => React.ReactNode

  // Dependencies (opt-in)
  dependencies?: Dependency[]

  // Interactions
  onItemClick?: (itemId: number, e: PointerEvent) => void
  onItemDoubleClick?: (itemId: number, e: PointerEvent) => void
  onItemContextMenu?: (itemId: number, e: PointerEvent) => void
  onItemMove?: (itemId: number, newStartTime: number) => void
  onItemHover?: (itemId: number | null, e: PointerEvent) => void
  onCanvasDoubleClick?: (groupId: number, time: number) => void
  onCanvasContextMenu?: (groupId: number, time: number, e: PointerEvent) => void
  onTimeChange?: (start: number, end: number) => void
  onZoom?: (start: number, end: number) => void

  // Selection
  selected?: number[]

  // Children (markers, headers)
  children?: React.ReactNode
}
```

### Item Renderer — Draw Functions

```typescript
type CanvasItemRenderer = (
  ctx: CanvasRenderingContext2D,
  item: Item,
  bounds: ItemBounds,
  state: ItemState,
  helpers: DrawHelpers,
) => void

interface ItemBounds {
  x: number
  y: number
  width: number
  height: number
}

interface ItemState {
  selected: boolean
  hovered: boolean
  dragging: boolean
  filtered: boolean
}

interface DrawHelpers {
  roundRect(x: number, y: number, w: number, h: number, radius?: number): void
  fillText(text: string, x: number, y: number, maxWidth?: number): void
  gradient(x: number, w: number, color1: string, color2: string): CanvasGradient
  leftBar(color: string, width?: number): void
  icon(type: 'check' | 'danger-red' | 'danger-yellow', x: number, y: number, size?: number): void
  badge(text: string, x: number, y: number, bgColor: string): void
}
```

### Day & Row Styling

```typescript
interface DayStyle {
  backgroundColor?: string
  borderColor?: string
  opacity?: number
}

interface RowStyle {
  backgroundColor?: string
  borderBottomColor?: string
}
```

### Dependencies

```typescript
interface Dependency {
  fromItemId: number
  toItemId: number
  type?: 'finish-to-start' | 'start-to-start' | 'finish-to-finish'
  color?: string
}
```

### Data Shapes

```typescript
interface Group {
  id: number | string
  title: string
  type?: string
  hierarchy?: string
  root?: boolean
  [key: string]: any
}

interface Item {
  id: number
  group: number | string
  start_time: number
  end_time: number
  type?: string
  [key: string]: any
}
```

### Markers & Headers (Children API)

```typescript
<CanvasTimeline ...>
  <TodayMarker color="#FD7171" width={6} />
  <CustomMarker date={timestamp} color="#3B82F6" width={6} />

  <TimelineHeaders>
    <SidebarHeader>{(props) => <YourSidebarHeader {...props} />}</SidebarHeader>
    <DateHeader unit="month" />
    <DateHeader unit="day" />
    <CustomMonthHeader ... />
  </TimelineHeaders>
</CanvasTimeline>
```

---

## 5. Interaction System

### Hit Testing
1. Pointer event → convert pixel `(x, y)` to `(time, groupIndex)` via ViewState
2. Query interval tree for items at that time in that group — O(log n + k)
3. For stacked items, check y against each item's stacked bounds
4. Return topmost item

### Drag-to-Move
1. `mousedown` on item → enter drag mode, capture item snapshot
2. `mousemove` → overlay layer renders drag ghost (smooth, follows cursor) + snap guide line
3. Grid and items layers untouched during drag
4. `mouseup` → calculate snapped time, call `onItemMove(itemId, newStartTime)`, clear overlay
5. Items layer redraws after app updates state

- `canMove: false` disables drag entirely
- `canChangeGroup: false` locks ghost to same row

### Scroll & Pan
- **Mouse wheel** (vertical): scroll groups, sidebar syncs
- **Shift + wheel** (horizontal): pan time axis
- **Pinch-to-zoom** (trackpad): zoom around cursor position
- **Click-drag on empty canvas**: pan time axis horizontally
- All throttled via `requestAnimationFrame`

### Zoom
- Changes `visibleTimeStart`/`visibleTimeEnd`
- Clamped between `minZoom` and `maxZoom`
- Anchored to cursor position
- Triggers: grid redraw, items redraw, headers React re-render
- Fires `onZoom` and `onTimeChange`

### Selection
- Click on item updates `selected[]` via app state
- Library renders visual state (ring) on items layer

---

## 6. Performance Architecture

### Spatial Index (Interval Tree)
- Built once on `items` prop change — O(n log n)
- Queried on scroll/zoom for visible items — O(log n + k)
- Queried on pointer events for hit testing — O(log n + k)
- Rebuilt only when items array reference changes

### Stacking Algorithm
- Computed once when items or groups change, result cached
- Sweep-line algorithm: sort by start time, maintain active set, assign stack levels — O(n log n)
- Output: `Map<itemId, { stackLevel, y }>` — lookup only during render

### Render Budgets

| Layer | When | Target |
|-------|------|--------|
| Grid | Zoom, resize | < 2ms |
| Items | Scroll into new buffer, data change | < 8ms |
| Overlay | Every frame during interaction | < 1ms |
| **Total** | | **< 16ms (60fps)** |

### Buffer Strategy
- Items layer pre-renders 1.5x viewport width on each side
- Scroll within buffer = CSS `transform: translateX()` — zero canvas redraw
- Exiting buffer → redraw items layer with new buffer center
- Most scroll frames: **0ms canvas work**

### Redraw Trigger Matrix

| Event | Grid | Items | Overlay | Sidebar | Headers |
|-------|------|-------|---------|---------|---------|
| H-scroll (in buffer) | translate | translate | clear | -- | translate |
| H-scroll (exit buffer) | translate | **redraw** | clear | -- | translate |
| V-scroll | translate | translate | clear | sync | -- |
| Zoom | **redraw** | **redraw** | clear | -- | re-render |
| Item data change | -- | **redraw** | -- | -- | -- |
| Group data change | **redraw** | **redraw** | -- | re-render | -- |
| Drag in progress | -- | -- | **redraw** | -- | -- |
| Drag end | -- | **redraw** | clear | -- | -- |
| Hover | -- | -- | cursor line | -- | -- |
| Window resize | **redraw** | **redraw** | clear | re-measure | re-measure |

### Memory
- 3 canvases at ~3x viewport width x height x 4 bytes x devicePixelRatio^2
- 1920x800 viewport at 2x DPI: ~106 MB (acceptable for desktop)
- Interval tree + layout cache for 5000 items: < 1 MB

---

## 7. Dependency Arrows

Opt-in feature for DeliveryTimeline and GanttTimeline. Not used by ConstructionTimeline.

### Rendering
- Drawn on items layer after all items (arrows appear on top)
- Bezier S-curves from source right-edge midpoint to target left-edge midpoint
- Arrowhead: small filled triangle at target end
- Default color: `#94A3B8`, 1.5px line width
- Only arrows where both items are in visible buffer are drawn

### Hover Interaction
- Hovering an item highlights connected arrows to `#269bf7`, 2px width
- Drawn on overlay layer — items layer untouched

### Performance
- Lookup map: `itemId -> [connected dependencies]`
- Only visible items' dependencies iterated on redraw

### Not Included
- No interactive arrow creation
- No arrow routing/collision avoidance
- No arrow labels

---

## 8. Package Structure

```
@taskctrl/canvas-timeline/
  src/
    index.ts                     # Public API exports
    CanvasTimeline.tsx           # Root component

    core/
      ViewState.ts              # Time<->pixel mapping, zoom/pan state
      IntervalTree.ts           # Spatial index
      LayoutEngine.ts           # Stacking algorithm, position cache
      HitTest.ts                # Pointer event -> item resolution

    canvas/
      GridLayer.ts              # Layer 0: rows, grid, day/row styles
      ItemsLayer.ts             # Layer 1: items, group items, arrows
      OverlayLayer.ts           # Layer 2: drag ghost, snap guide, cursor
      DrawHelpers.ts            # Utility drawing functions
      CanvasManager.ts          # DPI, layer lifecycle, redraw scheduling

    interaction/
      ScrollHandler.ts          # Wheel, pan, trackpad
      ZoomHandler.ts            # Pinch-to-zoom, wheel zoom
      DragHandler.ts            # Smooth drag with snap

    dom/
      Sidebar.tsx               # Virtualized group list
      TimelineHeaders.tsx       # Header container
      DateHeader.tsx            # Date interval header
      SidebarHeader.tsx         # Sidebar header section
      CustomHeader.tsx          # Generic custom header
      TodayMarker.tsx           # Marker config
      CustomMarker.tsx          # Marker config

    types.ts                    # All public TypeScript interfaces

  package.json
  tsconfig.json
  vite.config.ts
  vitest.config.ts
```

### Build

```json
{
  "name": "@taskctrl/canvas-timeline",
  "main": "./dist/canvas-timeline.cjs.js",
  "module": "./dist/canvas-timeline.es.js",
  "types": "./dist/index.d.ts",
  "peerDependencies": {
    "react": "^18 || ^19",
    "react-dom": "^18 || ^19",
    "dayjs": "^1.11.10"
  }
}
```

Zero runtime dependencies beyond peer deps. No lodash, no interact.js, no CSS files.

### Testing
- **Unit tests** (vitest): ViewState math, IntervalTree queries, LayoutEngine stacking, HitTest
- **Integration tests** (vitest + jsdom): Component rendering, event handling, prop-change redraw triggers
- Canvas tests use mock `CanvasRenderingContext2D` to assert draw calls

---

## 9. Migration Example — ConstructionTimeline

### Before (current DOM-based)

```tsx
// Item renderer returns React components
const itemRenderer = ({ item, getItemProps, itemContext }) => (
  <>
    {item.type === 'construction_locomotive' && (
      <ConstructionCanvasItem item={item} getItemProps={getItemProps} ... />
    )}
  </>
)

<Timeline
  groups={getGroups}
  items={items}
  itemRenderer={itemRenderer}
  groupRenderer={groupRenderer}
  ...
/>
```

### After (canvas-based)

```tsx
// Item renderer draws on canvas
const itemRenderer: CanvasItemRenderer = (ctx, item, bounds, state, h) => {
  const bg = resolveColor(item.status_color, item.hand_over_color, item.train_color)
  ctx.fillStyle = bg
  h.roundRect(bounds.x, bounds.y, bounds.width, bounds.height, 3)

  if (item.clean_status === 'red_clean_zone') h.leftBar('#EF5350')
  if (item.has_checked_out) h.icon('check', bounds.x + 8, bounds.y + 4, 16)

  ctx.fillStyle = '#374151'
  h.fillText(item.name, bounds.x + 28, bounds.y + bounds.height / 2 + 4, bounds.width - 36)

  if (state.selected) {
    ctx.strokeStyle = '#a3a3a3'
    ctx.lineWidth = 2
    h.roundRect(bounds.x - 1, bounds.y - 1, bounds.width + 2, bounds.height + 2, 3)
    ctx.stroke()
  }
}

// Hover/context menus become portal-based
<CanvasTimeline
  groups={getGroups}
  items={items}
  itemRenderer={itemRenderer}
  onItemHover={(id, e) => setHovered(id ? { id, pos: { left: e.clientX, top: e.clientY } } : null)}
  onItemContextMenu={(id, e) => setContextMenu({ id, pos: { left: e.clientX, top: e.clientY } })}
  ...
/>
{hovered && <ConstructionHoverInfo id={hovered.id} pos={hovered.pos} />}
{contextMenu && <WagonContextMenu id={contextMenu.id} pos={contextMenu.pos} />}
```

### What Changes
- Item renderers: React components -> canvas draw functions
- Hover info / context menus: inline in item -> DOM portals triggered by callbacks
- Sidebar group rendering: stays as React (via `sidebarGroupRenderer`)
- Headers: stay as React (CustomMonthHeader works as-is)
- Data shapes, callbacks, state management: unchanged

### What Stays the Same
- `groups`, `items` arrays — same shape
- `onItemMove`, `onItemClick`, `onCanvasDoubleClick` — same callbacks
- `onTimeChange`, `onZoom` — same
- `selected[]` — same
- Sidebar resize, persisted state, printing support — app-level, unchanged
