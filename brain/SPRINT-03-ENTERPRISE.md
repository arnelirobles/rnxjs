# Sprint 3: Enterprise Readiness

**Priority**: HIGH
**Duration**: 3-4 weeks
**Goal**: Remove enterprise adoption blockers

---

## Overview

This sprint addresses the features and capabilities that enterprise organizations require before adopting a new framework. These are non-negotiable for companies with compliance, accessibility, and internationalization requirements.

---

## Task 3.1: Full TypeScript Support

### Current State

- TypeScript declarations exist (`index.d.ts`)
- Source code is JavaScript
- No strict type checking

### Target State

- Full TypeScript source code
- Strict type checking
- Generic type support for state
- IntelliSense in all IDEs

### Implementation Plan

#### Phase 1: Convert Core to TypeScript

```typescript
// utils/createReactiveState.ts

/**
 * Reactive state configuration options
 */
export interface ReactiveStateOptions<T> {
  /** Initial state object */
  initialState: T;
  /** Computed property definitions */
  computed?: ComputedDefinitions<T>;
  /** Enable debug logging */
  debug?: boolean;
}

/**
 * Computed property definitions
 */
export type ComputedDefinitions<T> = {
  [K: string]: (state: T) => unknown;
};

/**
 * Subscription callback type
 */
export type SubscriptionCallback<V = unknown> = (value: V) => void;

/**
 * Unsubscribe function type
 */
export type Unsubscribe = () => void;

/**
 * Reactive state proxy interface
 */
export interface ReactiveState<T> extends T {
  /**
   * Subscribe to changes on a specific path
   * @param path - Dot-notation path (e.g., 'user.email')
   * @param callback - Called when value changes
   * @returns Unsubscribe function
   */
  subscribe<V = unknown>(path: string, callback: SubscriptionCallback<V>): Unsubscribe;

  /**
   * Get nested value by path
   * @param path - Dot-notation path
   * @returns Value at path or undefined
   */
  getNestedValue<V = unknown>(path: string): V | undefined;

  /**
   * Unsubscribe all listeners
   */
  $unsubscribeAll(): void;

  /**
   * Destroy state and cleanup resources
   */
  $destroy(): void;
}

/**
 * Create a reactive state object with automatic change detection
 *
 * @template T - State object type
 * @param initialState - Initial state values
 * @returns Reactive state proxy
 *
 * @example
 * interface AppState {
 *   user: { name: string; email: string };
 *   items: string[];
 * }
 *
 * const state = createReactiveState<AppState>({
 *   user: { name: '', email: '' },
 *   items: []
 * });
 *
 * state.subscribe('user.name', (name) => {
 *   console.log('Name changed:', name);
 * });
 *
 * state.user.name = 'Alice'; // Triggers callback
 */
export function createReactiveState<T extends object>(
  initialState: T
): ReactiveState<T> {
  // Implementation...
}
```

#### Phase 2: Typed Components

