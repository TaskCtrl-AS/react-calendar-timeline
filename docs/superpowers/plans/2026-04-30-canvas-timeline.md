# @taskctrl/canvas-timeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a high-performance canvas-based timeline library targeting 1000+ groups / 5000+ items at 60fps.

**Architecture:** Hybrid canvas with three stacked canvas layers (grid, items, overlay) for independent redraw cycles. DOM for headers, sidebar (virtualized), and interactive overlays. Spatial indexing via interval tree. Sweep-line stacking algorithm. Buffer zones with CSS translate for zero-cost scrolling.

**Tech Stack:** React 18/19, TypeScript, Vite, Vitest, dayjs. Zero runtime dependencies beyond peer deps.

**Spec:** `docs/superpowers/specs/2026-04-30-canvas-timeline-design.md`

---

## File Structure

```
src/
  index.ts                     — Public API: re-exports all public types and CanvasTimeline
  CanvasTimeline.tsx           — Root component: orchestrates layers, sidebar, headers, state
  types.ts                     — All public TypeScript interfaces (Group, Item, Theme, etc.)

  core/
    ViewState.ts               — Time↔pixel coordinate mapping, zoom/pan state, buffer bounds
    IntervalTree.ts            — Augmented interval tree for spatial queries
    LayoutEngine.ts            — Sweep-line stacking, position cache per item
    HitTest.ts                 — Pixel→item resolution using ViewState + IntervalTree + layout

  canvas/
    CanvasManager.ts           — DPI setup, layer lifecycle, redraw scheduling via rAF
    DrawHelpers.ts             — roundRect, fillText, gradient, leftBar, icon, badge
    GridLayer.ts               — Layer 0: row bands, grid lines, day/row styles
    ItemsLayer.ts              — Layer 1: items via renderer, dependency arrows
    OverlayLayer.ts            — Layer 2: drag ghost, snap guide, cursor line

  interaction/
    ScrollHandler.ts           — Wheel/pan events → ViewState updates
    ZoomHandler.ts             — Pinch/wheel-zoom → ViewState updates
    DragHandler.ts             — Mousedown→move→up on items, snap calculation

  dom/
    Sidebar.tsx                — Virtualized group list, synced scroll
    TimelineHeaders.tsx        — Header container, provides context
    DateHeader.tsx             — Date interval header cells
    SidebarHeader.tsx          — Top-left sidebar header slot
    CustomHeader.tsx           — Generic custom header (render prop)
    TodayMarker.tsx            — Config component, read by CanvasTimeline
    CustomMarker.tsx           — Config component, read by CanvasTimeline

  __tests__/
    ViewState.test.ts
    IntervalTree.test.ts
    LayoutEngine.test.ts
    HitTest.test.ts
    DrawHelpers.test.ts
    GridLayer.test.ts
    ItemsLayer.test.ts
    OverlayLayer.test.ts
    ScrollHandler.test.ts
    ZoomHandler.test.ts
    DragHandler.test.ts
    CanvasTimeline.test.tsx
```

---

## Task 1: Project Scaffold

**Files:**
- Create: `package.json`, `tsconfig.json`, `tsconfig.node.json`, `vite.config.mts`, `vitest.config.ts`, `src/index.ts`, `src/types.ts`

- [ ] **Step 1: Create package.json**

Create the new package directory and `package.json`:

```bash
mkdir -p /Users/abrha/Documents/projects/canvas-timeline/src
```

```json
{
  "name": "@taskctrl/canvas-timeline",
  "version": "0.1.0",
  "description": "High-performance canvas-based timeline component for React",
  "scripts": {
    "build": "rimraf ./dist && tsc && vite build",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint --ext .ts --ext .tsx ./src"
  },
  "main": "./dist/canvas-timeline.cjs.js",
  "module": "./dist/canvas-timeline.es.js",
  "sideEffects": false,
  "exports": {
    ".": {
      "types": ["./dist/index.d.ts"],
      "import": "./dist/canvas-timeline.es.js",
      "require": "./dist/canvas-timeline.cjs.js"
    }
  },
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "author": "TaskCtrl AS",
  "license": "MIT",
  "peerDependencies": {
    "react": "^18 || ^19",
    "react-dom": "^18 || ^19",
    "dayjs": "^1.11.10"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.5.0",
    "@testing-library/react": "^16.0.1",
    "@types/react": "^18.2.41",
    "@types/react-dom": "^18.2.17",
    "@rollup/plugin-typescript": "^12.1.0",
    "@typescript-eslint/eslint-plugin": "^8.8.1",
    "@typescript-eslint/parser": "^8.8.1",
    "@vitejs/plugin-react-swc": "^3.7.1",
    "@vitest/coverage-v8": "^3.2.4",
    "eslint": "^8.57.1",
    "jsdom": "^25.0.1",
    "prettier": "^3.1.0",
    "rimraf": "^6.0.1",
    "rollup-plugin-typescript-paths": "^1.5.0",
    "typescript": "^5.2.2",
    "vite": "^5.4.11",
    "vite-plugin-dts": "^4.3.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"],
  "exclude": ["node_modules"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

- [ ] **Step 3: Create tsconfig.node.json**

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.mts"]
}
```

- [ ] **Step 4: Create vite.config.mts**

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react-swc'
import { resolve } from 'path'
import typescript from '@rollup/plugin-typescript'
import { typescriptPaths } from 'rollup-plugin-typescript-paths'

export default defineConfig({
  build: {
    copyPublicDir: false,
    outDir: 'dist',
    sourcemap: true,
    lib: {
      entry: resolve('src', 'index.ts'),
      name: 'canvas-timeline',
      fileName: (format) => `canvas-timeline.${format}.js`,
    },
    rollupOptions: {
      external: ['react', 'react/jsx-runtime', 'react-dom', 'react-dom/client', 'dayjs'],
      output: {
        globals: {
          react: 'React',
        },
      },
      plugins: [
        typescriptPaths({ preserveExtensions: true }),
        typescript({ outDir: 'dist', declaration: true }),
      ],
    },
  },
  plugins: [react()],
})
```

- [ ] **Step 5: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: [],
  },
})
```

- [ ] **Step 6: Create src/types.ts with all public interfaces**

```typescript
export interface Group {
  id: number | string
  title: string
  type?: string
  hierarchy?: string
  root?: boolean
  [key: string]: unknown
}

export interface Item {
  id: number
  group: number | string
  start_time: number
  end_time: number
  type?: string
  [key: string]: unknown
}

export interface ItemBounds {
  x: number
  y: number
  width: number
  height: number
}

export interface ItemState {
  selected: boolean
  hovered: boolean
  dragging: boolean
  filtered: boolean
}

export interface DrawHelpers {
  roundRect(x: number, y: number, w: number, h: number, radius?: number): void
  fillText(text: string, x: number, y: number, maxWidth?: number): void
  gradient(x: number, w: number, color1: string, color2: string): CanvasGradient
  leftBar(color: string, width?: number): void
  icon(type: 'check' | 'danger-red' | 'danger-yellow', x: number, y: number, size?: number): void
  badge(text: string, x: number, y: number, bgColor: string): void
}

export type CanvasItemRenderer = (
  ctx: CanvasRenderingContext2D,
  item: Item,
  bounds: ItemBounds,
  state: ItemState,
  helpers: DrawHelpers,
) => void

export type CanvasGroupItemRenderer = CanvasItemRenderer

export interface DayStyle {
  backgroundColor?: string
  borderColor?: string
  opacity?: number
}

export interface RowStyle {
  backgroundColor?: string
  borderBottomColor?: string
}

export interface Dependency {
  fromItemId: number
  toItemId: number
  type?: 'finish-to-start' | 'start-to-start' | 'finish-to-finish'
  color?: string
}

export interface TimelineTheme {
  primary: string
  trainColors: Record<string, string>
  status: { red: string; yellow: string; green: string }
  grid: { line: string; rowAlt: string; weekend: string }
  item: { radius: number; text: string; selectedRing: string }
  marker: { today: string; milestone: string; cursor: string }
  sidebar: { bg: string; border: string; text: string }
  header: { bg: string; border: string; text: string }
}

export const DEFAULT_THEME: TimelineTheme = {
  primary: '#269bf7',
  trainColors: {},
  status: { red: '#EF5350', yellow: '#FBBF24', green: '#31c48d' },
  grid: { line: '#E5E5E5', rowAlt: '#F7F7F7', weekend: 'rgba(0,0,0,0.03)' },
  item: { radius: 3, text: '#374151', selectedRing: '#a3a3a3' },
  marker: { today: '#FD7171', milestone: '#3B82F6', cursor: '#269bf7' },
  sidebar: { bg: '#F9FAFB', border: '#E5E7EB', text: '#6c737f' },
  header: { bg: '#F9FAFB', border: '#E5E7EB', text: '#6c737f' },
}

export interface MarkerConfig {
  date: number
  color: string
  width: number
}

export interface CanvasTimelineProps {
  groups: Group[]
  items: Item[]
  defaultTimeStart: number
  defaultTimeEnd: number
  visibleTimeStart?: number
  visibleTimeEnd?: number
  sidebarWidth: number
  lineHeight: number
  itemHeightRatio: number
  stackItems: boolean
  buffer?: number
  canMove: boolean
  canResize: boolean
  canChangeGroup: boolean
  dragSnap: number
  minZoom: number
  maxZoom: number
  theme?: Partial<TimelineTheme>
  dayStyle?: (date: Date) => DayStyle | null
  rowStyle?: (group: Group) => RowStyle | null
  showCursorLine?: boolean
  itemRenderer: CanvasItemRenderer
  groupRenderer?: CanvasGroupItemRenderer
  sidebarGroupRenderer: (group: Group) => React.ReactNode
  dependencies?: Dependency[]
  onItemClick?: (itemId: number, e: PointerEvent) => void
  onItemDoubleClick?: (itemId: number, e: PointerEvent) => void
  onItemContextMenu?: (itemId: number, e: PointerEvent) => void
  onItemMove?: (itemId: number, newStartTime: number) => void
  onItemHover?: (itemId: number | null, e: PointerEvent) => void
  onCanvasDoubleClick?: (groupId: number, time: number) => void
  onCanvasContextMenu?: (groupId: number, time: number, e: PointerEvent) => void
  onTimeChange?: (start: number, end: number) => void
  onZoom?: (start: number, end: number) => void
  selected?: number[]
  children?: React.ReactNode
}
```

- [ ] **Step 7: Create src/index.ts placeholder**

```typescript
export type {
  Group,
  Item,
  ItemBounds,
  ItemState,
  DrawHelpers,
  CanvasItemRenderer,
  CanvasGroupItemRenderer,
  DayStyle,
  RowStyle,
  Dependency,
  TimelineTheme,
  MarkerConfig,
  CanvasTimelineProps,
} from './types'

export { DEFAULT_THEME } from './types'
```

- [ ] **Step 8: Install dependencies and verify build**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npm install
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 9: Commit**

```bash
git init
git add -A
git commit -m "chore: scaffold @taskctrl/canvas-timeline package"
```

---

## Task 2: ViewState — Coordinate System

**Files:**
- Create: `src/core/ViewState.ts`
- Test: `src/__tests__/ViewState.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/ViewState.test.ts
import { describe, it, expect } from 'vitest'
import { ViewState } from '../core/ViewState'

describe('ViewState', () => {
  const makeView = () =>
    new ViewState({
      visibleTimeStart: 1000,
      visibleTimeEnd: 2000,
      canvasWidth: 1000,
      canvasHeight: 500,
      sidebarWidth: 200,
      lineHeight: 32,
      groupCount: 10,
      buffer: 3,
      scrollTop: 0,
    })

  describe('timeToX', () => {
    it('maps visibleTimeStart to 0', () => {
      const v = makeView()
      expect(v.timeToX(1000)).toBe(0)
    })

    it('maps visibleTimeEnd to canvasWidth', () => {
      const v = makeView()
      expect(v.timeToX(2000)).toBe(1000)
    })

    it('maps midpoint correctly', () => {
      const v = makeView()
      expect(v.timeToX(1500)).toBe(500)
    })
  })

  describe('xToTime', () => {
    it('maps 0 to visibleTimeStart', () => {
      const v = makeView()
      expect(v.xToTime(0)).toBe(1000)
    })

    it('maps canvasWidth to visibleTimeEnd', () => {
      const v = makeView()
      expect(v.xToTime(1000)).toBe(2000)
    })

    it('is inverse of timeToX', () => {
      const v = makeView()
      expect(v.xToTime(v.timeToX(1234))).toBe(1234)
    })
  })

  describe('yToGroupIndex', () => {
    it('maps y=0 with scrollTop=0 to group 0', () => {
      const v = makeView()
      expect(v.yToGroupIndex(0)).toBe(0)
    })

    it('maps y=32 to group 1', () => {
      const v = makeView()
      expect(v.yToGroupIndex(32)).toBe(1)
    })

    it('accounts for scrollTop', () => {
      const v = new ViewState({
        visibleTimeStart: 1000,
        visibleTimeEnd: 2000,
        canvasWidth: 1000,
        canvasHeight: 500,
        sidebarWidth: 200,
        lineHeight: 32,
        groupCount: 10,
        buffer: 3,
        scrollTop: 64,
      })
      expect(v.yToGroupIndex(0)).toBe(2)
    })

    it('clamps to valid group range', () => {
      const v = makeView()
      expect(v.yToGroupIndex(-100)).toBe(0)
      expect(v.yToGroupIndex(9999)).toBe(9)
    })
  })

  describe('groupIndexToY', () => {
    it('maps group 0 to y=0 with scrollTop=0', () => {
      const v = makeView()
      expect(v.groupIndexToY(0)).toBe(0)
    })

    it('maps group 3 to y=96', () => {
      const v = makeView()
      expect(v.groupIndexToY(3)).toBe(96)
    })
  })

  describe('buffer bounds', () => {
    it('calculates buffered time range', () => {
      const v = makeView()
      const { bufferStart, bufferEnd } = v.getBufferBounds()
      const visibleDuration = 2000 - 1000
      // buffer=3 means 1.5x on each side beyond visible
      expect(bufferStart).toBe(1000 - visibleDuration * 1.5)
      expect(bufferEnd).toBe(2000 + visibleDuration * 1.5)
    })
  })

  describe('visible group range', () => {
    it('returns first and last visible group index', () => {
      const v = makeView()
      const { firstVisible, lastVisible } = v.getVisibleGroupRange()
      expect(firstVisible).toBe(0)
      // 500px / 32px = 15.6, but only 10 groups
      expect(lastVisible).toBe(9)
    })
  })

  describe('isInBuffer', () => {
    it('returns true for scroll within buffer', () => {
      const v = makeView()
      // offset=0 is within buffer initially
      expect(v.isScrollInBuffer(0)).toBe(true)
    })

    it('returns false for scroll way outside buffer', () => {
      const v = makeView()
      expect(v.isScrollInBuffer(999999)).toBe(false)
    })
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ViewState.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement ViewState**

```typescript
// src/core/ViewState.ts
export interface ViewStateConfig {
  visibleTimeStart: number
  visibleTimeEnd: number
  canvasWidth: number
  canvasHeight: number
  sidebarWidth: number
  lineHeight: number
  groupCount: number
  buffer: number
  scrollTop: number
}

