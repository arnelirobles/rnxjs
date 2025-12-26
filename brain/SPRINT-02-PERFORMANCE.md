# Sprint 2: Performance & Core Improvements

**Priority**: HIGH
**Duration**: 2-3 weeks
**Goal**: Match Vue/React performance characteristics for common use cases

---

## Overview

This sprint addresses performance bottlenecks that could cause issues in real-world applications. The goal is to ensure rnxJS performs competitively with Vue for lists up to 1,000 items.

---

## Task 2.1: Keyed List Diffing for data-for

### Current Problem

```javascript
// DataBinder.js - Current implementation
function renderList() {
  // PROBLEM: Destroys ALL DOM nodes and recreates them
  renderedElements.forEach(child => child.remove());
  renderedElements = [];

  array.forEach((item, index) => {
    const clone = template.cloneNode(true);
    // ... recreate everything
  });
}
```

**Impact**: O(n) DOM operations on every array change, even for single item updates.

### Solution: Keyed Reconciliation

```javascript
// framework/ListRenderer.js

/**
 * Efficient list rendering with keyed diffing
 * Similar to Vue's v-for with :key
 */
export class ListRenderer {
  constructor(template, placeholder, state, arrayPath, options = {}) {
    this.template = template;
    this.placeholder = placeholder;
    this.state = state;
    this.arrayPath = arrayPath;
    this.keyFn = options.key || ((item, index) => index);
    this.varName = options.varName;
    this.indexName = options.indexName;

    // Map of key -> { element, item, index }
    this.renderedMap = new Map();
    this.renderedOrder = []; // Keys in current order
  }

  /**
   * Render or update the list
   */
  render() {
    const array = this.getArray();
    if (!Array.isArray(array)) {
      console.warn(`[rnxJS] data-for: "${this.arrayPath}" is not an array`);
      this.clear();
      return;
    }

    const newKeys = [];
    const newMap = new Map();

    // Build new state
    array.forEach((item, index) => {
      const key = String(this.keyFn(item, index));
      newKeys.push(key);
      newMap.set(key, { item, index });
    });

    // Remove items that no longer exist
    for (const [key, rendered] of this.renderedMap) {
      if (!newMap.has(key)) {
        rendered.element.remove();
        this.renderedMap.delete(key);
      }
    }

    // Add or update items
    const parent = this.placeholder.parentNode;
    let prevElement = this.placeholder;

    for (let i = 0; i < newKeys.length; i++) {
      const key = newKeys[i];
      const { item, index } = newMap.get(key);

      let rendered = this.renderedMap.get(key);

      if (rendered) {
        // Update existing item if data changed
        if (rendered.item !== item || rendered.index !== index) {
          this.updateElement(rendered.element, item, index);
          rendered.item = item;
          rendered.index = index;
        }

        // Move if position changed
        if (rendered.element.previousSibling !== prevElement) {
          prevElement.after(rendered.element);
        }
      } else {
        // Create new item
        const element = this.createElement(item, index);
        prevElement.after(element);
        rendered = { element, item, index };
        this.renderedMap.set(key, rendered);
      }

      prevElement = rendered.element;
    }

    this.renderedOrder = newKeys;
  }

  /**
   * Create a new element from template
   */
  createElement(item, index) {
    const clone = this.template.cloneNode(true);
    this.updateElement(clone, item, index);
    return clone;
  }

  /**
   * Update element content with item data
   */
  updateElement(element, item, index) {
    // Update data-bind elements within
    const bindings = element.querySelectorAll('[data-bind]');
    bindings.forEach(el => {
      const path = el.getAttribute('data-bind');
      const value = this.resolveBinding(path, item, index);
      if (value !== undefined) {
        el.textContent = value;
      }
    });

    // Update element itself if it has data-bind
    const selfPath = element.getAttribute('data-bind');
    if (selfPath) {
      const value = this.resolveBinding(selfPath, item, index);
      if (value !== undefined && typeof item !== 'object') {
        element.textContent = value;
      }
    }
  }

  /**
   * Resolve a binding path for list item context
   */
  resolveBinding(path, item, index) {
    if (path === this.varName) {
      return typeof item === 'object' ? JSON.stringify(item) : item;
    }
    if (path === this.indexName) {
      return String(index);
    }
    if (path.startsWith(this.varName + '.')) {
      const subPath = path.slice(this.varName.length + 1);
      return this.getNestedValue(item, subPath);
    }
    return undefined;
  }

  /**
   * Get nested value from object
   */
  getNestedValue(obj, path) {
    return path.split('.').reduce((curr, key) => curr?.[key], obj);
  }

  /**
   * Get array from state
   */
  getArray() {
    return this.arrayPath.split('.').reduce(
      (curr, key) => curr?.[key],
      this.state
    );
  }

  /**
   * Clear all rendered elements
   */
  clear() {
    for (const [, rendered] of this.renderedMap) {
      rendered.element.remove();
    }
    this.renderedMap.clear();
    this.renderedOrder = [];
  }

  /**
   * Destroy the renderer
   */
  destroy() {
    this.clear();
    this.placeholder.remove();
  }
}
```