```typescript
// utils/createComponent.ts

/**
 * Component props base type
 */
export interface ComponentProps {
  children?: Node[];
  [key: string]: unknown;
}

/**
 * Component lifecycle hooks
 */
export interface ComponentLifecycle {
  /** Called after render */
  useEffect(effect: () => void | (() => void)): void;
  /** Called on unmount */
  onUnmount(cleanup: () => void): void;
}

/**
 * Component state management
 */
export interface ComponentState<S> {
  /** Get current state */
  getState(): S;
  /** Update state (partial merge) */
  setState(updates: Partial<S>): void;
  /** Hook-style state */
  useState<V>(initial: V): [() => V, (value: V) => void];
}

/**
 * Component refs
 */
export interface ComponentRefs {
  [key: string]: HTMLElement | null;
}

/**
 * Component options
 */
export interface ComponentOptions<P extends ComponentProps, S = {}> {
  /** Initial state */
  state?: S;
  /** Template function */
  template: (props: P, state: S) => string;
  /** Setup function with lifecycle access */
  setup?: (
    props: P,
    context: ComponentLifecycle & ComponentState<S>
  ) => void;
}

/**
 * Component instance
 */
export interface ComponentInstance {
  /** Root element */
  element: HTMLElement;
  /** Element refs */
  refs: ComponentRefs;
  /** Re-render component */
  render(): void;
  /** Destroy and cleanup */
  destroy(): void;
}

/**
 * Create a component with state and lifecycle
 *
 * @example
 * interface CounterProps { initial?: number }
 * interface CounterState { count: number }
 *
 * const Counter = createComponent<CounterProps, CounterState>({
 *   state: { count: 0 },
 *   template: (props, state) => `
 *     <div class="counter">
 *       <span data-ref="display">${state.count}</span>
 *       <button data-ref="increment">+</button>
 *     </div>
 *   `,
 *   setup(props, { refs, setState, useEffect }) {
 *     refs.increment?.addEventListener('click', () => {
 *       setState({ count: state.count + 1 });
 *     });
 *   }
 * });
 */
export function createComponent<P extends ComponentProps, S = {}>(
  options: ComponentOptions<P, S>
): (props: P) => ComponentInstance;
```

#### Phase 3: Typed Data Binding

```typescript
// framework/DataBinder.ts

/**
 * Data binding options
 */
export interface BindDataOptions {
  /** Validation rules map */
  validationRules?: Record<string, string>;
  /** Custom validators */
  validators?: Record<string, ValidatorFn>;
  /** Debounce input events (ms) */
  debounce?: number;
}

/**
 * Validator function type
 */
export type ValidatorFn = (
  value: unknown,
  param?: string
) => string | null;

/**
 * Bind data attributes to reactive state
 *
 * @param rootElement - Root element to search for data-bind
 * @param state - Reactive state object
 * @param options - Binding options
 *
 * @example
 * const state = createReactiveState({
 *   email: '',
 *   errors: {}
 * });
 *
 * bindData(document.getElementById('form'), state, {
 *   validators: {
 *     unique: async (value) => {
 *       const exists = await checkEmail(value);
 *       return exists ? 'Email already taken' : null;
 *     }
 *   }
 * });
 */
export function bindData<T extends object>(
  rootElement: Element | Document,
  state: ReactiveState<T>,
  options?: BindDataOptions
): void;
```

### Build Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

---

## Task 3.2: Accessibility (WCAG 2.1 AA)

### Audit Current Components

| Component | Keyboard | Screen Reader | Focus Mgmt | ARIA |
|-----------|----------|---------------|------------|------|
| Button | ✅ | ⚠️ | ✅ | ⚠️ |
| Modal | ⚠️ | ❌ | ❌ | ❌ |
| Tabs | ❌ | ❌ | ❌ | ❌ |
| Accordion | ❌ | ❌ | ❌ | ❌ |
| Select | ✅ (native) | ✅ | ✅ | ✅ |
| Input | ✅ | ⚠️ | ✅ | ⚠️ |

### Implementation: Modal Accessibility