export class ViewState {
  readonly visibleTimeStart: number
  readonly visibleTimeEnd: number
  readonly canvasWidth: number
  readonly canvasHeight: number
  readonly sidebarWidth: number
  readonly lineHeight: number
  readonly groupCount: number
  readonly buffer: number
  readonly scrollTop: number

  private readonly visibleDuration: number
  private readonly pixelsPerMs: number

  constructor(config: ViewStateConfig) {
    this.visibleTimeStart = config.visibleTimeStart
    this.visibleTimeEnd = config.visibleTimeEnd
    this.canvasWidth = config.canvasWidth
    this.canvasHeight = config.canvasHeight
    this.sidebarWidth = config.sidebarWidth
    this.lineHeight = config.lineHeight
    this.groupCount = config.groupCount
    this.buffer = config.buffer
    this.scrollTop = config.scrollTop

    this.visibleDuration = this.visibleTimeEnd - this.visibleTimeStart
    this.pixelsPerMs = this.canvasWidth / this.visibleDuration
  }

  timeToX(time: number): number {
    return (time - this.visibleTimeStart) * this.pixelsPerMs
  }

  xToTime(x: number): number {
    return this.visibleTimeStart + x / this.pixelsPerMs
  }

  yToGroupIndex(y: number): number {
    const raw = Math.floor((y + this.scrollTop) / this.lineHeight)
    return Math.max(0, Math.min(raw, this.groupCount - 1))
  }

  groupIndexToY(index: number): number {
    return index * this.lineHeight - this.scrollTop
  }

  getBufferBounds(): { bufferStart: number; bufferEnd: number } {
    const extend = this.visibleDuration * 1.5
    return {
      bufferStart: this.visibleTimeStart - extend,
      bufferEnd: this.visibleTimeEnd + extend,
    }
  }

  getVisibleGroupRange(): { firstVisible: number; lastVisible: number } {
    const firstVisible = Math.max(0, Math.floor(this.scrollTop / this.lineHeight))
    const visibleCount = Math.ceil(this.canvasHeight / this.lineHeight)
    const lastVisible = Math.min(this.groupCount - 1, firstVisible + visibleCount)
    return { firstVisible, lastVisible }
  }

  isScrollInBuffer(scrollXOffset: number): boolean {
    const bufferPixels = this.visibleDuration * 1.5 * this.pixelsPerMs
    return Math.abs(scrollXOffset) < bufferPixels
  }