### Add data-key Attribute

```html
<!-- Usage with explicit key -->
<li data-for="user in users" data-key="user.id" data-bind="user.name"></li>

<!-- Key function examples -->
<tr data-for="item in items" data-key="item.id">
  <td data-bind="item.name"></td>
  <td data-bind="item.price"></td>
</tr>
```

### Benchmark Test

```javascript
// tests/performance/list-rendering.bench.js

import { bench, describe } from 'vitest';

describe('List Rendering Performance', () => {
  const createItems = (count) =>
    Array.from({ length: count }, (_, i) => ({
      id: i,
      name: `Item ${i}`,
      value: Math.random()
    }));

  bench('render 100 items', async () => {
    const state = createReactiveState({ items: createItems(100) });
    // ... render
  });

  bench('render 1000 items', async () => {
    const state = createReactiveState({ items: createItems(1000) });
    // ... render
  });

  bench('update single item in 1000', async () => {
    const state = createReactiveState({ items: createItems(1000) });
    // Initial render
    // Update one item
    state.items[500].name = 'Updated';
    // Measure re-render time
  });

  bench('prepend item to 1000', async () => {
    const state = createReactiveState({ items: createItems(1000) });
    // Initial render
    state.items.unshift({ id: -1, name: 'New First' });
    // Measure re-render time
  });
});
```

### Target Performance

| Operation | Current | Target | Vue 3 |
|-----------|---------|--------|-------|
| Render 1000 items | 50ms | 20ms | 15ms |
| Update 1 of 1000 | 50ms | 1ms | 0.5ms |
| Prepend to 1000 | 50ms | 5ms | 2ms |
| Delete from 1000 | 50ms | 2ms | 1ms |

---

## Task 2.2: Update Batching

### Current Problem

```javascript
// Each assignment triggers immediate update
state.user.name = 'Alice';  // Update 1
state.user.email = 'a@b.c'; // Update 2
state.user.age = 30;        // Update 3
// 3 separate DOM updates!
```

### Solution: Microtask Batching

```javascript
// utils/createReactiveState.js - Add batching

// Batch queue
let pendingNotifications = new Map();
let batchScheduled = false;

/**
 * Queue a notification for batched execution
 */
function queueNotification(path, value, notify) {
  pendingNotifications.set(path, { value, notify });

  if (!batchScheduled) {
    batchScheduled = true;
    queueMicrotask(flushNotifications);
  }
}

/**
 * Flush all pending notifications
 */
function flushNotifications() {
  const notifications = pendingNotifications;
  pendingNotifications = new Map();
  batchScheduled = false;

  for (const [path, { value, notify }] of notifications) {
    notify(path, value);
  }
}

/**
 * Force synchronous flush (for testing)
 */
export function flushSync() {
  if (batchScheduled) {
    flushNotifications();
  }
}

// In the Proxy handler:
set(obj, prop, value) {
  const oldValue = obj[prop];
  const currentPath = basePath ? `${basePath}.${prop}` : prop;

  if (oldValue !== value) {
    obj[prop] = value;
    // Queue instead of immediate notify
    queueNotification(currentPath, value, notify);
  }

  return true;
}
```

### Usage

```javascript
// Automatic batching
state.user.name = 'Alice';
state.user.email = 'alice@example.com';
state.user.age = 30;
// Only 1 DOM update cycle!

// Force sync when needed
import { flushSync } from 'rnxjs';

state.value = 'new';
flushSync(); // Immediately process all pending updates
console.log(document.querySelector('[data-bind="value"]').textContent); // 'new'
```

---

## Task 2.3: Computed Property Optimization

### Current Implementation

The current implementation is good but can be improved:

```javascript
// Current: Recomputes on any dependency change
get() {
  const cached = computedCache.get(name);
  if (cached.dirty) {
    cached.value = getter(state);
    cached.dirty = false;
  }
  return cached.value;
}
```

### Improvements