```javascript
// components/bootstrap/Modal.js

export function Modal({
  title,
  children,
  isOpen = false,
  onClose,
  size = 'md',
  closeOnEscape = true,
  closeOnBackdrop = true,
  initialFocus = null, // Selector for initial focus
  returnFocus = true,  // Return focus on close
}) {
  // Store the element that had focus before modal opened
  let previousActiveElement = null;

  // Create modal structure with proper ARIA
  const modal = document.createElement('div');
  modal.className = `modal fade ${isOpen ? 'show' : ''}`;
  modal.setAttribute('role', 'dialog');
  modal.setAttribute('aria-modal', 'true');
  modal.setAttribute('aria-labelledby', 'modal-title');
  modal.setAttribute('tabindex', '-1');

  modal.innerHTML = `
    <div class="modal-dialog modal-${size}" role="document">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title" id="modal-title">${escapeHtml(title)}</h5>
          <button
            type="button"
            class="btn-close"
            aria-label="Close"
            data-ref="closeBtn"
          ></button>
        </div>
        <div class="modal-body" data-slot="body"></div>
        <div class="modal-footer" data-slot="footer"></div>
      </div>
    </div>
  `;

  // Focus trap - keep focus within modal
  const focusableSelectors = [
    'button:not([disabled])',
    'input:not([disabled])',
    'select:not([disabled])',
    'textarea:not([disabled])',
    'a[href]',
    '[tabindex]:not([tabindex="-1"])'
  ].join(',');

  function getFocusableElements() {
    return modal.querySelectorAll(focusableSelectors);
  }

  function trapFocus(e) {
    const focusable = getFocusableElements();
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.key === 'Tab') {
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }
  }

  function handleKeydown(e) {
    if (e.key === 'Escape' && closeOnEscape) {
      close();
    }
    trapFocus(e);
  }

  function open() {
    previousActiveElement = document.activeElement;
    modal.classList.add('show');
    modal.style.display = 'block';
    document.body.classList.add('modal-open');

    // Set initial focus
    requestAnimationFrame(() => {
      if (initialFocus) {
        const target = modal.querySelector(initialFocus);
        target?.focus();
      } else {
        const focusable = getFocusableElements();
        focusable[0]?.focus();
      }
    });

    modal.addEventListener('keydown', handleKeydown);

    // Announce to screen readers
    modal.setAttribute('aria-hidden', 'false');
  }

  function close() {
    modal.classList.remove('show');
    modal.style.display = 'none';
    document.body.classList.remove('modal-open');
    modal.removeEventListener('keydown', handleKeydown);
    modal.setAttribute('aria-hidden', 'true');

    if (returnFocus && previousActiveElement) {
      previousActiveElement.focus();
    }

    if (typeof onClose === 'function') {
      onClose();
    }
  }

  // Close on backdrop click
  if (closeOnBackdrop) {
    modal.addEventListener('click', (e) => {
      if (e.target === modal) {
        close();
      }
    });
  }

  // Close button
  modal.querySelector('[data-ref="closeBtn"]')
    .addEventListener('click', close);

  // Public API
  modal.open = open;
  modal.close = close;

  if (isOpen) {
    requestAnimationFrame(open);
  }

  return modal;
}
```

### Implementation: Accessible Tabs