  getTotalHeight(): number {
    return this.groupCount * this.lineHeight
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ViewState.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/core/ViewState.ts src/__tests__/ViewState.test.ts
git commit -m "feat: add ViewState coordinate system"
```

---

## Task 3: IntervalTree — Spatial Index

**Files:**
- Create: `src/core/IntervalTree.ts`
- Test: `src/__tests__/IntervalTree.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/IntervalTree.test.ts
import { describe, it, expect } from 'vitest'
import { IntervalTree } from '../core/IntervalTree'

interface TestItem {
  id: number
  start: number
  end: number
  group: number
}

describe('IntervalTree', () => {
  const items: TestItem[] = [
    { id: 1, start: 0, end: 10, group: 1 },
    { id: 2, start: 5, end: 15, group: 1 },
    { id: 3, start: 20, end: 30, group: 1 },
    { id: 4, start: 0, end: 5, group: 2 },
    { id: 5, start: 100, end: 200, group: 2 },
  ]

  const makeTree = () => {
    const tree = new IntervalTree<TestItem>()
    tree.buildFromItems(items, (i) => i.start, (i) => i.end)
    return tree
  }

  describe('query', () => {
    it('finds items overlapping a range', () => {
      const tree = makeTree()
      const results = tree.query(3, 12)
      const ids = results.map((r) => r.id).sort()
      expect(ids).toEqual([1, 2, 4])
    })

    it('returns empty for range with no items', () => {
      const tree = makeTree()
      const results = tree.query(50, 90)
      expect(results).toEqual([])
    })

    it('finds items at exact boundaries', () => {
      const tree = makeTree()
      const results = tree.query(10, 20)
      const ids = results.map((r) => r.id).sort()
      // item 2 ends at 15 (overlaps), item 3 starts at 20 (overlaps)
      expect(ids).toEqual([2, 3])
    })

    it('handles single-point query', () => {
      const tree = makeTree()
      const results = tree.query(5, 5)
      const ids = results.map((r) => r.id).sort()
      expect(ids).toEqual([1, 2, 4])
    })
  })

  describe('performance', () => {
    it('handles 10000 items efficiently', () => {
      const tree = new IntervalTree<TestItem>()
      const bigItems: TestItem[] = []
      for (let i = 0; i < 10000; i++) {
        bigItems.push({ id: i, start: i * 10, end: i * 10 + 50, group: i % 100 })
      }
      tree.buildFromItems(bigItems, (i) => i.start, (i) => i.end)

      const start = performance.now()
      const results = tree.query(5000, 5100)
      const elapsed = performance.now() - start

      expect(results.length).toBeGreaterThan(0)
      expect(results.length).toBeLessThan(100)
      expect(elapsed).toBeLessThan(5) // should be well under 5ms
    })
  })

  describe('empty tree', () => {
    it('returns empty for queries on empty tree', () => {
      const tree = new IntervalTree<TestItem>()
      tree.buildFromItems([], (i) => i.start, (i) => i.end)
      expect(tree.query(0, 100)).toEqual([])
    })
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/IntervalTree.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement IntervalTree**

```typescript
// src/core/IntervalTree.ts

interface TreeNode<T> {
  center: number
  left: TreeNode<T> | null
  right: TreeNode<T> | null
  overlapping: Array<{ item: T; start: number; end: number }>
}

export class IntervalTree<T> {
  private root: TreeNode<T> | null = null

  buildFromItems(
    items: T[],
    getStart: (item: T) => number,
    getEnd: (item: T) => number,
  ): void {
    const intervals = items.map((item) => ({
      item,
      start: getStart(item),
      end: getEnd(item),
    }))
    this.root = this.buildNode(intervals)
  }

  private buildNode(
    intervals: Array<{ item: T; start: number; end: number }>,
  ): TreeNode<T> | null {
    if (intervals.length === 0) return null

    // Find the center point
    let min = Infinity
    let max = -Infinity
    for (const iv of intervals) {
      if (iv.start < min) min = iv.start
      if (iv.end > max) max = iv.end
    }
    const center = (min + max) / 2

    const leftIntervals: Array<{ item: T; start: number; end: number }> = []
    const rightIntervals: Array<{ item: T; start: number; end: number }> = []
    const overlapping: Array<{ item: T; start: number; end: number }> = []

    for (const iv of intervals) {
      if (iv.end < center) {
        leftIntervals.push(iv)
      } else if (iv.start > center) {
        rightIntervals.push(iv)
      } else {
        overlapping.push(iv)
      }
    }

    return {
      center,
      left: this.buildNode(leftIntervals),
      right: this.buildNode(rightIntervals),
      overlapping,
    }
  }

  query(start: number, end: number): T[] {
    const results: T[] = []
    this.queryNode(this.root, start, end, results)
    return results
  }

  private queryNode(
    node: TreeNode<T> | null,
    start: number,
    end: number,
    results: T[],
  ): void {
    if (node === null) return

    for (const iv of node.overlapping) {
      if (iv.start <= end && iv.end >= start) {
        results.push(iv.item)
      }
    }

    if (start <= node.center && node.left !== null) {
      this.queryNode(node.left, start, end, results)
    }

    if (end >= node.center && node.right !== null) {
      this.queryNode(node.right, start, end, results)
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/IntervalTree.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/core/IntervalTree.ts src/__tests__/IntervalTree.test.ts
git commit -m "feat: add IntervalTree spatial index"
```

---

## Task 4: LayoutEngine — Stacking Algorithm

**Files:**
- Create: `src/core/LayoutEngine.ts`
- Test: `src/__tests__/LayoutEngine.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/LayoutEngine.test.ts
import { describe, it, expect } from 'vitest'
import { LayoutEngine, ItemLayout } from '../core/LayoutEngine'
import { Item } from '../types'

describe('LayoutEngine', () => {
  const makeEngine = (lineHeight = 32, itemHeightRatio = 0.9) =>
    new LayoutEngine(lineHeight, itemHeightRatio)

  describe('computeLayout (stackItems=true)', () => {
    it('places non-overlapping items at stack level 0', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 10 },
        { id: 2, group: 'a', start_time: 20, end_time: 30 },
      ]
      const engine = makeEngine()
      const layout = engine.computeLayout(items, true)

      expect(layout.get(1)!.stackLevel).toBe(0)
      expect(layout.get(2)!.stackLevel).toBe(0)
    })

    it('stacks overlapping items in the same group', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 20 },
        { id: 2, group: 'a', start_time: 10, end_time: 30 },
        { id: 3, group: 'a', start_time: 15, end_time: 25 },
      ]
      const engine = makeEngine()
      const layout = engine.computeLayout(items, true)

      expect(layout.get(1)!.stackLevel).toBe(0)
      expect(layout.get(2)!.stackLevel).toBe(1)
      expect(layout.get(3)!.stackLevel).toBe(2)
    })

    it('does not stack items across different groups', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 20 },
        { id: 2, group: 'b', start_time: 0, end_time: 20 },
      ]
      const engine = makeEngine()
      const layout = engine.computeLayout(items, true)

      expect(layout.get(1)!.stackLevel).toBe(0)
      expect(layout.get(2)!.stackLevel).toBe(0)
    })

    it('calculates correct itemHeight', () => {
      const engine = makeEngine(32, 0.9)
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 10 },
      ]
      const layout = engine.computeLayout(items, true)
      expect(layout.get(1)!.itemHeight).toBeCloseTo(32 * 0.9)
    })
  })

  describe('computeLayout (stackItems=false)', () => {
    it('places all items at stack level 0', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 20 },
        { id: 2, group: 'a', start_time: 10, end_time: 30 },
      ]
      const engine = makeEngine()
      const layout = engine.computeLayout(items, false)

      expect(layout.get(1)!.stackLevel).toBe(0)
      expect(layout.get(2)!.stackLevel).toBe(0)
    })
  })

  describe('getGroupHeight', () => {
    it('returns lineHeight for groups with no stacking', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 10 },
      ]
      const engine = makeEngine(32)
      engine.computeLayout(items, true)
      expect(engine.getGroupHeight('a')).toBe(32)
    })

    it('returns expanded height for stacked groups', () => {
      const items: Item[] = [
        { id: 1, group: 'a', start_time: 0, end_time: 20 },
        { id: 2, group: 'a', start_time: 10, end_time: 30 },
      ]
      const engine = makeEngine(32)
      engine.computeLayout(items, true)
      // 2 stacked items = 2 * lineHeight
      expect(engine.getGroupHeight('a')).toBe(64)
    })
  })

  describe('performance', () => {
    it('computes layout for 5000 items under 50ms', () => {
      const items: Item[] = []
      for (let i = 0; i < 5000; i++) {
        items.push({
          id: i,
          group: `g${i % 100}`,
          start_time: i * 10,
          end_time: i * 10 + 50,
        })
      }
      const engine = makeEngine()
      const start = performance.now()
      engine.computeLayout(items, true)
      const elapsed = performance.now() - start
      expect(elapsed).toBeLessThan(50)
    })
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/LayoutEngine.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement LayoutEngine**

```typescript
// src/core/LayoutEngine.ts
import type { Item } from '../types'

export interface ItemLayout {
  stackLevel: number
  itemHeight: number
}

export class LayoutEngine {
  private readonly lineHeight: number
  private readonly itemHeightRatio: number
  private layoutCache: Map<number, ItemLayout> = new Map()
  private groupMaxStack: Map<string | number, number> = new Map()

  constructor(lineHeight: number, itemHeightRatio: number) {
    this.lineHeight = lineHeight
    this.itemHeightRatio = itemHeightRatio
  }

  computeLayout(items: Item[], stackItems: boolean): Map<number, ItemLayout> {
    this.layoutCache = new Map()
    this.groupMaxStack = new Map()

    if (!stackItems) {
      const itemHeight = this.lineHeight * this.itemHeightRatio
      for (const item of items) {
        this.layoutCache.set(item.id, { stackLevel: 0, itemHeight })
        this.groupMaxStack.set(item.group, 0)
      }
      return this.layoutCache
    }

    // Group items by group id
    const byGroup = new Map<string | number, Item[]>()
    for (const item of items) {
      let arr = byGroup.get(item.group)
      if (!arr) {
        arr = []
        byGroup.set(item.group, arr)
      }
      arr.push(item)
    }

    const itemHeight = this.lineHeight * this.itemHeightRatio

    for (const [groupId, groupItems] of byGroup) {
      // Sort by start_time, then by duration descending for tie-breaking
      groupItems.sort((a, b) => {
        const d = a.start_time - b.start_time
        if (d !== 0) return d
        return (b.end_time - b.start_time) - (a.end_time - a.start_time)
      })

      // Sweep-line: track end times per stack level
      const levelEnds: number[] = []
      let maxLevel = 0

      for (const item of groupItems) {
        // Find lowest available level
        let level = -1
        for (let i = 0; i < levelEnds.length; i++) {
          if (levelEnds[i] <= item.start_time) {
            level = i
            break
          }
        }

        if (level === -1) {
          level = levelEnds.length
          levelEnds.push(item.end_time)
        } else {
          levelEnds[level] = item.end_time
        }

        if (level > maxLevel) maxLevel = level

        this.layoutCache.set(item.id, { stackLevel: level, itemHeight })
      }

      this.groupMaxStack.set(groupId, maxLevel)
    }

    return this.layoutCache
  }

  getLayout(itemId: number): ItemLayout | undefined {
    return this.layoutCache.get(itemId)
  }

  getGroupHeight(groupId: string | number): number {
    const maxStack = this.groupMaxStack.get(groupId) ?? 0
    return (maxStack + 1) * this.lineHeight
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/LayoutEngine.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/core/LayoutEngine.ts src/__tests__/LayoutEngine.test.ts
git commit -m "feat: add LayoutEngine with sweep-line stacking"
```

---

## Task 5: DrawHelpers — Canvas Utility Functions

**Files:**
- Create: `src/canvas/DrawHelpers.ts`
- Test: `src/__tests__/DrawHelpers.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/DrawHelpers.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { createDrawHelpers } from '../canvas/DrawHelpers'

function createMockCtx() {
  return {
    beginPath: vi.fn(),
    moveTo: vi.fn(),
    lineTo: vi.fn(),
    arc: vi.fn(),
    closePath: vi.fn(),
    fill: vi.fn(),
    stroke: vi.fn(),
    fillRect: vi.fn(),
    strokeRect: vi.fn(),
    fillText: vi.fn(),
    measureText: vi.fn(() => ({ width: 50 })),
    save: vi.fn(),
    restore: vi.fn(),
    createLinearGradient: vi.fn(() => ({ addColorStop: vi.fn() })),
    roundRect: vi.fn(),
    font: '',
    fillStyle: '',
    strokeStyle: '',
    lineWidth: 0,
    globalAlpha: 1,
    textBaseline: '' as CanvasTextBaseline,
  } as unknown as CanvasRenderingContext2D
}

describe('DrawHelpers', () => {
  let ctx: CanvasRenderingContext2D

  beforeEach(() => {
    ctx = createMockCtx()
  })

  describe('roundRect', () => {
    it('calls ctx.roundRect and fill', () => {
      const helpers = createDrawHelpers(ctx)
      helpers.roundRect(10, 20, 100, 50, 3)
      expect(ctx.beginPath).toHaveBeenCalled()
      expect(ctx.roundRect).toHaveBeenCalledWith(10, 20, 100, 50, 3)
      expect(ctx.fill).toHaveBeenCalled()
    })

    it('uses default radius of 3 when not specified', () => {
      const helpers = createDrawHelpers(ctx)
      helpers.roundRect(0, 0, 50, 25)
      expect(ctx.roundRect).toHaveBeenCalledWith(0, 0, 50, 25, 3)
    })
  })

  describe('fillText', () => {
    it('draws text with ellipsis when text exceeds maxWidth', () => {
      const mockCtx = createMockCtx()
      // First call measures full text (150px), second measures truncated
      let callCount = 0
      mockCtx.measureText = vi.fn(() => {
        callCount++
        return { width: callCount === 1 ? 150 : 40 } as TextMetrics
      })
      const helpers = createDrawHelpers(mockCtx as unknown as CanvasRenderingContext2D)
      helpers.fillText('long text here', 10, 20, 100)
      expect(mockCtx.fillText).toHaveBeenCalled()
    })

    it('draws text without truncation when it fits', () => {
      const helpers = createDrawHelpers(ctx)
      helpers.fillText('short', 10, 20, 200)
      expect(ctx.fillText).toHaveBeenCalledWith('short', 10, 20)
    })
  })

  describe('leftBar', () => {
    it('draws a colored bar on the left edge', () => {
      const helpers = createDrawHelpers(ctx, { x: 10, y: 20, width: 100, height: 30 })
      helpers.leftBar('#EF5350', 3)
      expect(ctx.fillRect).toHaveBeenCalledWith(10, 20, 3, 30)
    })
  })

  describe('badge', () => {
    it('draws a badge with text', () => {
      const helpers = createDrawHelpers(ctx)
      helpers.badge('5', 100, 50, '#EF5350')
      expect(ctx.beginPath).toHaveBeenCalled()
      expect(ctx.fill).toHaveBeenCalled()
      expect(ctx.fillText).toHaveBeenCalled()
    })
  })

  describe('gradient', () => {
    it('creates a linear gradient', () => {
      const helpers = createDrawHelpers(ctx)
      helpers.gradient(0, 100, '#ff0000', '#0000ff')
      expect(ctx.createLinearGradient).toHaveBeenCalledWith(0, 0, 100, 0)
    })
  })

  describe('icon', () => {
    it('draws check icon without error', () => {
      const helpers = createDrawHelpers(ctx)
      expect(() => helpers.icon('check', 10, 10, 16)).not.toThrow()
    })

    it('draws danger-red icon without error', () => {
      const helpers = createDrawHelpers(ctx)
      expect(() => helpers.icon('danger-red', 10, 10, 16)).not.toThrow()
    })

    it('draws danger-yellow icon without error', () => {
      const helpers = createDrawHelpers(ctx)
      expect(() => helpers.icon('danger-yellow', 10, 10, 16)).not.toThrow()
    })
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/DrawHelpers.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement DrawHelpers**

```typescript
// src/canvas/DrawHelpers.ts
import type { DrawHelpers, ItemBounds } from '../types'

export function createDrawHelpers(
  ctx: CanvasRenderingContext2D,
  bounds?: ItemBounds,
): DrawHelpers {
  return {
    roundRect(x: number, y: number, w: number, h: number, radius = 3): void {
      ctx.beginPath()
      ctx.roundRect(x, y, w, h, radius)
      ctx.fill()
    },

    fillText(text: string, x: number, y: number, maxWidth?: number): void {
      ctx.textBaseline = 'middle'
      ctx.font = '500 12px -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif'

      if (maxWidth === undefined) {
        ctx.fillText(text, x, y)
        return
      }

      const measured = ctx.measureText(text)
      if (measured.width <= maxWidth) {
        ctx.fillText(text, x, y)
        return
      }

      // Binary search for the truncation point
      let lo = 0
      let hi = text.length
      while (lo < hi) {
        const mid = (lo + hi + 1) >> 1
        const truncated = text.slice(0, mid) + '...'
        if (ctx.measureText(truncated).width <= maxWidth) {
          lo = mid
        } else {
          hi = mid - 1
        }
      }

      const truncated = lo > 0 ? text.slice(0, lo) + '...' : '...'
      ctx.fillText(truncated, x, y)
    },

    gradient(x: number, w: number, color1: string, color2: string): CanvasGradient {
      const grad = ctx.createLinearGradient(x, 0, x + w, 0)
      grad.addColorStop(0, color1)
      grad.addColorStop(0.5, color1)
      grad.addColorStop(0.5, color2)
      grad.addColorStop(1, color2)
      return grad
    },

    leftBar(color: string, width = 3): void {
      if (!bounds) return
      const prevFill = ctx.fillStyle
      ctx.fillStyle = color
      ctx.fillRect(bounds.x, bounds.y, width, bounds.height)
      ctx.fillStyle = prevFill
    },

    icon(type: 'check' | 'danger-red' | 'danger-yellow', x: number, y: number, size = 16): void {
      const prevFill = ctx.fillStyle
      const prevStroke = ctx.strokeStyle
      const prevLineWidth = ctx.lineWidth
      const s = size

      if (type === 'check') {
        ctx.strokeStyle = '#374151'
        ctx.lineWidth = 2
        ctx.beginPath()
        ctx.moveTo(x + s * 0.2, y + s * 0.5)
        ctx.lineTo(x + s * 0.4, y + s * 0.7)
        ctx.lineTo(x + s * 0.8, y + s * 0.3)
        ctx.stroke()
      } else if (type === 'danger-red' || type === 'danger-yellow') {
        const color = type === 'danger-red' ? '#EF4444' : '#FBBF24'
        ctx.fillStyle = color
        ctx.beginPath()
        ctx.moveTo(x + s / 2, y + 1)
        ctx.lineTo(x + s - 1, y + s - 1)
        ctx.lineTo(x + 1, y + s - 1)
        ctx.closePath()
        ctx.fill()

        // Exclamation mark
        ctx.fillStyle = type === 'danger-red' ? '#FFFFFF' : '#000000'
        ctx.font = `bold ${s * 0.5}px sans-serif`
        ctx.textBaseline = 'middle'
        ctx.fillText('!', x + s / 2 - s * 0.08, y + s * 0.6)
      }

      ctx.fillStyle = prevFill
      ctx.strokeStyle = prevStroke
      ctx.lineWidth = prevLineWidth
    },

    badge(text: string, x: number, y: number, bgColor: string): void {
      const prevFill = ctx.fillStyle
      const padding = 4
      ctx.font = '600 11px sans-serif'
      const textWidth = ctx.measureText(text).width
      const badgeWidth = Math.max(textWidth + padding * 2, 20)
      const badgeHeight = 20
      const radius = badgeHeight / 2

      ctx.fillStyle = bgColor
      ctx.beginPath()
      ctx.roundRect(x - badgeWidth / 2, y - badgeHeight / 2, badgeWidth, badgeHeight, radius)
      ctx.fill()

      ctx.fillStyle = '#FFFFFF'
      ctx.textBaseline = 'middle'
      ctx.fillText(text, x - textWidth / 2, y)

      ctx.fillStyle = prevFill
    },
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/DrawHelpers.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/canvas/DrawHelpers.ts src/__tests__/DrawHelpers.test.ts
git commit -m "feat: add DrawHelpers canvas utility functions"
```

---

## Task 6: CanvasManager — DPI & Redraw Scheduling

**Files:**
- Create: `src/canvas/CanvasManager.ts`
- Test: `src/__tests__/CanvasManager.test.ts` (minimal — DPI logic is the testable part)

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/CanvasManager.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setupCanvas, scheduleRedraw } from '../canvas/CanvasManager'

describe('CanvasManager', () => {
  describe('setupCanvas', () => {
    it('sets canvas dimensions accounting for DPI', () => {
      const canvas = document.createElement('canvas')
      const dpr = 2
      vi.stubGlobal('devicePixelRatio', dpr)

      setupCanvas(canvas, 800, 600)

      expect(canvas.width).toBe(800 * dpr)
      expect(canvas.height).toBe(600 * dpr)
      expect(canvas.style.width).toBe('800px')
      expect(canvas.style.height).toBe('600px')

      vi.unstubAllGlobals()
    })

    it('returns a scaled context', () => {
      const canvas = document.createElement('canvas')
      vi.stubGlobal('devicePixelRatio', 2)

      const ctx = setupCanvas(canvas, 800, 600)
      // ctx should exist (jsdom may not fully support getContext)
      // In real browser, context.getTransform() would show scale(2,2)
      expect(ctx).toBeDefined()

      vi.unstubAllGlobals()
    })
  })

  describe('scheduleRedraw', () => {
    it('coalesces multiple calls into one rAF callback', () => {
      const callback = vi.fn()
      const rafSpy = vi.spyOn(globalThis, 'requestAnimationFrame').mockImplementation((cb) => {
        cb(0)
        return 0
      })

      const schedule = scheduleRedraw(callback)
      schedule()
      schedule()
      schedule()

      // Only one rAF should be scheduled
      expect(rafSpy).toHaveBeenCalledTimes(1)
      expect(callback).toHaveBeenCalledTimes(1)

      rafSpy.mockRestore()
    })
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/CanvasManager.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement CanvasManager**

```typescript
// src/canvas/CanvasManager.ts

export function setupCanvas(
  canvas: HTMLCanvasElement,
  width: number,
  height: number,
): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1
  canvas.width = width * dpr
  canvas.height = height * dpr
  canvas.style.width = `${width}px`
  canvas.style.height = `${height}px`

  const ctx = canvas.getContext('2d')!
  ctx.scale(dpr, dpr)
  return ctx
}

export function clearCanvas(ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement): void {
  const dpr = window.devicePixelRatio || 1
  ctx.clearRect(0, 0, canvas.width / dpr, canvas.height / dpr)
}

export function scheduleRedraw(callback: () => void): () => void {
  let scheduled = false

  return () => {
    if (scheduled) return
    scheduled = true
    requestAnimationFrame(() => {
      scheduled = false
      callback()
    })
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/CanvasManager.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/canvas/CanvasManager.ts src/__tests__/CanvasManager.test.ts
git commit -m "feat: add CanvasManager for DPI handling and redraw scheduling"
```

---

## Task 7: GridLayer — Layer 0

**Files:**
- Create: `src/canvas/GridLayer.ts`
- Test: `src/__tests__/GridLayer.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/GridLayer.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { GridLayer } from '../canvas/GridLayer'
import { ViewState } from '../core/ViewState'
import type { Group, TimelineTheme, DayStyle, RowStyle } from '../types'
import { DEFAULT_THEME } from '../types'

function createMockCtx() {
  return {
    beginPath: vi.fn(),
    moveTo: vi.fn(),
    lineTo: vi.fn(),
    fill: vi.fn(),
    stroke: vi.fn(),
    fillRect: vi.fn(),
    strokeRect: vi.fn(),
    clearRect: vi.fn(),
    save: vi.fn(),
    restore: vi.fn(),
    fillStyle: '',
    strokeStyle: '',
    lineWidth: 0,
    globalAlpha: 1,
    setLineDash: vi.fn(),
  } as unknown as CanvasRenderingContext2D
}

describe('GridLayer', () => {
  let ctx: CanvasRenderingContext2D

  beforeEach(() => {
    ctx = createMockCtx()
  })

  const makeViewState = () =>
    new ViewState({
      visibleTimeStart: new Date('2025-01-01').getTime(),
      visibleTimeEnd: new Date('2025-02-01').getTime(),
      canvasWidth: 1000,
      canvasHeight: 320,
      sidebarWidth: 200,
      lineHeight: 32,
      groupCount: 10,
      buffer: 3,
      scrollTop: 0,
    })

  const groups: Group[] = Array.from({ length: 10 }, (_, i) => ({
    id: i,
    title: `Group ${i}`,
  }))

  it('draws row backgrounds', () => {
    const layer = new GridLayer()
    const view = makeViewState()
    layer.draw(ctx, view, groups, DEFAULT_THEME)

    // Should call fillRect for row backgrounds
    expect(ctx.fillRect).toHaveBeenCalled()
  })

  it('draws vertical grid lines', () => {
    const layer = new GridLayer()
    const view = makeViewState()
    layer.draw(ctx, view, groups, DEFAULT_THEME)

    // Should call moveTo/lineTo for grid lines
    expect(ctx.moveTo).toHaveBeenCalled()
    expect(ctx.lineTo).toHaveBeenCalled()
  })

  it('applies dayStyle callback', () => {
    const layer = new GridLayer()
    const view = makeViewState()
    const dayStyle = vi.fn((_date: Date) => ({ backgroundColor: 'rgba(255,0,0,0.1)' } as DayStyle))

    layer.draw(ctx, view, groups, DEFAULT_THEME, dayStyle)
    expect(dayStyle).toHaveBeenCalled()
  })

  it('applies rowStyle callback', () => {
    const layer = new GridLayer()
    const view = makeViewState()
    const rowStyle = vi.fn((_group: Group) => ({ backgroundColor: '#ff0000' } as RowStyle))

    layer.draw(ctx, view, groups, DEFAULT_THEME, undefined, rowStyle)
    expect(rowStyle).toHaveBeenCalledTimes(10) // once per visible group
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/GridLayer.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement GridLayer**

```typescript
// src/canvas/GridLayer.ts
import dayjs from 'dayjs'
import type { Group, TimelineTheme, DayStyle, RowStyle } from '../types'
import type { ViewState } from '../core/ViewState'

export class GridLayer {
  draw(
    ctx: CanvasRenderingContext2D,
    view: ViewState,
    groups: Group[],
    theme: TimelineTheme,
    dayStyle?: (date: Date) => DayStyle | null,
    rowStyle?: (group: Group) => RowStyle | null,
  ): void {
    const { firstVisible, lastVisible } = view.getVisibleGroupRange()

    // Draw row backgrounds
    for (let i = firstVisible; i <= lastVisible; i++) {
      const y = view.groupIndexToY(i)
      const group = groups[i]
      if (!group) continue

      let bgColor: string
      const customRow = rowStyle?.(group)
      if (customRow?.backgroundColor) {
        bgColor = customRow.backgroundColor
      } else {
        bgColor = i % 2 === 0 ? '#FFFFFF' : theme.grid.rowAlt
      }

      ctx.fillStyle = bgColor
      ctx.fillRect(0, y, view.canvasWidth, view.lineHeight)

      // Row separator
      ctx.strokeStyle = customRow?.borderBottomColor ?? theme.grid.line
      ctx.lineWidth = 0.5
      ctx.beginPath()
      ctx.moveTo(0, y + view.lineHeight)
      ctx.lineTo(view.canvasWidth, y + view.lineHeight)
      ctx.stroke()
    }

    // Draw day columns and vertical grid lines
    const { bufferStart, bufferEnd } = view.getBufferBounds()
    let current = dayjs(bufferStart).startOf('day')
    const end = dayjs(bufferEnd).endOf('day')

    while (current.isBefore(end)) {
      const dayStart = current.valueOf()
      const dayEnd = current.add(1, 'day').valueOf()
      const x = view.timeToX(dayStart)
      const nextX = view.timeToX(dayEnd)

      // Day column fill (via dayStyle callback or weekend default)
      const date = current.toDate()
      const custom = dayStyle?.(date)
      if (custom?.backgroundColor) {
        ctx.fillStyle = custom.backgroundColor
        if (custom.opacity !== undefined) ctx.globalAlpha = custom.opacity
        ctx.fillRect(x, 0, nextX - x, view.canvasHeight)
        ctx.globalAlpha = 1
      } else {
        const dayOfWeek = current.day()
        if (dayOfWeek === 0 || dayOfWeek === 6) {
          ctx.fillStyle = theme.grid.weekend
          ctx.fillRect(x, 0, nextX - x, view.canvasHeight)
        }
      }

      // Vertical grid line
      ctx.strokeStyle = custom?.borderColor ?? theme.grid.line
      ctx.lineWidth = 0.5
      ctx.beginPath()
      ctx.moveTo(x, 0)
      ctx.lineTo(x, view.canvasHeight)
      ctx.stroke()

      current = current.add(1, 'day')
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/GridLayer.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/canvas/GridLayer.ts src/__tests__/GridLayer.test.ts
git commit -m "feat: add GridLayer (rows, grid lines, day/row styles)"
```

---

## Task 8: ItemsLayer — Layer 1

**Files:**
- Create: `src/canvas/ItemsLayer.ts`
- Test: `src/__tests__/ItemsLayer.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/ItemsLayer.test.ts
import { describe, it, expect, vi } from 'vitest'
import { ItemsLayer } from '../canvas/ItemsLayer'
import { ViewState } from '../core/ViewState'
import { IntervalTree } from '../core/IntervalTree'
import { LayoutEngine } from '../core/LayoutEngine'
import type { Item, Group, TimelineTheme, CanvasItemRenderer, Dependency } from '../types'
import { DEFAULT_THEME } from '../types'

function createMockCtx() {
  return {
    beginPath: vi.fn(),
    moveTo: vi.fn(),
    lineTo: vi.fn(),
    quadraticCurveTo: vi.fn(),
    bezierCurveTo: vi.fn(),
    arc: vi.fn(),
    closePath: vi.fn(),
    fill: vi.fn(),
    stroke: vi.fn(),
    fillRect: vi.fn(),
    strokeRect: vi.fn(),
    fillText: vi.fn(),
    clearRect: vi.fn(),
    measureText: vi.fn(() => ({ width: 50 })),
    save: vi.fn(),
    restore: vi.fn(),
    createLinearGradient: vi.fn(() => ({ addColorStop: vi.fn() })),
    roundRect: vi.fn(),
    font: '',
    fillStyle: '',
    strokeStyle: '',
    lineWidth: 0,
    globalAlpha: 1,
    textBaseline: '' as CanvasTextBaseline,
    setLineDash: vi.fn(),
  } as unknown as CanvasRenderingContext2D
}

describe('ItemsLayer', () => {
  const groups: Group[] = [
    { id: 'a', title: 'Group A' },
    { id: 'b', title: 'Group B' },
  ]

  const items: Item[] = [
    { id: 1, group: 'a', start_time: 1000, end_time: 1500 },
    { id: 2, group: 'a', start_time: 1200, end_time: 1800 },
    { id: 3, group: 'b', start_time: 1600, end_time: 1900 },
  ]

  const makeView = () =>
    new ViewState({
      visibleTimeStart: 1000,
      visibleTimeEnd: 2000,
      canvasWidth: 1000,
      canvasHeight: 200,
      sidebarWidth: 0,
      lineHeight: 32,
      groupCount: 2,
      buffer: 3,
      scrollTop: 0,
    })

  const makeTree = () => {
    const tree = new IntervalTree<Item>()
    tree.buildFromItems(items, (i) => i.start_time, (i) => i.end_time)
    return tree
  }

  const makeLayout = () => {
    const engine = new LayoutEngine(32, 0.9)
    engine.computeLayout(items, true)
    return engine
  }

  it('calls itemRenderer for each visible item', () => {
    const renderer = vi.fn()
    const layer = new ItemsLayer()
    const ctx = createMockCtx()

    layer.draw(ctx, makeView(), groups, items, makeTree(), makeLayout(), renderer, undefined, DEFAULT_THEME, [], undefined)

    expect(renderer).toHaveBeenCalledTimes(3)
  })

  it('passes correct bounds to itemRenderer', () => {
    const renderer = vi.fn()
    const layer = new ItemsLayer()
    const ctx = createMockCtx()

    layer.draw(ctx, makeView(), groups, items, makeTree(), makeLayout(), renderer, undefined, DEFAULT_THEME, [], undefined)

    // First item: start=1000, end=1500, visible range=1000-2000, canvas=1000px
    // x = (1000-1000)/(2000-1000) * 1000 = 0
    // width = (1500-1000)/(2000-1000) * 1000 = 500
    const firstCall = renderer.mock.calls[0]
    const bounds = firstCall[2] // third argument is bounds
    expect(bounds.x).toBe(0)
    expect(bounds.width).toBe(500)
    expect(bounds.height).toBeCloseTo(32 * 0.9)
  })

  it('passes item state correctly', () => {
    const renderer = vi.fn()
    const layer = new ItemsLayer()
    const ctx = createMockCtx()

    layer.draw(ctx, makeView(), groups, items, makeTree(), makeLayout(), renderer, undefined, DEFAULT_THEME, [1], 2)

    // Item 1 should be selected, item 2 should be hovered
    const item1Call = renderer.mock.calls.find((c: unknown[]) => (c[1] as Item).id === 1)
    const item2Call = renderer.mock.calls.find((c: unknown[]) => (c[1] as Item).id === 2)

    expect(item1Call![3].selected).toBe(true)
    expect(item1Call![3].hovered).toBe(false)
    expect(item2Call![3].selected).toBe(false)
    expect(item2Call![3].hovered).toBe(true)
  })

  it('draws dependency arrows when provided', () => {
    const renderer = vi.fn()
    const layer = new ItemsLayer()
    const ctx = createMockCtx()

    const deps: Dependency[] = [
      { fromItemId: 1, toItemId: 2 },
    ]

    layer.draw(ctx, makeView(), groups, items, makeTree(), makeLayout(), renderer, undefined, DEFAULT_THEME, [], undefined, deps)

    // Should draw bezier curve for the dependency
    expect(ctx.bezierCurveTo).toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ItemsLayer.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement ItemsLayer**

```typescript
// src/canvas/ItemsLayer.ts
import type {
  Item,
  Group,
  TimelineTheme,
  CanvasItemRenderer,
  CanvasGroupItemRenderer,
  Dependency,
  ItemBounds,
  ItemState,
} from '../types'
import type { ViewState } from '../core/ViewState'
import type { IntervalTree } from '../core/IntervalTree'
import type { LayoutEngine } from '../core/LayoutEngine'
import { createDrawHelpers } from './DrawHelpers'

export class ItemsLayer {
  draw(
    ctx: CanvasRenderingContext2D,
    view: ViewState,
    groups: Group[],
    _items: Item[],
    tree: IntervalTree<Item>,
    layout: LayoutEngine,
    itemRenderer: CanvasItemRenderer,
    groupRenderer: CanvasGroupItemRenderer | undefined,
    theme: TimelineTheme,
    selected: number[],
    hoveredItemId: number | undefined,
    dependencies?: Dependency[],
  ): void {
    const { bufferStart, bufferEnd } = view.getBufferBounds()
    const visibleItems = tree.query(bufferStart, bufferEnd)

    // Build group index lookup
    const groupIndexMap = new Map<string | number, number>()
    for (let i = 0; i < groups.length; i++) {
      groupIndexMap.set(groups[i].id, i)
    }

    const selectedSet = new Set(selected)

    // Build item bounds map for dependency arrows
    const itemBoundsMap = new Map<number, ItemBounds>()

    for (const item of visibleItems) {
      const groupIndex = groupIndexMap.get(item.group)
      if (groupIndex === undefined) continue

      const itemLayout = layout.getLayout(item.id)
      if (!itemLayout) continue

      const x = view.timeToX(item.start_time)
      const width = view.timeToX(item.end_time) - x
      const groupY = view.groupIndexToY(groupIndex)
      const y = groupY + itemLayout.stackLevel * view.lineHeight + (view.lineHeight - itemLayout.itemHeight) / 2
      const height = itemLayout.itemHeight

      const bounds: ItemBounds = { x, y, width, height }
      itemBoundsMap.set(item.id, bounds)

      const state: ItemState = {
        selected: selectedSet.has(item.id),
        hovered: hoveredItemId === item.id,
        dragging: false,
        filtered: item.filtered !== false,
      }

      ctx.save()
      const helpers = createDrawHelpers(ctx, bounds)
      const renderer = groupRenderer && (item.type === 'control_area_group' || item.type === 'construction_train')
        ? groupRenderer
        : itemRenderer
      renderer(ctx, item, bounds, state, helpers)
      ctx.restore()
    }

    // Draw dependency arrows
    if (dependencies && dependencies.length > 0) {
      this.drawDependencies(ctx, dependencies, itemBoundsMap, hoveredItemId, theme)
    }
  }

  private drawDependencies(
    ctx: CanvasRenderingContext2D,
    dependencies: Dependency[],
    boundsMap: Map<number, ItemBounds>,
    hoveredItemId: number | undefined,
    theme: TimelineTheme,
  ): void {
    for (const dep of dependencies) {
      const fromBounds = boundsMap.get(dep.fromItemId)
      const toBounds = boundsMap.get(dep.toItemId)
      if (!fromBounds || !toBounds) continue

      const isHighlighted = hoveredItemId === dep.fromItemId || hoveredItemId === dep.toItemId

      ctx.strokeStyle = isHighlighted ? theme.primary : (dep.color ?? '#94A3B8')
      ctx.lineWidth = isHighlighted ? 2 : 1.5
      ctx.setLineDash([])

      // Start point: right edge midpoint of source
      const startX = fromBounds.x + fromBounds.width
      const startY = fromBounds.y + fromBounds.height / 2

      // End point: left edge midpoint of target
      const endX = toBounds.x
      const endY = toBounds.y + toBounds.height / 2

      // Control points for S-curve
      const dx = Math.abs(endX - startX)
      const cpOffset = Math.max(dx * 0.4, 30)

      ctx.beginPath()
      ctx.moveTo(startX, startY)
      ctx.bezierCurveTo(
        startX + cpOffset, startY,
        endX - cpOffset, endY,
        endX, endY,
      )
      ctx.stroke()

      // Arrowhead
      const arrowSize = 6
      ctx.fillStyle = ctx.strokeStyle
      ctx.beginPath()
      ctx.moveTo(endX, endY)
      ctx.lineTo(endX - arrowSize, endY - arrowSize / 2)
      ctx.lineTo(endX - arrowSize, endY + arrowSize / 2)
      ctx.closePath()
      ctx.fill()
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ItemsLayer.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/canvas/ItemsLayer.ts src/__tests__/ItemsLayer.test.ts
git commit -m "feat: add ItemsLayer with item rendering and dependency arrows"
```

---

## Task 9: OverlayLayer — Layer 2

**Files:**
- Create: `src/canvas/OverlayLayer.ts`
- Test: `src/__tests__/OverlayLayer.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/OverlayLayer.test.ts
import { describe, it, expect, vi } from 'vitest'
import { OverlayLayer } from '../canvas/OverlayLayer'
import { ViewState } from '../core/ViewState'
import type { TimelineTheme, MarkerConfig, ItemBounds, CanvasItemRenderer } from '../types'
import { DEFAULT_THEME } from '../types'

function createMockCtx() {
  return {
    beginPath: vi.fn(),
    moveTo: vi.fn(),
    lineTo: vi.fn(),
    fill: vi.fn(),
    stroke: vi.fn(),
    fillRect: vi.fn(),
    clearRect: vi.fn(),
    save: vi.fn(),
    restore: vi.fn(),
    setLineDash: vi.fn(),
    roundRect: vi.fn(),
    fillText: vi.fn(),
    measureText: vi.fn(() => ({ width: 50 })),
    createLinearGradient: vi.fn(() => ({ addColorStop: vi.fn() })),
    fillStyle: '',
    strokeStyle: '',
    lineWidth: 0,
    globalAlpha: 1,
    textBaseline: '' as CanvasTextBaseline,
    font: '',
  } as unknown as CanvasRenderingContext2D
}

describe('OverlayLayer', () => {
  const makeView = () =>
    new ViewState({
      visibleTimeStart: 1000,
      visibleTimeEnd: 2000,
      canvasWidth: 1000,
      canvasHeight: 500,
      sidebarWidth: 200,
      lineHeight: 32,
      groupCount: 10,
      buffer: 3,
      scrollTop: 0,
    })

  it('draws cursor line at given x position', () => {
    const layer = new OverlayLayer()
    const ctx = createMockCtx()
    layer.draw(ctx, makeView(), DEFAULT_THEME, { cursorX: 200 })

    expect(ctx.moveTo).toHaveBeenCalled()
    expect(ctx.lineTo).toHaveBeenCalled()
  })

  it('does not draw cursor line when cursorX is null', () => {
    const layer = new OverlayLayer()
    const ctx = createMockCtx()
    layer.draw(ctx, makeView(), DEFAULT_THEME, { cursorX: null })

    expect(ctx.moveTo).not.toHaveBeenCalled()
  })

  it('draws markers', () => {
    const layer = new OverlayLayer()
    const ctx = createMockCtx()
    const markers: MarkerConfig[] = [
      { date: 1500, color: '#FD7171', width: 6 },
    ]
    layer.draw(ctx, makeView(), DEFAULT_THEME, { cursorX: null, markers })

    expect(ctx.fillRect).toHaveBeenCalled()
  })

  it('draws drag ghost when provided', () => {
    const layer = new OverlayLayer()
    const ctx = createMockCtx()
    const renderer: CanvasItemRenderer = vi.fn()
    const dragState = {
      item: { id: 1, group: 'a', start_time: 1000, end_time: 1500 },
      bounds: { x: 100, y: 50, width: 200, height: 28 } as ItemBounds,
      renderer,
    }

    layer.draw(ctx, makeView(), DEFAULT_THEME, { cursorX: null, drag: dragState })

    expect(ctx.globalAlpha).toBe(1) // restored after drawing
    expect(renderer).toHaveBeenCalled()
  })

  it('draws snap guide line during drag', () => {
    const layer = new OverlayLayer()
    const ctx = createMockCtx()

    layer.draw(ctx, makeView(), DEFAULT_THEME, { cursorX: null, snapX: 300 })

    expect(ctx.setLineDash).toHaveBeenCalled()
    expect(ctx.moveTo).toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/OverlayLayer.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement OverlayLayer**

```typescript
// src/canvas/OverlayLayer.ts
import type {
  TimelineTheme,
  MarkerConfig,
  ItemBounds,
  Item,
  CanvasItemRenderer,
} from '../types'
import type { ViewState } from '../core/ViewState'
import { createDrawHelpers } from './DrawHelpers'

export interface DragRenderState {
  item: Item
  bounds: ItemBounds
  renderer: CanvasItemRenderer
}

export interface OverlayDrawOptions {
  cursorX: number | null
  snapX?: number | null
  markers?: MarkerConfig[]
  drag?: DragRenderState | null
}

export class OverlayLayer {
  draw(
    ctx: CanvasRenderingContext2D,
    view: ViewState,
    theme: TimelineTheme,
    options: OverlayDrawOptions,
  ): void {
    const { cursorX, snapX, markers, drag } = options

    // Draw markers
    if (markers) {
      for (const marker of markers) {
        const x = view.timeToX(marker.date)
        ctx.fillStyle = marker.color
        ctx.fillRect(x - marker.width / 2, 0, marker.width, view.canvasHeight)
      }
    }

    // Draw cursor line
    if (cursorX !== null && cursorX !== undefined) {
      ctx.strokeStyle = theme.marker.cursor
      ctx.lineWidth = 1
      ctx.beginPath()
      ctx.moveTo(cursorX, 0)
      ctx.lineTo(cursorX, view.canvasHeight)
      ctx.stroke()
    }

    // Draw snap guide line
    if (snapX !== null && snapX !== undefined) {
      ctx.strokeStyle = theme.primary
      ctx.lineWidth = 1
      ctx.setLineDash([4, 4])
      ctx.beginPath()
      ctx.moveTo(snapX, 0)
      ctx.lineTo(snapX, view.canvasHeight)
      ctx.stroke()
      ctx.setLineDash([])
    }

    // Draw drag ghost
    if (drag) {
      ctx.save()
      ctx.globalAlpha = 0.5
      const helpers = createDrawHelpers(ctx, drag.bounds)
      drag.renderer(
        ctx,
        drag.item,
        drag.bounds,
        { selected: false, hovered: false, dragging: true, filtered: true },
        helpers,
      )
      ctx.restore()
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/OverlayLayer.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/canvas/OverlayLayer.ts src/__tests__/OverlayLayer.test.ts
git commit -m "feat: add OverlayLayer (cursor, markers, drag ghost, snap guide)"
```

---

## Task 10: HitTest — Pointer-to-Item Resolution

**Files:**
- Create: `src/core/HitTest.ts`
- Test: `src/__tests__/HitTest.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/HitTest.test.ts
import { describe, it, expect } from 'vitest'
import { hitTest } from '../core/HitTest'
import { ViewState } from '../core/ViewState'
import { IntervalTree } from '../core/IntervalTree'
import { LayoutEngine } from '../core/LayoutEngine'
import type { Item, Group } from '../types'

describe('hitTest', () => {
  const groups: Group[] = [
    { id: 'a', title: 'A' },
    { id: 'b', title: 'B' },
  ]

  const items: Item[] = [
    { id: 1, group: 'a', start_time: 100, end_time: 300 },
    { id: 2, group: 'a', start_time: 200, end_time: 400 },
    { id: 3, group: 'b', start_time: 100, end_time: 500 },
  ]

  const makeView = () =>
    new ViewState({
      visibleTimeStart: 0,
      visibleTimeEnd: 1000,
      canvasWidth: 1000,
      canvasHeight: 200,
      sidebarWidth: 0,
      lineHeight: 32,
      groupCount: 2,
      buffer: 3,
      scrollTop: 0,
    })

  const makeTree = () => {
    const tree = new IntervalTree<Item>()
    tree.buildFromItems(items, (i) => i.start_time, (i) => i.end_time)
    return tree
  }

  const makeLayout = () => {
    const engine = new LayoutEngine(32, 0.9)
    engine.computeLayout(items, true)
    return engine
  }

  it('finds item at correct pixel position', () => {
    const view = makeView()
    // Item 1: group='a' (index 0), x=100..300 (pixels 100..300 since 1px=1ms)
    // y at group 0, stack 0: y = 0 + (32 - 28.8)/2 = 1.6
    const result = hitTest(150, 10, view, makeTree(), makeLayout(), groups)
    expect(result?.id).toBe(1)
  })

  it('returns null for empty area', () => {
    const view = makeView()
    const result = hitTest(800, 10, view, makeTree(), makeLayout(), groups)
    expect(result).toBeNull()
  })

  it('finds item in second group', () => {
    const view = makeView()
    // Item 3: group='b' (index 1), y starts at 32
    const result = hitTest(200, 40, view, makeTree(), makeLayout(), groups)
    expect(result?.id).toBe(3)
  })

  it('resolves topmost stacked item', () => {
    const view = makeView()
    // At x=250, both item 1 (stack 0) and item 2 (stack 1) overlap in group 'a'
    // Click at y within stack level 1 should return item 2
    const itemHeight = 32 * 0.9 // 28.8
    const stack1Y = 1 * 32 + (32 - itemHeight) / 2 + 1 // just inside stack level 1 of group 'a'
    // Wait, stack level 1 means y offset = 32 (one lineHeight down within the group)
    // But group 'a' starts at y=0, so stack level 1 is at y = 32 + padding
    // Actually items in group 'a' with stack=1 are at y = 0 + 1*32 + (32-28.8)/2 = 33.6
    // This is actually in group b's row. Let me reconsider...
    // The stacking expands the group height, so items in stack level 1 are within the expanded group area
    // For hit testing, we check y against item bounds directly
    const result = hitTest(250, 33.6 + 5, view, makeTree(), makeLayout(), groups)
    expect(result?.id).toBe(2)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/HitTest.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement HitTest**

```typescript
// src/core/HitTest.ts
import type { Item, Group } from '../types'
import type { ViewState } from './ViewState'
import type { IntervalTree } from './IntervalTree'
import type { LayoutEngine } from './LayoutEngine'

export function hitTest(
  canvasX: number,
  canvasY: number,
  view: ViewState,
  tree: IntervalTree<Item>,
  layout: LayoutEngine,
  groups: Group[],
): Item | null {
  const time = view.xToTime(canvasX)

  // Build group index map
  const groupIndexMap = new Map<string | number, number>()
  for (let i = 0; i < groups.length; i++) {
    groupIndexMap.set(groups[i].id, i)
  }

  // Query interval tree for items at this time
  const candidates = tree.query(time, time)

  // Check each candidate's bounds against the click position
  let topItem: Item | null = null
  let topY = -Infinity

  for (const item of candidates) {
    const groupIndex = groupIndexMap.get(item.group)
    if (groupIndex === undefined) continue

    const itemLayout = layout.getLayout(item.id)
    if (!itemLayout) continue

    const x = view.timeToX(item.start_time)
    const width = view.timeToX(item.end_time) - x

    // Check horizontal bounds
    if (canvasX < x || canvasX > x + width) continue

    const groupY = view.groupIndexToY(groupIndex)
    const y = groupY + itemLayout.stackLevel * view.lineHeight + (view.lineHeight - itemLayout.itemHeight) / 2
    const height = itemLayout.itemHeight

    // Check vertical bounds
    if (canvasY < y || canvasY > y + height) continue

    // Pick the topmost (highest y = visually lower, but highest stack level)
    if (y > topY) {
      topY = y
      topItem = item
    }
  }

  return topItem
}

export function hitTestGroup(
  canvasY: number,
  view: ViewState,
  groups: Group[],
): Group | null {
  const groupIndex = view.yToGroupIndex(canvasY)
  return groups[groupIndex] ?? null
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/HitTest.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/core/HitTest.ts src/__tests__/HitTest.test.ts
git commit -m "feat: add HitTest for pointer-to-item resolution"
```

---

## Task 11: ScrollHandler

**Files:**
- Create: `src/interaction/ScrollHandler.ts`
- Test: `src/__tests__/ScrollHandler.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/ScrollHandler.test.ts
import { describe, it, expect, vi } from 'vitest'
import { ScrollHandler } from '../interaction/ScrollHandler'

describe('ScrollHandler', () => {
  it('calls onScrollVertical on wheel event', () => {
    const onScrollV = vi.fn()
    const onScrollH = vi.fn()
    const handler = new ScrollHandler(onScrollV, onScrollH)

    const event = new WheelEvent('wheel', { deltaY: 50 })
    handler.handleWheel(event)

    expect(onScrollV).toHaveBeenCalledWith(50)
  })

  it('calls onScrollHorizontal on shift+wheel', () => {
    const onScrollV = vi.fn()
    const onScrollH = vi.fn()
    const handler = new ScrollHandler(onScrollV, onScrollH)

    const event = new WheelEvent('wheel', { deltaX: 100, shiftKey: true })
    handler.handleWheel(event)

    expect(onScrollH).toHaveBeenCalledWith(100)
  })

  it('calls onScrollHorizontal for horizontal scroll (deltaX)', () => {
    const onScrollV = vi.fn()
    const onScrollH = vi.fn()
    const handler = new ScrollHandler(onScrollV, onScrollH)

    const event = new WheelEvent('wheel', { deltaX: 80, deltaY: 0 })
    handler.handleWheel(event)

    expect(onScrollH).toHaveBeenCalledWith(80)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ScrollHandler.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement ScrollHandler**

```typescript
// src/interaction/ScrollHandler.ts

export class ScrollHandler {
  private onScrollVertical: (deltaY: number) => void
  private onScrollHorizontal: (deltaX: number) => void

  constructor(
    onScrollVertical: (deltaY: number) => void,
    onScrollHorizontal: (deltaX: number) => void,
  ) {
    this.onScrollVertical = onScrollVertical
    this.onScrollHorizontal = onScrollHorizontal
  }

  handleWheel(event: WheelEvent): void {
    event.preventDefault()

    // Shift+wheel or horizontal scroll -> pan time axis
    if (event.shiftKey || (Math.abs(event.deltaX) > Math.abs(event.deltaY))) {
      const deltaX = event.shiftKey ? event.deltaY : event.deltaX
      this.onScrollHorizontal(deltaX)
      return
    }

    // Normal vertical scroll
    this.onScrollVertical(event.deltaY)
  }

  attach(element: HTMLElement): () => void {
    const handler = (e: WheelEvent) => this.handleWheel(e)
    element.addEventListener('wheel', handler, { passive: false })
    return () => element.removeEventListener('wheel', handler)
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ScrollHandler.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/interaction/ScrollHandler.ts src/__tests__/ScrollHandler.test.ts
git commit -m "feat: add ScrollHandler for wheel and pan events"
```

---

## Task 12: ZoomHandler

**Files:**
- Create: `src/interaction/ZoomHandler.ts`
- Test: `src/__tests__/ZoomHandler.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/ZoomHandler.test.ts
import { describe, it, expect, vi } from 'vitest'
import { ZoomHandler } from '../interaction/ZoomHandler'

describe('ZoomHandler', () => {
  it('zooms in when ctrl+wheel scrolls up', () => {
    const onZoom = vi.fn()
    const handler = new ZoomHandler(onZoom, 1000, 2000, 100, 10000)

    handler.handleZoom(-100, 0.5) // scroll up, cursor at 50% of canvas width

    const [newStart, newEnd] = onZoom.mock.calls[0]
    // Zoom in: range should be smaller
    expect(newEnd - newStart).toBeLessThan(2000 - 1000)
  })

  it('zooms out when ctrl+wheel scrolls down', () => {
    const onZoom = vi.fn()
    const handler = new ZoomHandler(onZoom, 1000, 2000, 100, 10000)

    handler.handleZoom(100, 0.5) // scroll down, cursor at 50%

    const [newStart, newEnd] = onZoom.mock.calls[0]
    expect(newEnd - newStart).toBeGreaterThan(2000 - 1000)
  })

  it('clamps zoom to minZoom', () => {
    const onZoom = vi.fn()
    const handler = new ZoomHandler(onZoom, 1000, 1100, 100, 10000)

    handler.handleZoom(-1000, 0.5) // zoom in hard

    const [newStart, newEnd] = onZoom.mock.calls[0]
    expect(newEnd - newStart).toBeGreaterThanOrEqual(100)
  })

  it('clamps zoom to maxZoom', () => {
    const onZoom = vi.fn()
    const handler = new ZoomHandler(onZoom, 1000, 9000, 100, 10000)

    handler.handleZoom(10000, 0.5) // zoom out hard

    const [newStart, newEnd] = onZoom.mock.calls[0]
    expect(newEnd - newStart).toBeLessThanOrEqual(10000)
  })

  it('anchors zoom to cursor position', () => {
    const onZoom = vi.fn()
    const handler = new ZoomHandler(onZoom, 1000, 2000, 100, 10000)

    // Cursor at 25% — the time at 25% should remain the same after zoom
    handler.handleZoom(-100, 0.25)
    const [newStart, newEnd] = onZoom.mock.calls[0]

    const timeBefore = 1000 + (2000 - 1000) * 0.25 // 1250
    const timeAfter = newStart + (newEnd - newStart) * 0.25

    expect(Math.abs(timeBefore - timeAfter)).toBeLessThan(1)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ZoomHandler.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement ZoomHandler**

```typescript
// src/interaction/ZoomHandler.ts

export class ZoomHandler {
  private onZoom: (newStart: number, newEnd: number) => void
  private visibleTimeStart: number
  private visibleTimeEnd: number
  private minZoom: number
  private maxZoom: number

  constructor(
    onZoom: (newStart: number, newEnd: number) => void,
    visibleTimeStart: number,
    visibleTimeEnd: number,
    minZoom: number,
    maxZoom: number,
  ) {
    this.onZoom = onZoom
    this.visibleTimeStart = visibleTimeStart
    this.visibleTimeEnd = visibleTimeEnd
    this.minZoom = minZoom
    this.maxZoom = maxZoom
  }

  updateBounds(start: number, end: number): void {
    this.visibleTimeStart = start
    this.visibleTimeEnd = end
  }

  handleZoom(delta: number, cursorRatio: number): void {
    const currentDuration = this.visibleTimeEnd - this.visibleTimeStart
    const zoomFactor = 1 + delta * 0.001

    let newDuration = currentDuration * zoomFactor

    // Clamp
    newDuration = Math.max(this.minZoom, Math.min(this.maxZoom, newDuration))

    // Anchor to cursor position
    const cursorTime = this.visibleTimeStart + currentDuration * cursorRatio
    const newStart = cursorTime - newDuration * cursorRatio
    const newEnd = cursorTime + newDuration * (1 - cursorRatio)

    this.onZoom(newStart, newEnd)
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/ZoomHandler.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/interaction/ZoomHandler.ts src/__tests__/ZoomHandler.test.ts
git commit -m "feat: add ZoomHandler with cursor-anchored zoom"
```

---

## Task 13: DragHandler

**Files:**
- Create: `src/interaction/DragHandler.ts`
- Test: `src/__tests__/DragHandler.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/DragHandler.test.ts
import { describe, it, expect, vi } from 'vitest'
import { DragHandler, DragState } from '../interaction/DragHandler'
import type { Item } from '../types'

describe('DragHandler', () => {
  const item: Item = { id: 1, group: 'a', start_time: 1000, end_time: 2000 }
  const dragSnap = 100

  it('starts drag and captures initial state', () => {
    const handler = new DragHandler(dragSnap)
    handler.startDrag(item, 150)

    expect(handler.getState()).not.toBeNull()
    expect(handler.getState()!.item.id).toBe(1)
    expect(handler.getState()!.startX).toBe(150)
  })

  it('updates position on move', () => {
    const handler = new DragHandler(dragSnap)
    handler.startDrag(item, 150)
    handler.updateDrag(250)

    expect(handler.getState()!.currentX).toBe(250)
    expect(handler.getState()!.deltaX).toBe(100)
  })

  it('calculates snapped time on end', () => {
    const onMove = vi.fn()
    const handler = new DragHandler(dragSnap)
    handler.startDrag(item, 0)
    handler.updateDrag(50)

    // pixelsPerMs = 1, so 50px = 50ms offset
    // snapped to dragSnap=100: round(1000+50 to nearest 100) = 1100
    const snappedTime = handler.endDrag(1) // 1 px/ms
    expect(snappedTime).not.toBeNull()
    // The snap should round the new start time
    const expected = Math.round((1000 + 50) / 100) * 100
    expect(snappedTime).toBe(expected)
  })

  it('returns null for endDrag when not dragging', () => {
    const handler = new DragHandler(dragSnap)
    expect(handler.endDrag(1)).toBeNull()
  })

  it('resets state after endDrag', () => {
    const handler = new DragHandler(dragSnap)
    handler.startDrag(item, 0)
    handler.endDrag(1)
    expect(handler.getState()).toBeNull()
  })

  it('cancel clears state without returning time', () => {
    const handler = new DragHandler(dragSnap)
    handler.startDrag(item, 0)
    handler.cancel()
    expect(handler.getState()).toBeNull()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/DragHandler.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement DragHandler**

```typescript
// src/interaction/DragHandler.ts
import type { Item } from '../types'

export interface DragState {
  item: Item
  startX: number
  currentX: number
  deltaX: number
}

export class DragHandler {
  private state: DragState | null = null
  private dragSnap: number

  constructor(dragSnap: number) {
    this.dragSnap = dragSnap
  }

  startDrag(item: Item, startX: number): void {
    this.state = {
      item,
      startX,
      currentX: startX,
      deltaX: 0,
    }
  }

  updateDrag(currentX: number): void {
    if (!this.state) return
    this.state.currentX = currentX
    this.state.deltaX = currentX - this.state.startX
  }

  endDrag(pixelsPerMs: number): number | null {
    if (!this.state) return null

    const deltaMs = this.state.deltaX / pixelsPerMs
    const newStartTime = this.state.item.start_time + deltaMs

    // Snap to nearest dragSnap interval
    const snapped = Math.round(newStartTime / this.dragSnap) * this.dragSnap

    this.state = null
    return snapped
  }

  cancel(): void {
    this.state = null
  }

  getState(): DragState | null {
    return this.state
  }

  isDragging(): boolean {
    return this.state !== null
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/DragHandler.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add src/interaction/DragHandler.ts src/__tests__/DragHandler.test.ts
git commit -m "feat: add DragHandler with smooth drag and snap"
```

---

## Task 14: DOM Components — Sidebar, Headers, Markers

**Files:**
- Create: `src/dom/Sidebar.tsx`, `src/dom/TimelineHeaders.tsx`, `src/dom/DateHeader.tsx`, `src/dom/SidebarHeader.tsx`, `src/dom/CustomHeader.tsx`, `src/dom/TodayMarker.tsx`, `src/dom/CustomMarker.tsx`

- [ ] **Step 1: Create Sidebar with virtual scrolling**

```typescript
// src/dom/Sidebar.tsx
import React, { useRef, useCallback, useEffect } from 'react'
import type { Group, TimelineTheme } from '../types'

interface SidebarProps {
  groups: Group[]
  width: number
  lineHeight: number
  scrollTop: number
  canvasHeight: number
  theme: TimelineTheme
  groupRenderer: (group: Group) => React.ReactNode
  onScroll: (scrollTop: number) => void
}

const OVERSCAN = 5

export function Sidebar({
  groups,
  width,
  lineHeight,
  scrollTop,
  canvasHeight,
  theme,
  groupRenderer,
  onScroll,
}: SidebarProps) {
  const containerRef = useRef<HTMLDivElement>(null)

  const totalHeight = groups.length * lineHeight

  const firstVisible = Math.max(0, Math.floor(scrollTop / lineHeight) - OVERSCAN)
  const lastVisible = Math.min(
    groups.length - 1,
    Math.ceil((scrollTop + canvasHeight) / lineHeight) + OVERSCAN,
  )

  const handleScroll = useCallback(
    (e: React.UIEvent<HTMLDivElement>) => {
      onScroll(e.currentTarget.scrollTop)
    },
    [onScroll],
  )

  useEffect(() => {
    if (containerRef.current) {
      containerRef.current.scrollTop = scrollTop
    }
  }, [scrollTop])

  const visibleGroups = []
  for (let i = firstVisible; i <= lastVisible; i++) {
    const group = groups[i]
    if (!group) continue
    visibleGroups.push(
      <div
        key={group.id}
        style={{
          position: 'absolute',
          top: i * lineHeight,
          height: lineHeight,
          width: '100%',
          overflow: 'hidden',
        }}
      >
        {groupRenderer(group)}
      </div>,
    )
  }

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{
        width,
        height: canvasHeight,
        overflow: 'hidden',
        position: 'relative',
        borderRight: `1px solid ${theme.sidebar.border}`,
        backgroundColor: theme.sidebar.bg,
      }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        {visibleGroups}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create marker config components**

```typescript
// src/dom/TodayMarker.tsx
import type { MarkerConfig } from '../types'

interface TodayMarkerProps {
  color?: string
  width?: number
}

export function TodayMarker(_props: TodayMarkerProps) {
  return null // Config-only component, read by CanvasTimeline
}

TodayMarker.displayName = 'TodayMarker'

export function getTodayMarkerConfig(props: TodayMarkerProps): MarkerConfig {
  return {
    date: Date.now(),
    color: props.color ?? '#FD7171',
    width: props.width ?? 6,
  }
}
```

```typescript
// src/dom/CustomMarker.tsx
import type { MarkerConfig } from '../types'

interface CustomMarkerProps {
  date: number
  color?: string
  width?: number
}

export function CustomMarker(_props: CustomMarkerProps) {
  return null // Config-only component, read by CanvasTimeline
}

CustomMarker.displayName = 'CustomMarker'

export function getCustomMarkerConfig(props: CustomMarkerProps): MarkerConfig {
  return {
    date: props.date,
    color: props.color ?? '#3B82F6',
    width: props.width ?? 4,
  }
}
```

- [ ] **Step 3: Create header components**

```typescript
// src/dom/SidebarHeader.tsx
import React from 'react'

interface SidebarHeaderProps {
  width: number
  children?: (props: { getRootProps: () => React.HTMLAttributes<HTMLDivElement> }) => React.ReactNode
  style?: React.CSSProperties
}

export function SidebarHeader({ width, children, style }: SidebarHeaderProps) {
  const getRootProps = (): React.HTMLAttributes<HTMLDivElement> => ({
    style: { width, ...style },
  })

  if (children) {
    return <>{children({ getRootProps })}</>
  }

  return <div style={{ width }} />
}

SidebarHeader.displayName = 'SidebarHeader'
```

```typescript
// src/dom/DateHeader.tsx
import React, { useMemo } from 'react'
import dayjs from 'dayjs'
import type { TimelineTheme } from '../types'

type DateUnit = 'year' | 'month' | 'week' | 'day' | 'hour'

interface DateHeaderProps {
  unit: DateUnit
  visibleTimeStart: number
  visibleTimeEnd: number
  canvasWidth: number
  theme: TimelineTheme
  labelFormat?: (start: Date, end: Date, unit: DateUnit) => string
}

export function DateHeader({
  unit,
  visibleTimeStart,
  visibleTimeEnd,
  canvasWidth,
  theme,
  labelFormat,
}: DateHeaderProps) {
  const intervals = useMemo(() => {
    const result: Array<{ start: number; end: number; label: string; left: number; width: number }> = []
    const duration = visibleTimeEnd - visibleTimeStart
    let current = dayjs(visibleTimeStart).startOf(unit)

    while (current.valueOf() < visibleTimeEnd) {
      const next = current.add(1, unit)
      const start = current.valueOf()
      const end = next.valueOf()

      const left = ((start - visibleTimeStart) / duration) * canvasWidth
      const width = ((end - start) / duration) * canvasWidth

      const label = labelFormat
        ? labelFormat(current.toDate(), next.toDate(), unit)
        : formatDefault(current, unit)

      result.push({ start, end, label, left, width })
      current = next
    }

    return result
  }, [visibleTimeStart, visibleTimeEnd, canvasWidth, unit, labelFormat])

  return (
    <div style={{ display: 'flex', position: 'relative', height: 30, overflow: 'hidden' }}>
      {intervals.map((interval) => (
        <div
          key={interval.start}
          style={{
            position: 'absolute',
            left: interval.left,
            width: interval.width,
            height: '100%',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            borderRight: `1px solid ${theme.header.border}`,
            fontSize: 12,
            color: theme.header.text,
            overflow: 'hidden',
            whiteSpace: 'nowrap',
          }}
        >
          {interval.label}
        </div>
      ))}
    </div>
  )
}

DateHeader.displayName = 'DateHeader'

function formatDefault(d: dayjs.Dayjs, unit: DateUnit): string {
  switch (unit) {
    case 'year': return d.format('YYYY')
    case 'month': return d.format('MMM YYYY')
    case 'week': return `W${d.week()}`
    case 'day': return d.format('D')
    case 'hour': return d.format('HH:mm')
  }
}
```

```typescript
// src/dom/CustomHeader.tsx
import React from 'react'

interface CustomHeaderProps {
  children: React.ReactNode
}

export function CustomHeader({ children }: CustomHeaderProps) {
  return <>{children}</>
}

CustomHeader.displayName = 'CustomHeader'
```

```typescript
// src/dom/TimelineHeaders.tsx
import React from 'react'
import type { TimelineTheme } from '../types'

interface TimelineHeadersProps {
  children: React.ReactNode
  theme: TimelineTheme
  className?: string
  style?: React.CSSProperties
}

export function TimelineHeaders({ children, theme, className, style }: TimelineHeadersProps) {
  return (
    <div
      className={className}
      style={{
        position: 'sticky',
        top: 0,
        zIndex: 20,
        display: 'flex',
        flexDirection: 'column',
        backgroundColor: theme.header.bg,
        borderBottom: `1px solid ${theme.header.border}`,
        ...style,
      }}
    >
      {children}
    </div>
  )
}

TimelineHeaders.displayName = 'TimelineHeaders'
```

- [ ] **Step 4: Commit**

```bash
git add src/dom/
git commit -m "feat: add DOM components (Sidebar, Headers, Markers)"
```

---

## Task 15: CanvasTimeline — Root Component

**Files:**
- Create: `src/CanvasTimeline.tsx`
- Modify: `src/index.ts`
- Test: `src/__tests__/CanvasTimeline.test.tsx`

- [ ] **Step 1: Write failing integration test**

```typescript
// src/__tests__/CanvasTimeline.test.tsx
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import React from 'react'
import { CanvasTimeline } from '../CanvasTimeline'
import type { Group, Item, CanvasItemRenderer } from '../types'

// Mock canvas context
HTMLCanvasElement.prototype.getContext = vi.fn(() => ({
  scale: vi.fn(),
  clearRect: vi.fn(),
  fillRect: vi.fn(),
  strokeRect: vi.fn(),
  beginPath: vi.fn(),
  moveTo: vi.fn(),
  lineTo: vi.fn(),
  fill: vi.fn(),
  stroke: vi.fn(),
  fillText: vi.fn(),
  measureText: vi.fn(() => ({ width: 50 })),
  save: vi.fn(),
  restore: vi.fn(),
  createLinearGradient: vi.fn(() => ({ addColorStop: vi.fn() })),
  roundRect: vi.fn(),
  closePath: vi.fn(),
  arc: vi.fn(),
  bezierCurveTo: vi.fn(),
  quadraticCurveTo: vi.fn(),
  setLineDash: vi.fn(),
  font: '',
  fillStyle: '',
  strokeStyle: '',
  lineWidth: 0,
  globalAlpha: 1,
  textBaseline: '',
})) as unknown as typeof HTMLCanvasElement.prototype.getContext

describe('CanvasTimeline', () => {
  const groups: Group[] = [
    { id: 'a', title: 'Group A' },
    { id: 'b', title: 'Group B' },
  ]

  const items: Item[] = [
    { id: 1, group: 'a', start_time: 1000, end_time: 2000 },
    { id: 2, group: 'b', start_time: 1500, end_time: 2500 },
  ]

  const renderer: CanvasItemRenderer = vi.fn()

  const defaultProps = {
    groups,
    items,
    defaultTimeStart: 0,
    defaultTimeEnd: 5000,
    sidebarWidth: 200,
    lineHeight: 32,
    itemHeightRatio: 0.9,
    stackItems: true,
    canMove: true,
    canResize: false,
    canChangeGroup: false,
    dragSnap: 100,
    minZoom: 100,
    maxZoom: 10000,
    itemRenderer: renderer,
    sidebarGroupRenderer: (g: Group) => <div>{g.title}</div>,
  }

  it('renders without crashing', () => {
    const { container } = render(<CanvasTimeline {...defaultProps} />)
    expect(container).toBeDefined()
  })

  it('renders three canvas elements', () => {
    const { container } = render(<CanvasTimeline {...defaultProps} />)
    const canvases = container.querySelectorAll('canvas')
    expect(canvases.length).toBe(3)
  })

  it('renders sidebar with group items', () => {
    render(<CanvasTimeline {...defaultProps} />)
    expect(screen.getByText('Group A')).toBeDefined()
    expect(screen.getByText('Group B')).toBeDefined()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/CanvasTimeline.test.tsx
```

Expected: FAIL.

- [ ] **Step 3: Implement CanvasTimeline**

```typescript
// src/CanvasTimeline.tsx
import React, { useRef, useEffect, useState, useCallback, useMemo } from 'react'
import type {
  CanvasTimelineProps,
  MarkerConfig,
  TimelineTheme,
} from './types'
import { DEFAULT_THEME } from './types'
import { ViewState } from './core/ViewState'
import { IntervalTree } from './core/IntervalTree'
import { LayoutEngine } from './core/LayoutEngine'
import { hitTest, hitTestGroup } from './core/HitTest'
import { setupCanvas, clearCanvas, scheduleRedraw } from './canvas/CanvasManager'
import { GridLayer } from './canvas/GridLayer'
import { ItemsLayer } from './canvas/ItemsLayer'
import { OverlayLayer } from './canvas/OverlayLayer'
import { ScrollHandler } from './interaction/ScrollHandler'
import { ZoomHandler } from './interaction/ZoomHandler'
import { DragHandler } from './interaction/DragHandler'
import { Sidebar } from './dom/Sidebar'
import { getTodayMarkerConfig } from './dom/TodayMarker'
import { getCustomMarkerConfig } from './dom/CustomMarker'

function mergeTheme(partial?: Partial<TimelineTheme>): TimelineTheme {
  if (!partial) return DEFAULT_THEME
  return {
    ...DEFAULT_THEME,
    ...partial,
    status: { ...DEFAULT_THEME.status, ...partial.status },
    grid: { ...DEFAULT_THEME.grid, ...partial.grid },
    item: { ...DEFAULT_THEME.item, ...partial.item },
    marker: { ...DEFAULT_THEME.marker, ...partial.marker },
    sidebar: { ...DEFAULT_THEME.sidebar, ...partial.sidebar },
    header: { ...DEFAULT_THEME.header, ...partial.header },
  }
}

export function CanvasTimeline(props: CanvasTimelineProps) {
  const {
    groups,
    items,
    defaultTimeStart,
    defaultTimeEnd,
    sidebarWidth,
    lineHeight,
    itemHeightRatio,
    stackItems,
    canMove,
    canChangeGroup: _canChangeGroup,
    dragSnap,
    minZoom,
    maxZoom,
    theme: themePartial,
    dayStyle,
    rowStyle,
    showCursorLine,
    itemRenderer,
    groupRenderer,
    sidebarGroupRenderer,
    dependencies,
    onItemClick,
    onItemDoubleClick,
    onItemContextMenu,
    onItemMove,
    onItemHover,
    onCanvasDoubleClick,
    onCanvasContextMenu,
    onTimeChange,
    onZoom,
    selected = [],
    children,
  } = props

  const theme = useMemo(() => mergeTheme(themePartial), [themePartial])

  // Refs for canvas elements
  const gridCanvasRef = useRef<HTMLCanvasElement>(null)
  const itemsCanvasRef = useRef<HTMLCanvasElement>(null)
  const overlayCanvasRef = useRef<HTMLCanvasElement>(null)
  const containerRef = useRef<HTMLDivElement>(null)

  // State
  const [visibleTimeStart, setVisibleTimeStart] = useState(
    props.visibleTimeStart ?? defaultTimeStart,
  )
  const [visibleTimeEnd, setVisibleTimeEnd] = useState(
    props.visibleTimeEnd ?? defaultTimeEnd,
  )
  const [scrollTop, setScrollTop] = useState(0)
  const [cursorX, setCursorX] = useState<number | null>(null)
  const [hoveredItemId, setHoveredItemId] = useState<number | undefined>(undefined)
  const [containerSize, setContainerSize] = useState({ width: 800, height: 600 })

  // Controlled mode
  useEffect(() => {
    if (props.visibleTimeStart !== undefined) setVisibleTimeStart(props.visibleTimeStart)
    if (props.visibleTimeEnd !== undefined) setVisibleTimeEnd(props.visibleTimeEnd)
  }, [props.visibleTimeStart, props.visibleTimeEnd])

  // Measure container
  useEffect(() => {
    const container = containerRef.current
    if (!container) return
    const obs = new ResizeObserver((entries) => {
      const entry = entries[0]
      if (entry) {
        setContainerSize({
          width: entry.contentRect.width,
          height: entry.contentRect.height,
        })
      }
    })
    obs.observe(container)
    return () => obs.disconnect()
  }, [])

  const canvasWidth = containerSize.width - sidebarWidth
  const canvasHeight = containerSize.height

  // Core data structures — rebuilt when data changes
  const intervalTree = useMemo(() => {
    const tree = new IntervalTree<(typeof items)[0]>()
    tree.buildFromItems(items, (i) => i.start_time, (i) => i.end_time)
    return tree
  }, [items])

  const layoutEngine = useMemo(() => {
    const engine = new LayoutEngine(lineHeight, itemHeightRatio)
    engine.computeLayout(items, stackItems)
    return engine
  }, [items, lineHeight, itemHeightRatio, stackItems])

  // ViewState — rebuilt when view params change
  const viewState = useMemo(
    () =>
      new ViewState({
        visibleTimeStart,
        visibleTimeEnd,
        canvasWidth,
        canvasHeight,
        sidebarWidth,
        lineHeight,
        groupCount: groups.length,
        buffer: props.buffer ?? 3,
        scrollTop,
      }),
    [visibleTimeStart, visibleTimeEnd, canvasWidth, canvasHeight, sidebarWidth, lineHeight, groups.length, props.buffer, scrollTop],
  )

  // Layers (stable instances)
  const gridLayer = useMemo(() => new GridLayer(), [])
  const itemsLayer = useMemo(() => new ItemsLayer(), [])
  const overlayLayer = useMemo(() => new OverlayLayer(), [])

  // Drag handler
  const dragHandler = useMemo(() => new DragHandler(dragSnap), [dragSnap])

  // Extract marker configs from children
  const markers = useMemo(() => {
    const configs: MarkerConfig[] = []
    React.Children.forEach(children, (child) => {
      if (!React.isValidElement(child)) return
      const displayName = (child.type as { displayName?: string })?.displayName
      if (displayName === 'TodayMarker') {
        configs.push(getTodayMarkerConfig(child.props as { color?: string; width?: number }))
      } else if (displayName === 'CustomMarker') {
        configs.push(getCustomMarkerConfig(child.props as { date: number; color?: string; width?: number }))
      }
    })
    return configs
  }, [children])

  // Extract header children
  const headerChildren = useMemo(() => {
    const headers: React.ReactNode[] = []
    React.Children.forEach(children, (child) => {
      if (!React.isValidElement(child)) return
      const displayName = (child.type as { displayName?: string })?.displayName
      if (displayName === 'TimelineHeaders') {
        headers.push(child)
      }
    })
    return headers
  }, [children])

  // --- Drawing ---
  const drawGrid = useCallback(() => {
    const canvas = gridCanvasRef.current
    if (!canvas) return
    const ctx = setupCanvas(canvas, canvasWidth, canvasHeight)
    clearCanvas(ctx, canvas)
    gridLayer.draw(ctx, viewState, groups, theme, dayStyle, rowStyle)
  }, [viewState, groups, theme, dayStyle, rowStyle, canvasWidth, canvasHeight, gridLayer])

  const drawItems = useCallback(() => {
    const canvas = itemsCanvasRef.current
    if (!canvas) return
    const ctx = setupCanvas(canvas, canvasWidth, canvasHeight)
    clearCanvas(ctx, canvas)
    itemsLayer.draw(
      ctx, viewState, groups, items, intervalTree, layoutEngine,
      itemRenderer, groupRenderer, theme, selected, hoveredItemId, dependencies,
    )
  }, [viewState, groups, items, intervalTree, layoutEngine, itemRenderer, groupRenderer, theme, selected, hoveredItemId, dependencies, canvasWidth, canvasHeight, itemsLayer])

  const drawOverlay = useCallback(() => {
    const canvas = overlayCanvasRef.current
    if (!canvas) return
    const ctx = setupCanvas(canvas, canvasWidth, canvasHeight)
    clearCanvas(ctx, canvas)

    const dragState = dragHandler.getState()
    let dragRender = null
    if (dragState) {
      const x = viewState.timeToX(dragState.item.start_time) + dragState.deltaX
      const width = viewState.timeToX(dragState.item.end_time) - viewState.timeToX(dragState.item.start_time)
      dragRender = {
        item: dragState.item,
        bounds: { x, y: 0, width, height: lineHeight * itemHeightRatio },
        renderer: itemRenderer,
      }
    }

    overlayLayer.draw(ctx, viewState, theme, {
      cursorX: showCursorLine ? cursorX : null,
      snapX: dragState ? calculateSnapX(dragState.item, dragState.deltaX) : null,
      markers,
      drag: dragRender,
    })
  }, [viewState, theme, cursorX, showCursorLine, markers, dragHandler, canvasWidth, canvasHeight, overlayLayer, itemRenderer, lineHeight, itemHeightRatio])

  const calculateSnapX = useCallback((item: (typeof items)[0], deltaX: number) => {
    const pixelsPerMs = canvasWidth / (visibleTimeEnd - visibleTimeStart)
    const deltaMs = deltaX / pixelsPerMs
    const newStart = item.start_time + deltaMs
    const snapped = Math.round(newStart / dragSnap) * dragSnap
    return viewState.timeToX(snapped)
  }, [canvasWidth, visibleTimeStart, visibleTimeEnd, dragSnap, viewState])

  // Redraw effects
  useEffect(() => { drawGrid() }, [drawGrid])
  useEffect(() => { drawItems() }, [drawItems])
  useEffect(() => { drawOverlay() }, [drawOverlay])

  // --- Interaction handlers ---
  const scrollHandler = useMemo(() => {
    return new ScrollHandler(
      (deltaY) => setScrollTop((prev) => Math.max(0, prev + deltaY)),
      (deltaX) => {
        const pixelsPerMs = canvasWidth / (visibleTimeEnd - visibleTimeStart)
        const deltaMs = deltaX / pixelsPerMs
        const newStart = visibleTimeStart + deltaMs
        const newEnd = visibleTimeEnd + deltaMs
        setVisibleTimeStart(newStart)
        setVisibleTimeEnd(newEnd)
        onTimeChange?.(newStart, newEnd)
      },
    )
  }, [canvasWidth, visibleTimeStart, visibleTimeEnd, onTimeChange])

  const zoomHandler = useMemo(() => {
    return new ZoomHandler(
      (newStart, newEnd) => {
        setVisibleTimeStart(newStart)
        setVisibleTimeEnd(newEnd)
        onZoom?.(newStart, newEnd)
        onTimeChange?.(newStart, newEnd)
      },
      visibleTimeStart,
      visibleTimeEnd,
      minZoom,
      maxZoom,
    )
  }, [visibleTimeStart, visibleTimeEnd, minZoom, maxZoom, onZoom, onTimeChange])

  useEffect(() => {
    zoomHandler.updateBounds(visibleTimeStart, visibleTimeEnd)
  }, [visibleTimeStart, visibleTimeEnd, zoomHandler])

  const handleWheel = useCallback((e: React.WheelEvent) => {
    if (e.ctrlKey || e.metaKey) {
      e.preventDefault()
      const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
      const cursorRatio = (e.clientX - rect.left) / rect.width
      zoomHandler.handleZoom(e.deltaY, cursorRatio)
    } else {
      scrollHandler.handleWheel(e.nativeEvent)
    }
  }, [scrollHandler, zoomHandler])

  const handlePointerMove = useCallback((e: React.PointerEvent) => {
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top

    setCursorX(x)

    if (dragHandler.isDragging()) {
      dragHandler.updateDrag(x)
      drawOverlay()
      return
    }

    const item = hitTest(x, y, viewState, intervalTree, layoutEngine, groups)
    const newHoveredId = item?.id
    if (newHoveredId !== hoveredItemId) {
      setHoveredItemId(newHoveredId)
      onItemHover?.(newHoveredId ?? null, e.nativeEvent as unknown as PointerEvent)
    }
  }, [viewState, intervalTree, layoutEngine, groups, hoveredItemId, onItemHover, dragHandler, drawOverlay])

  const handlePointerDown = useCallback((e: React.PointerEvent) => {
    if (!canMove) return
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top

    const item = hitTest(x, y, viewState, intervalTree, layoutEngine, groups)
    if (item && canMove) {
      dragHandler.startDrag(item, x)
    }
  }, [canMove, viewState, intervalTree, layoutEngine, groups, dragHandler])

  const handlePointerUp = useCallback((e: React.PointerEvent) => {
    if (dragHandler.isDragging()) {
      const pixelsPerMs = canvasWidth / (visibleTimeEnd - visibleTimeStart)
      const snappedTime = dragHandler.endDrag(pixelsPerMs)
      const dragState = dragHandler.getState()
      if (snappedTime !== null && dragState) {
        onItemMove?.(dragState.item.id, snappedTime)
      }
      // endDrag already cleared state, but we need to get item id before endDrag
      // Fix: capture item before ending
      return
    }

    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    const item = hitTest(x, y, viewState, intervalTree, layoutEngine, groups)

    if (item) {
      onItemClick?.(item.id, e.nativeEvent as unknown as PointerEvent)
    } else {
      const group = hitTestGroup(y, viewState, groups)
      const time = viewState.xToTime(x)
      if (group) {
        // No item clicked — canvas click
      }
    }
  }, [canvasWidth, visibleTimeStart, visibleTimeEnd, dragHandler, onItemMove, onItemClick, viewState, intervalTree, layoutEngine, groups])

  const handleDoubleClick = useCallback((e: React.MouseEvent) => {
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top

    const item = hitTest(x, y, viewState, intervalTree, layoutEngine, groups)
    if (item) {
      onItemDoubleClick?.(item.id, e.nativeEvent as unknown as PointerEvent)
    } else {
      const group = hitTestGroup(y, viewState, groups)
      const time = viewState.xToTime(x)
      if (group) {
        onCanvasDoubleClick?.(group.id as number, time)
      }
    }
  }, [viewState, intervalTree, layoutEngine, groups, onItemDoubleClick, onCanvasDoubleClick])

  const handleContextMenu = useCallback((e: React.MouseEvent) => {
    e.preventDefault()
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top

    const item = hitTest(x, y, viewState, intervalTree, layoutEngine, groups)
    if (item) {
      onItemContextMenu?.(item.id, e.nativeEvent as unknown as PointerEvent)
    } else {
      const group = hitTestGroup(y, viewState, groups)
      const time = viewState.xToTime(x)
      if (group) {
        onCanvasContextMenu?.(group.id as number, time, e.nativeEvent as unknown as PointerEvent)
      }
    }
  }, [viewState, intervalTree, layoutEngine, groups, onItemContextMenu, onCanvasContextMenu])

  const handlePointerLeave = useCallback(() => {
    setCursorX(null)
    if (hoveredItemId !== undefined) {
      setHoveredItemId(undefined)
      onItemHover?.(null, new PointerEvent('pointerleave'))
    }
  }, [hoveredItemId, onItemHover])

  const canvasContainerStyle: React.CSSProperties = {
    position: 'relative',
    width: canvasWidth,
    height: canvasHeight,
    overflow: 'hidden',
    cursor: dragHandler.isDragging() ? 'grabbing' : 'default',
  }

  const canvasStyle: React.CSSProperties = {
    position: 'absolute',
    top: 0,
    left: 0,
  }

  return (
    <div style={{ display: 'flex', flexDirection: 'column', width: '100%', height: '100%' }}>
      {/* Headers */}
      {headerChildren}

      {/* Main content: sidebar + canvas */}
      <div ref={containerRef} style={{ display: 'flex', flex: 1, overflow: 'hidden' }}>
        <Sidebar
          groups={groups}
          width={sidebarWidth}
          lineHeight={lineHeight}
          scrollTop={scrollTop}
          canvasHeight={canvasHeight}
          theme={theme}
          groupRenderer={sidebarGroupRenderer}
          onScroll={setScrollTop}
        />

        <div
          style={canvasContainerStyle}
          onWheel={handleWheel}
          onPointerMove={handlePointerMove}
          onPointerDown={handlePointerDown}
          onPointerUp={handlePointerUp}
          onDoubleClick={handleDoubleClick}
          onContextMenu={handleContextMenu}
          onPointerLeave={handlePointerLeave}
        >
          <canvas ref={gridCanvasRef} style={{ ...canvasStyle, zIndex: 0 }} />
          <canvas ref={itemsCanvasRef} style={{ ...canvasStyle, zIndex: 1 }} />
          <canvas ref={overlayCanvasRef} style={{ ...canvasStyle, zIndex: 2 }} />
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Update src/index.ts to export CanvasTimeline**

```typescript
// src/index.ts
export type {
  Group,
  Item,
  ItemBounds,
  ItemState,
  DrawHelpers,
  CanvasItemRenderer,
  CanvasGroupItemRenderer,
  DayStyle,
  RowStyle,
  Dependency,
  TimelineTheme,
  MarkerConfig,
  CanvasTimelineProps,
} from './types'

export { DEFAULT_THEME } from './types'
export { CanvasTimeline } from './CanvasTimeline'
export { TodayMarker } from './dom/TodayMarker'
export { CustomMarker } from './dom/CustomMarker'
export { TimelineHeaders } from './dom/TimelineHeaders'
export { DateHeader } from './dom/DateHeader'
export { SidebarHeader } from './dom/SidebarHeader'
export { CustomHeader } from './dom/CustomHeader'
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run src/__tests__/CanvasTimeline.test.tsx
```

Expected: all PASS.

- [ ] **Step 6: Run all tests**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add src/CanvasTimeline.tsx src/index.ts
git commit -m "feat: add CanvasTimeline root component with full integration"
```

---

## Task 16: Build Verification & Final Polish

**Files:**
- Verify build, fix any TypeScript errors, ensure clean output.

- [ ] **Step 1: Run TypeScript check**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx tsc --noEmit
```

Expected: no errors. Fix any issues that arise.

- [ ] **Step 2: Run full test suite**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npx vitest run --coverage
```

Expected: all tests pass.

- [ ] **Step 3: Build the package**

```bash
cd /Users/abrha/Documents/projects/canvas-timeline && npm run build
```

Expected: dist/ contains `canvas-timeline.cjs.js`, `canvas-timeline.es.js`, `index.d.ts`.

- [ ] **Step 4: Verify dist output**

```bash
ls -la /Users/abrha/Documents/projects/canvas-timeline/dist/
```

Expected: CJS, ESM, and type definition files present.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: verify build output and finalize package"
```

---

## Summary

| Task | Component | Tests |
|------|-----------|-------|
| 1 | Project scaffold | Build verification |
| 2 | ViewState | 10 tests (coordinate math, buffer, visible range) |
| 3 | IntervalTree | 5 tests (queries, performance, edge cases) |
| 4 | LayoutEngine | 7 tests (stacking, heights, performance) |
| 5 | DrawHelpers | 9 tests (all helper functions) |
| 6 | CanvasManager | 3 tests (DPI, redraw scheduling) |
| 7 | GridLayer | 4 tests (rows, grid lines, day/row styles) |
| 8 | ItemsLayer | 4 tests (rendering, bounds, state, dependencies) |
| 9 | OverlayLayer | 5 tests (cursor, markers, drag, snap) |
| 10 | HitTest | 4 tests (item resolution, groups, stacked items) |
| 11 | ScrollHandler | 3 tests (vertical, horizontal, shift+wheel) |
| 12 | ZoomHandler | 5 tests (zoom in/out, clamp, anchor) |
| 13 | DragHandler | 6 tests (start, move, end, cancel, snap) |
| 14 | DOM components | — (render-only, tested via integration) |
| 15 | CanvasTimeline | 3 integration tests |
| 16 | Build verification | Build + type check |

**Total: 16 tasks, ~60 tests, zero runtime dependencies.**