```javascript
/**
 * Enhanced computed properties with:
 * - Lazy evaluation
 * - Automatic dependency tracking
 * - Subscription support
 * - Equality checking to prevent unnecessary updates
 */
function createComputed(name, getter, state, notify) {
  let cachedValue;
  let isDirty = true;
  let dependencies = new Set();
  let isComputing = false;

  // Track which paths are accessed during computation
  const trackDependency = (path) => {
    if (isComputing) {
      dependencies.add(path);
    }
  };

  // Compute the value
  const compute = () => {
    if (!isDirty) return cachedValue;

    isComputing = true;
    dependencies.clear();

    try {
      const newValue = getter(state);

      // Only notify if value actually changed
      if (!isComputing && !shallowEqual(cachedValue, newValue)) {
        notify(name, newValue);
      }

      cachedValue = newValue;
      isDirty = false;
    } finally {
      isComputing = false;
    }

    return cachedValue;
  };

  // Mark as dirty when dependencies change
  const invalidate = () => {
    if (!isDirty) {
      isDirty = true;
      // Notify subscribers that computed may have changed
      queueMicrotask(() => {
        compute(); // Recompute and notify if changed
      });
    }
  };

  return {
    get: compute,
    invalidate,
    getDependencies: () => new Set(dependencies),
    trackDependency
  };
}

/**
 * Shallow equality check for computed values
 */
function shallowEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== typeof b) return false;
  if (typeof a !== 'object' || a === null) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) return false;

  for (const key of keysA) {
    if (a[key] !== b[key]) return false;
  }

  return true;
}
```

---

## Task 2.4: Memory Leak Prevention

### Add WeakRef for Subscriptions

```javascript
// utils/createReactiveState.js

/**
 * Enhanced subscription with automatic cleanup
 */
function createSubscriptionManager() {
  const subscriptions = new Map();
  const cleanupInterval = 30000; // 30 seconds

  // Periodic cleanup of dead references
  let cleanupTimer = null;

  const startCleanup = () => {
    if (cleanupTimer) return;
    cleanupTimer = setInterval(() => {
      for (const [path, callbacks] of subscriptions) {
        for (const ref of callbacks) {
          if (ref.deref() === undefined) {
            callbacks.delete(ref);
          }
        }
        if (callbacks.size === 0) {
          subscriptions.delete(path);
        }
      }
    }, cleanupInterval);
  };

  const stopCleanup = () => {
    if (cleanupTimer) {
      clearInterval(cleanupTimer);
      cleanupTimer = null;
    }
  };

  return {
    subscribe(path, callback) {
      if (!subscriptions.has(path)) {
        subscriptions.set(path, new Set());
      }

      // Use WeakRef if available (modern browsers)
      const ref = typeof WeakRef !== 'undefined'
        ? new WeakRef(callback)
        : { deref: () => callback }; // Fallback

      subscriptions.get(path).add(ref);
      startCleanup();

      return () => {
        const callbacks = subscriptions.get(path);
        if (callbacks) {
          callbacks.delete(ref);
          if (callbacks.size === 0) {
            subscriptions.delete(path);
          }
        }
      };
    },

    notify(path, value) {
      const callbacks = subscriptions.get(path);
      if (!callbacks) return;

      for (const ref of callbacks) {
        const callback = ref.deref();
        if (callback) {
          try {
            callback(value);
          } catch (error) {
            console.error(`[rnxJS] Subscription error:`, error);
          }
        }
      }
    },

    destroy() {
      stopCleanup();
      subscriptions.clear();
    }
  };
}
```

### Add Component Disposal

```javascript
// utils/createComponent.js

export function createComponent(options) {
  // ... existing code ...

  // Track all cleanup functions
  const cleanupFns = new Set();

  const component = {
    // ... existing methods ...

    /**
     * Register a cleanup function
     */
    onCleanup(fn) {
      cleanupFns.add(fn);
      return () => cleanupFns.delete(fn);
    },

    /**
     * Destroy component and clean up all resources
     */
    destroy() {
      // Run all cleanup functions
      for (const fn of cleanupFns) {
        try {
          fn();
        } catch (error) {
          console.error('[rnxJS] Cleanup error:', error);
        }
      }
      cleanupFns.clear();

      // Remove from DOM
      if (element && element.parentNode) {
        element.parentNode.removeChild(element);
      }

      // Clear references
      element = null;
      state = null;
    }
  };

  // Auto-cleanup on DOM removal (if MutationObserver available)
  if (typeof MutationObserver !== 'undefined') {
    const observer = new MutationObserver((mutations) => {
      for (const mutation of mutations) {
        for (const node of mutation.removedNodes) {
          if (node === element || node.contains?.(element)) {
            component.destroy();
            observer.disconnect();
            return;
          }
        }
      }
    });

    // Observe parent for removals
    requestAnimationFrame(() => {
      if (element?.parentNode) {
        observer.observe(element.parentNode, { childList: true });
        cleanupFns.add(() => observer.disconnect());
      }
    });
  }

  return component;
}
```

---

## Task 2.5: Virtual Scrolling (Optional)

For very large lists (1000+ items), add virtual scrolling:

```javascript
// components/VirtualList.js

/**
 * Virtual scrolling list for large datasets
 *
 * Usage:
 * <VirtualList
 *   items="state.items"
 *   item-height="40"
 *   visible-count="20"
 *   template="item-template"
 * />
 */
export function VirtualList({
  items,
  itemHeight = 40,
  visibleCount = 20,
  template,
  state
}) {
  const containerHeight = visibleCount * itemHeight;
  const totalHeight = items.length * itemHeight;

  let scrollTop = 0;
  let startIndex = 0;

  const container = document.createElement('div');
  container.className = 'rnx-virtual-list';
  container.style.cssText = `
    height: ${containerHeight}px;
    overflow-y: auto;
    position: relative;
  `;

  const content = document.createElement('div');
  content.style.cssText = `
    height: ${totalHeight}px;
    position: relative;
  `;
  container.appendChild(content);

  const renderVisibleItems = () => {
    startIndex = Math.floor(scrollTop / itemHeight);
    const endIndex = Math.min(startIndex + visibleCount + 2, items.length);

    // Clear and re-render visible items
    content.innerHTML = '';

    for (let i = startIndex; i < endIndex; i++) {
      const item = items[i];
      const element = createItemElement(item, i, template);
      element.style.cssText = `
        position: absolute;
        top: ${i * itemHeight}px;
        height: ${itemHeight}px;
        width: 100%;
      `;
      content.appendChild(element);
    }
  };

  container.addEventListener('scroll', () => {
    const newScrollTop = container.scrollTop;
    if (Math.abs(newScrollTop - scrollTop) > itemHeight) {
      scrollTop = newScrollTop;
      renderVisibleItems();
    }
  });

  renderVisibleItems();

  return container;
}
```

---

## Task 2.6: Performance Monitoring

### Add Performance Hooks

```javascript
// utils/performance.js

/**
 * Performance monitoring for rnxJS
 */
export const rnxPerf = {
  enabled: false,
  marks: new Map(),
  measures: [],

  enable() {
    this.enabled = true;
  },

  disable() {
    this.enabled = false;
  },

  mark(name) {
    if (!this.enabled) return;
    this.marks.set(name, performance.now());
  },

  measure(name, startMark, endMark) {
    if (!this.enabled) return;

    const start = this.marks.get(startMark);
    const end = endMark ? this.marks.get(endMark) : performance.now();

    if (start !== undefined) {
      const duration = end - start;
      this.measures.push({ name, duration, timestamp: Date.now() });

      if (duration > 16) { // Longer than a frame
        console.warn(`[rnxJS Perf] Slow operation: ${name} took ${duration.toFixed(2)}ms`);
      }
    }
  },

  getReport() {
    const report = {};
    for (const { name, duration } of this.measures) {
      if (!report[name]) {
        report[name] = { count: 0, total: 0, min: Infinity, max: 0 };
      }
      report[name].count++;
      report[name].total += duration;
      report[name].min = Math.min(report[name].min, duration);
      report[name].max = Math.max(report[name].max, duration);
    }

    for (const name in report) {
      report[name].avg = report[name].total / report[name].count;
    }

    return report;
  },

  clear() {
    this.marks.clear();
    this.measures = [];
  }
};

// Integration with core functions
export function withPerf(name, fn) {
  return function(...args) {
    rnxPerf.mark(`${name}:start`);
    const result = fn.apply(this, args);
    rnxPerf.measure(name, `${name}:start`);
    return result;
  };
}
```

### Usage

```javascript
// Enable performance monitoring in development
import { rnxPerf } from 'rnxjs';

if (process.env.NODE_ENV === 'development') {
  rnxPerf.enable();
}

// After some operations
console.table(rnxPerf.getReport());
```

---

## Acceptance Criteria

### Must Have

- [ ] Keyed list diffing implemented
- [ ] `data-key` attribute supported
- [ ] Update batching via microtasks
- [ ] `flushSync()` utility added
- [ ] Memory leak prevention in subscriptions
- [ ] Component auto-cleanup on DOM removal

### Should Have

- [ ] Performance benchmark suite
- [ ] Performance monitoring utilities
- [ ] Computed property equality checking

### Nice to Have

- [ ] Virtual scrolling component
- [ ] Performance DevTools integration

---

## Performance Targets

| Metric | Current | Sprint Target | Vue 3 Baseline |
|--------|---------|---------------|----------------|
| Update 1 of 1000 items | 50ms | <5ms | 0.5ms |
| Bundle size | 33KB | <40KB | 42KB |
| Memory per 1000 items | Unknown | <1MB | ~1MB |
| Time to first paint | Fast | <100ms | ~100ms |

---

## Definition of Done

1. All performance benchmarks passing targets
2. No memory leaks in 1-hour stress test
3. Bundle size remains under 40KB
4. All existing tests still passing
5. Documentation updated with new features
6. Changelog updated