```javascript
// components/bootstrap/Tabs.js

export function Tabs({
  tabs, // [{ id, label, content }]
  activeTab = 0,
  onChange,
  orientation = 'horizontal', // 'horizontal' | 'vertical'
}) {
  let currentTab = activeTab;

  const container = document.createElement('div');
  container.className = 'rnx-tabs';

  // Tab list with proper ARIA
  const tabList = document.createElement('div');
  tabList.className = `nav nav-tabs ${orientation === 'vertical' ? 'flex-column' : ''}`;
  tabList.setAttribute('role', 'tablist');
  tabList.setAttribute('aria-orientation', orientation);

  // Tab panels container
  const panelContainer = document.createElement('div');
  panelContainer.className = 'tab-content';

  // Create tabs and panels
  tabs.forEach((tab, index) => {
    // Tab button
    const button = document.createElement('button');
    button.className = `nav-link ${index === currentTab ? 'active' : ''}`;
    button.setAttribute('role', 'tab');
    button.setAttribute('id', `tab-${tab.id}`);
    button.setAttribute('aria-controls', `panel-${tab.id}`);
    button.setAttribute('aria-selected', index === currentTab ? 'true' : 'false');
    button.setAttribute('tabindex', index === currentTab ? '0' : '-1');
    button.textContent = tab.label;
    tabList.appendChild(button);

    // Tab panel
    const panel = document.createElement('div');
    panel.className = `tab-pane ${index === currentTab ? 'show active' : ''}`;
    panel.setAttribute('role', 'tabpanel');
    panel.setAttribute('id', `panel-${tab.id}`);
    panel.setAttribute('aria-labelledby', `tab-${tab.id}`);
    panel.setAttribute('tabindex', '0');
    panel.innerHTML = tab.content;
    panelContainer.appendChild(panel);
  });

  // Keyboard navigation
  tabList.addEventListener('keydown', (e) => {
    const tabButtons = tabList.querySelectorAll('[role="tab"]');
    const currentIndex = Array.from(tabButtons).indexOf(document.activeElement);

    let newIndex = currentIndex;

    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowDown':
        newIndex = (currentIndex + 1) % tabButtons.length;
        break;
      case 'ArrowLeft':
      case 'ArrowUp':
        newIndex = (currentIndex - 1 + tabButtons.length) % tabButtons.length;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = tabButtons.length - 1;
        break;
      default:
        return;
    }

    e.preventDefault();
    selectTab(newIndex);
    tabButtons[newIndex].focus();
  });

  // Tab selection
  function selectTab(index) {
    const tabButtons = tabList.querySelectorAll('[role="tab"]');
    const panels = panelContainer.querySelectorAll('[role="tabpanel"]');

    // Update previous tab
    tabButtons[currentTab].classList.remove('active');
    tabButtons[currentTab].setAttribute('aria-selected', 'false');
    tabButtons[currentTab].setAttribute('tabindex', '-1');
    panels[currentTab].classList.remove('show', 'active');

    // Update new tab
    currentTab = index;
    tabButtons[currentTab].classList.add('active');
    tabButtons[currentTab].setAttribute('aria-selected', 'true');
    tabButtons[currentTab].setAttribute('tabindex', '0');
    panels[currentTab].classList.add('show', 'active');

    if (typeof onChange === 'function') {
      onChange(index, tabs[index]);
    }
  }

  // Click handler
  tabList.addEventListener('click', (e) => {
    const button = e.target.closest('[role="tab"]');
    if (button) {
      const index = Array.from(tabList.querySelectorAll('[role="tab"]')).indexOf(button);
      selectTab(index);
    }
  });

  container.appendChild(tabList);
  container.appendChild(panelContainer);

  return container;
}
```

### Accessibility Testing Utilities

```javascript
// utils/a11y.js

/**
 * Check if an element is focusable
 */
export function isFocusable(element) {
  if (element.disabled) return false;
  if (element.tabIndex < 0) return false;

  const focusableTags = ['A', 'BUTTON', 'INPUT', 'SELECT', 'TEXTAREA'];
  if (focusableTags.includes(element.tagName)) return true;

  return element.hasAttribute('tabindex');
}

/**
 * Get all focusable elements within a container
 */
export function getFocusableElements(container) {
  const elements = container.querySelectorAll(
    'a[href], button, input, select, textarea, [tabindex]'
  );
  return Array.from(elements).filter(isFocusable);
}

/**
 * Announce message to screen readers
 */
export function announce(message, priority = 'polite') {
  const announcer = document.createElement('div');
  announcer.setAttribute('role', 'status');
  announcer.setAttribute('aria-live', priority);
  announcer.setAttribute('aria-atomic', 'true');
  announcer.className = 'sr-only';
  announcer.textContent = message;

  document.body.appendChild(announcer);

  setTimeout(() => {
    document.body.removeChild(announcer);
  }, 1000);
}

/**
 * Create a skip link for keyboard navigation
 */
export function createSkipLink(targetId, text = 'Skip to main content') {
  const link = document.createElement('a');
  link.href = `#${targetId}`;
  link.className = 'rnx-skip-link sr-only sr-only-focusable';
  link.textContent = text;
  return link;
}
```

---

## Task 3.3: Internationalization (i18n)

### i18n Core Implementation

```javascript
// utils/i18n.js

/**
 * Internationalization system for rnxJS
 */
class I18n {
  constructor() {
    this.locale = 'en';
    this.fallbackLocale = 'en';
    this.messages = {};
    this.formatters = {};
    this.subscribers = new Set();
  }

  /**
   * Set current locale
   */
  setLocale(locale) {
    this.locale = locale;
    this.notifySubscribers();
    document.documentElement.lang = locale;
  }

  /**
   * Get current locale
   */
  getLocale() {
    return this.locale;
  }

  /**
   * Load messages for a locale
   */
  async loadMessages(locale, messages) {
    if (typeof messages === 'function') {
      // Lazy loading
      messages = await messages();
    }
    this.messages[locale] = { ...this.messages[locale], ...messages };
  }

  /**
   * Translate a key
   *
   * @param key - Translation key (dot notation: 'common.buttons.save')
   * @param params - Interpolation parameters
   * @param count - Pluralization count
   */
  t(key, params = {}, count = null) {
    const message = this.getMessage(key);

    if (!message) {
      console.warn(`[rnxJS i18n] Missing translation: ${key}`);
      return key;
    }

    // Handle pluralization
    let text = message;
    if (count !== null && typeof message === 'object') {
      text = this.pluralize(message, count);
    }

    // Interpolate parameters
    return this.interpolate(text, params);
  }

  /**
   * Get message from messages object
   */
  getMessage(key) {
    const locales = [this.locale, this.fallbackLocale];

    for (const locale of locales) {
      const messages = this.messages[locale];
      if (!messages) continue;

      const value = key.split('.').reduce(
        (obj, k) => obj?.[k],
        messages
      );

      if (value !== undefined) return value;
    }

    return null;
  }

  /**
   * Handle pluralization
   */
  pluralize(messages, count) {
    // Support: { zero, one, few, many, other }
    const rules = new Intl.PluralRules(this.locale);
    const rule = rules.select(count);

    return messages[rule] || messages.other || messages;
  }

  /**
   * Interpolate parameters into message
   */
  interpolate(message, params) {
    return message.replace(/\{(\w+)\}/g, (match, key) => {
      if (key in params) {
        return this.format(params[key], params[`${key}Type`]);
      }
      return match;
    });
  }

  /**
   * Format a value based on type
   */
  format(value, type) {
    if (type === 'number') {
      return new Intl.NumberFormat(this.locale).format(value);
    }
    if (type === 'currency') {
      return new Intl.NumberFormat(this.locale, {
        style: 'currency',
        currency: this.currency || 'USD'
      }).format(value);
    }
    if (type === 'date') {
      return new Intl.DateTimeFormat(this.locale).format(new Date(value));
    }
    if (type === 'relative') {
      return new Intl.RelativeTimeFormat(this.locale).format(value, 'day');
    }
    return value;
  }

  /**
   * Subscribe to locale changes
   */
  subscribe(callback) {
    this.subscribers.add(callback);
    return () => this.subscribers.delete(callback);
  }

  /**
   * Notify subscribers of locale change
   */
  notifySubscribers() {
    for (const callback of this.subscribers) {
      callback(this.locale);
    }
  }
}

// Singleton instance
export const i18n = new I18n();

// Convenience function
export function t(key, params, count) {
  return i18n.t(key, params, count);
}

// React to locale changes in data binding
export function setupI18nBinding(state) {
  i18n.subscribe(() => {
    // Trigger re-render of i18n-bound elements
    if (state && typeof state.subscribe === 'function') {
      state._i18nVersion = (state._i18nVersion || 0) + 1;
    }
  });
}
```

### i18n Data Binding

```html
<!-- Usage in templates -->
<h1 data-i18n="pages.home.title"></h1>
<p data-i18n="pages.home.welcome" data-i18n-params='{"name": "user.name"}'></p>
<span data-i18n="items.count" data-i18n-count="items.length"></span>

<!-- With formatted values -->
<span data-i18n="order.total" data-i18n-params='{"amount": "order.total", "amountType": "currency"}'></span>
```

```javascript
// framework/i18nBinder.js

/**
 * Process i18n bindings in the DOM
 */
export function bindI18n(root, state) {
  const elements = root.querySelectorAll('[data-i18n]');

  elements.forEach(el => {
    const key = el.getAttribute('data-i18n');
    const paramsAttr = el.getAttribute('data-i18n-params');
    const countPath = el.getAttribute('data-i18n-count');

    const update = () => {
      let params = {};

      // Resolve params from state
      if (paramsAttr) {
        try {
          const paramDefs = JSON.parse(paramsAttr);
          for (const [paramKey, statePath] of Object.entries(paramDefs)) {
            params[paramKey] = getNestedValue(state, statePath);
          }
        } catch (e) {
          console.warn('[rnxJS i18n] Invalid params:', paramsAttr);
        }
      }

      // Get count for pluralization
      const count = countPath ? getNestedValue(state, countPath) : null;

      // Translate and set
      el.textContent = t(key, params, count);
    };

    // Initial render
    update();

    // Subscribe to locale changes
    i18n.subscribe(update);

    // Subscribe to state changes for dynamic params
    if (paramsAttr || countPath) {
      // Watch relevant state paths
      state.subscribe('_i18nVersion', update);
    }
  });
}
```

### Message Files Structure

```javascript
// locales/en.js
export default {
  common: {
    buttons: {
      save: 'Save',
      cancel: 'Cancel',
      delete: 'Delete',
      confirm: 'Confirm'
    },
    validation: {
      required: 'This field is required',
      email: 'Please enter a valid email',
      minLength: 'Must be at least {min} characters'
    }
  },
  items: {
    count: {
      zero: 'No items',
      one: '{count} item',
      other: '{count} items'
    }
  }
};

// locales/es.js
export default {
  common: {
    buttons: {
      save: 'Guardar',
      cancel: 'Cancelar',
      delete: 'Eliminar',
      confirm: 'Confirmar'
    }
  },
  items: {
    count: {
      zero: 'Sin elementos',
      one: '{count} elemento',
      other: '{count} elementos'
    }
  }
};
```

---

## Task 3.4: Theming System

### CSS Custom Properties Theme

```css
/* css/themes/base.css */

:root {
  /* Color tokens */
  --rnx-primary: #0d6efd;
  --rnx-primary-hover: #0b5ed7;
  --rnx-primary-active: #0a58ca;
  --rnx-secondary: #6c757d;
  --rnx-success: #198754;
  --rnx-danger: #dc3545;
  --rnx-warning: #ffc107;
  --rnx-info: #0dcaf0;

  /* Surface colors */
  --rnx-background: #ffffff;
  --rnx-surface: #f8f9fa;
  --rnx-surface-variant: #e9ecef;

  /* Text colors */
  --rnx-text-primary: #212529;
  --rnx-text-secondary: #6c757d;
  --rnx-text-disabled: #adb5bd;
  --rnx-text-on-primary: #ffffff;

  /* Border */
  --rnx-border-color: #dee2e6;
  --rnx-border-radius: 0.375rem;

  /* Spacing */
  --rnx-spacing-xs: 0.25rem;
  --rnx-spacing-sm: 0.5rem;
  --rnx-spacing-md: 1rem;
  --rnx-spacing-lg: 1.5rem;
  --rnx-spacing-xl: 3rem;

  /* Typography */
  --rnx-font-family: system-ui, -apple-system, sans-serif;
  --rnx-font-size-sm: 0.875rem;
  --rnx-font-size-base: 1rem;
  --rnx-font-size-lg: 1.25rem;

  /* Shadows */
  --rnx-shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
  --rnx-shadow-md: 0 4px 6px rgba(0,0,0,0.1);
  --rnx-shadow-lg: 0 10px 15px rgba(0,0,0,0.1);

  /* Transitions */
  --rnx-transition-fast: 150ms ease;
  --rnx-transition-normal: 300ms ease;
}

/* Dark mode */
[data-theme="dark"] {
  --rnx-background: #121212;
  --rnx-surface: #1e1e1e;
  --rnx-surface-variant: #2d2d2d;
  --rnx-text-primary: #ffffff;
  --rnx-text-secondary: #b0b0b0;
  --rnx-border-color: #404040;
}
```

### Theme Manager

```javascript
// utils/theme.js

/**
 * Theme management for rnxJS
 */
class ThemeManager {
  constructor() {
    this.themes = new Map();
    this.currentTheme = 'default';
    this.subscribers = new Set();

    // Register built-in themes
    this.registerTheme('default', {});
    this.registerTheme('dark', { mode: 'dark' });
  }

  /**
   * Register a custom theme
   */
  registerTheme(name, config) {
    this.themes.set(name, config);
  }

  /**
   * Set the current theme
   */
  setTheme(name) {
    const theme = this.themes.get(name);
    if (!theme) {
      console.warn(`[rnxJS] Theme not found: ${name}`);
      return;
    }

    this.currentTheme = name;

    // Apply theme to document
    document.documentElement.setAttribute('data-theme', name);

    // Apply CSS custom properties
    if (theme.variables) {
      for (const [key, value] of Object.entries(theme.variables)) {
        document.documentElement.style.setProperty(`--rnx-${key}`, value);
      }
    }

    // Notify subscribers
    this.notifySubscribers(name);

    // Persist preference
    try {
      localStorage.setItem('rnx-theme', name);
    } catch (e) {
      // localStorage not available
    }
  }

  /**
   * Get current theme
   */
  getTheme() {
    return this.currentTheme;
  }

  /**
   * Toggle between light and dark
   */
  toggleDarkMode() {
    const isDark = this.currentTheme === 'dark' ||
      document.documentElement.getAttribute('data-theme') === 'dark';
    this.setTheme(isDark ? 'default' : 'dark');
  }

  /**
   * Detect system preference
   */
  detectSystemPreference() {
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    return prefersDark ? 'dark' : 'default';
  }

  /**
   * Initialize theme from stored or system preference
   */
  init() {
    try {
      const stored = localStorage.getItem('rnx-theme');
      if (stored && this.themes.has(stored)) {
        this.setTheme(stored);
        return;
      }
    } catch (e) {
      // localStorage not available
    }

    // Fall back to system preference
    this.setTheme(this.detectSystemPreference());

    // Listen for system preference changes
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (e) => {
      if (!localStorage.getItem('rnx-theme')) {
        this.setTheme(e.matches ? 'dark' : 'default');
      }
    });
  }

  /**
   * Subscribe to theme changes
   */
  subscribe(callback) {
    this.subscribers.add(callback);
    return () => this.subscribers.delete(callback);
  }

  notifySubscribers(theme) {
    for (const callback of this.subscribers) {
      callback(theme);
    }
  }
}

export const theme = new ThemeManager();
```

### Allow Custom CSS Frameworks

```javascript
// utils/config.js

/**
 * rnxJS configuration
 */
export const config = {
  // CSS framework integration
  cssFramework: 'bootstrap', // 'bootstrap' | 'tailwind' | 'custom' | 'none'

  // Component styling
  componentStyles: {
    button: {
      base: 'btn',
      variants: {
        primary: 'btn-primary',
        secondary: 'btn-secondary',
        // ...
      }
    }
  },

  // Allow override
  setConfig(options) {
    Object.assign(this, options);
  }
};

// Usage for custom styling
export function Button({ variant = 'primary', className = '', ...props }) {
  const styles = config.componentStyles.button;
  const classes = [
    styles.base,
    styles.variants[variant],
    className
  ].filter(Boolean).join(' ');

  // ...
}
```

---

## Task 3.5: Error Tracking Integration

```javascript
// utils/errorTracking.js

/**
 * Error tracking integration for rnxJS
 * Supports Sentry, LogRocket, Bugsnag, or custom handlers
 */
class ErrorTracker {
  constructor() {
    this.handlers = [];
    this.enabled = true;
  }

  /**
   * Add an error handler
   */
  addHandler(handler) {
    this.handlers.push(handler);
    return () => {
      const index = this.handlers.indexOf(handler);
      if (index > -1) this.handlers.splice(index, 1);
    };
  }

  /**
   * Capture an error
   */
  captureError(error, context = {}) {
    if (!this.enabled) return;

    const errorData = {
      message: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString(),
      context: {
        ...context,
        url: window.location.href,
        userAgent: navigator.userAgent
      }
    };

    // Console log in development
    console.error('[rnxJS Error]', errorData);

    // Call all handlers
    for (const handler of this.handlers) {
      try {
        handler(errorData);
      } catch (e) {
        console.error('[rnxJS] Error handler failed:', e);
      }
    }
  }

  /**
   * Capture a message/warning
   */
  captureMessage(message, level = 'warning', context = {}) {
    if (!this.enabled) return;

    for (const handler of this.handlers) {
      try {
        handler({ message, level, context });
      } catch (e) {
        console.error('[rnxJS] Error handler failed:', e);
      }
    }
  }

  /**
   * Integration with Sentry
   */
  static sentryHandler(Sentry) {
    return (error) => {
      if (error.stack) {
        Sentry.captureException(new Error(error.message));
      } else {
        Sentry.captureMessage(error.message, error.level || 'warning');
      }
    };
  }

  /**
   * Integration with LogRocket
   */
  static logRocketHandler(LogRocket) {
    return (error) => {
      if (error.stack) {
        LogRocket.captureException(new Error(error.message));
      } else {
        LogRocket.log(error.message);
      }
    };
  }
}

export const errorTracker = new ErrorTracker();

// Global error handler
window.addEventListener('error', (event) => {
  errorTracker.captureError(event.error, { source: 'window.onerror' });
});

window.addEventListener('unhandledrejection', (event) => {
  errorTracker.captureError(
    event.reason instanceof Error ? event.reason : new Error(String(event.reason)),
    { source: 'unhandledrejection' }
  );
});
```

### Usage with Sentry

```javascript
import * as Sentry from '@sentry/browser';
import { errorTracker, ErrorTracker } from 'rnxjs';

// Initialize Sentry
Sentry.init({ dsn: 'your-dsn' });

// Connect to rnxJS
errorTracker.addHandler(ErrorTracker.sentryHandler(Sentry));
```

---

## Acceptance Criteria

### Must Have

- [ ] Full TypeScript source with strict mode
- [ ] Type definitions for all public APIs
- [ ] Keyboard navigation for all components
- [ ] Focus management for Modal, Dropdown, etc.
- [ ] ARIA attributes on all components
- [ ] i18n core with pluralization support
- [ ] Theme system with CSS custom properties
- [ ] Error tracking hooks

### Should Have

- [ ] WCAG 2.1 AA compliance audit passed
- [ ] i18n lazy loading for locales
- [ ] System dark mode detection
- [ ] Screen reader announcements

### Nice to Have

- [ ] RTL support
- [ ] Custom locale date/number formats
- [ ] Theme preview component

---

## Enterprise Checklist

- [ ] TypeScript: Full source code
- [ ] Accessibility: WCAG 2.1 AA compliant
- [ ] i18n: 10+ languages supported
- [ ] Theming: Dark mode + custom themes
- [ ] Error tracking: Sentry/LogRocket integration
- [ ] Documentation: Enterprise usage guide
- [ ] Testing: 95%+ coverage

---

## Definition of Done

1. All TypeScript code compiles with strict mode
2. All components pass axe-core accessibility audit
3. i18n system supports English + Spanish + French
4. Theme system with light/dark toggle working
5. Error tracking integrated with at least one service
6. Documentation updated with enterprise guide
7. All tests passing
