# Sprint 5: Ecosystem & Adoption

**Priority**: MEDIUM
**Duration**: Ongoing
**Goal**: Build community and drive adoption

---

## Overview

This sprint focuses on building the ecosystem around rnxJS that will drive adoption. The best framework in the world is useless if no one knows about it or can find resources to learn it.

---

## Task 5.1: Backend Framework Integrations

### Django Integration

```python
# django_rnx/templatetags/rnx.py

from django import template
from django.utils.safestring import mark_safe
import json

register = template.Library()

@register.simple_tag
def rnx_state(data, var_name='state'):
    """
    Convert Django context to rnxJS reactive state

    Usage:
    {% load rnx %}
    {% rnx_state user_data 'state' %}
    """
    json_data = json.dumps(data)
    return mark_safe(f'''
    <script>
      const {var_name} = rnx.createReactiveState({json_data});
      rnx.loadComponents(document, {var_name});
    </script>
    ''')


@register.inclusion_tag('rnx/scripts.html')
def rnx_scripts(cdn=True, theme='bootstrap'):
    """
    Include rnxJS scripts

    Usage:
    {% rnx_scripts cdn=True theme='bootstrap' %}
    """
    return {
        'cdn': cdn,
        'theme': theme
    }


@register.simple_tag
def rnx_component(name, **kwargs):
    """
    Render an rnxJS component with props

    Usage:
    {% rnx_component 'Button' variant='primary' label='Save' %}
    """
    props = ' '.join(f'{k}="{v}"' for k, v in kwargs.items())
    return mark_safe(f'<{name} {props}></{name}>')
```

```html
<!-- templates/rnx/scripts.html -->
{% if cdn %}
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css" rel="stylesheet">
<script src="https://unpkg.com/@arnelirobles/rnxjs/dist/rnx.global.js"></script>
{% else %}
<link href="{% static 'css/bootstrap.min.css' %}" rel="stylesheet">
<script src="{% static 'js/rnx.global.js' %}"></script>
{% endif %}

{% if theme == 'material' %}
<link href="{% static 'css/bootstrap-m3-theme.css' %}" rel="stylesheet">
{% endif %}
```

```html
<!-- Example Django template -->
{% load rnx %}
<!DOCTYPE html>
<html>
<head>
  {% rnx_scripts %}
</head>
<body>
  <div class="container">
    <h1>Users</h1>

    <DataTable
      data="state.users"
      columns='{{ columns_json }}'
      page-size="10"
    />
  </div>

  {% rnx_state users_data 'state' %}
</body>
</html>
```

### Laravel Integration

```php
// packages/laravel-rnx/src/RnxServiceProvider.php

namespace Rnx\Laravel;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Blade;

class RnxServiceProvider extends ServiceProvider
{
    public function boot()
    {
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'rnx');

        $this->publishes([
            __DIR__.'/../public' => public_path('vendor/rnx'),
        ], 'rnx-assets');

        // Blade directives
        Blade::directive('rnxScripts', function ($expression) {
            return "<?php echo view('rnx::scripts', $expression)->render(); ?>";
        });

        Blade::directive('rnxState', function ($expression) {
            return "<?php echo app('rnx')->renderState($expression); ?>";
        });

        Blade::component('rnx-button', Components\Button::class);
        Blade::component('rnx-input', Components\Input::class);
        Blade::component('rnx-datatable', Components\DataTable::class);
    }
}
```

```php
// Example: Blade component
// resources/views/components/datatable.blade.php

@props([
    'data' => [],
    'columns' => [],
    'pageSize' => 10,
    'searchable' => true
])

<DataTable
    data="{{ $data }}"
    columns='@json($columns)'
    page-size="{{ $pageSize }}"
    searchable="{{ $searchable ? 'true' : 'false' }}"
/>
```

```blade
{{-- Example usage --}}
@extends('layouts.app')

@section('content')
<div class="container">
    <h1>Users</h1>

    <x-rnx-datatable
        :data="$users"
        :columns="[
            ['key' => 'name', 'label' => 'Name', 'sortable' => true],
            ['key' => 'email', 'label' => 'Email'],
            ['key' => 'role', 'label' => 'Role']
        ]"
        :page-size="20"
    />
</div>

@rnxState(['users' => $users])
@endsection
```

### Rails Integration

```ruby
# lib/rnx_rails/engine.rb

module RnxRails
  class Engine < ::Rails::Engine
    isolate_namespace RnxRails

    initializer "rnx_rails.assets" do |app|
      app.config.assets.paths << root.join("vendor", "assets", "javascripts")
      app.config.assets.paths << root.join("vendor", "assets", "stylesheets")
    end

    initializer "rnx_rails.helpers" do
      ActiveSupport.on_load(:action_view) do
        include RnxRails::Helpers
      end
    end
  end
end
```

```ruby
# lib/rnx_rails/helpers.rb

module RnxRails
  module Helpers
    def rnx_scripts(cdn: true, theme: 'bootstrap')
      render partial: 'rnx/scripts', locals: { cdn: cdn, theme: theme }
    end

    def rnx_state(data, var_name: 'state')
      content_tag :script do
        raw <<~JS
          const #{var_name} = rnx.createReactiveState(#{data.to_json});
          rnx.loadComponents(document, #{var_name});
        JS
      end
    end

    def rnx_component(name, **props)
      props_str = props.map { |k, v|
        value = v.is_a?(Hash) || v.is_a?(Array) ? v.to_json : v
        "#{k.to_s.dasherize}=\"#{h(value)}\""
      }.join(' ')

      raw "<#{name} #{props_str}></#{name}>"
    end
  end
end
```

### Express/Node Integration

```javascript
// packages/rnx-express/index.js

const path = require('path');

/**
 * Express middleware for rnxJS
 */
function rnxMiddleware(options = {}) {
  const {
    cdn = true,
    theme = 'bootstrap',
    staticPath = '/rnx'
  } = options;

  return (req, res, next) => {
    // Add rnx helpers to response locals
    res.locals.rnx = {
      scripts: () => cdn
        ? `<script src="https://unpkg.com/@arnelirobles/rnxjs/dist/rnx.global.js"></script>`
        : `<script src="${staticPath}/rnx.global.js"></script>`,

      state: (data, varName = 'state') => `
        <script>
          const ${varName} = rnx.createReactiveState(${JSON.stringify(data)});
          rnx.loadComponents(document, ${varName});
        </script>
      `,

      component: (name, props = {}) => {
        const propsStr = Object.entries(props)
          .map(([k, v]) => {
            const value = typeof v === 'object' ? JSON.stringify(v) : v;
            return `${k}="${value.replace(/"/g, '&quot;')}"`;
          })
          .join(' ');
        return `<${name} ${propsStr}></${name}>`;
      }
    };

    next();
  };
}

module.exports = { rnxMiddleware };
```

---

## Task 5.2: Plugin System

### Plugin Architecture

```javascript
// utils/plugins.js

/**
 * Plugin system for rnxJS
 */
class PluginManager {
  constructor() {
    this.plugins = new Map();
    this.hooks = new Map();
  }

  /**
   * Register a plugin
   */
  use(plugin, options = {}) {
    if (typeof plugin === 'function') {
      plugin = plugin(options);
    }

    if (!plugin.name) {
      throw new Error('Plugin must have a name');
    }

    if (this.plugins.has(plugin.name)) {
      console.warn(`[rnxJS] Plugin "${plugin.name}" is already registered`);
      return;
    }

    this.plugins.set(plugin.name, plugin);

    // Call plugin install hook
    if (typeof plugin.install === 'function') {
      plugin.install(this.getContext());
    }

    // Register plugin hooks
    if (plugin.hooks) {
      for (const [hookName, handler] of Object.entries(plugin.hooks)) {
        this.addHook(hookName, handler);
      }
    }

    console.log(`[rnxJS] Plugin "${plugin.name}" installed`);
  }

  /**
   * Add a hook handler
   */
  addHook(name, handler) {
    if (!this.hooks.has(name)) {
      this.hooks.set(name, []);
    }
    this.hooks.get(name).push(handler);
  }

  /**
   * Run hooks
   */
  async runHook(name, context) {
    const handlers = this.hooks.get(name) || [];
    for (const handler of handlers) {
      await handler(context);
    }
  }

  /**
   * Get plugin context
   */
  getContext() {
    return {
      registerComponent: rnx.registerComponent,
      createReactiveState: rnx.createReactiveState,
      addHook: this.addHook.bind(this)
    };
  }
}

export const plugins = new PluginManager();
```

### Example Plugins

#### Router Plugin

```javascript
// plugins/router.js

/**
 * Simple hash-based router for rnxJS
 *
 * Usage:
 * rnx.plugins.use(routerPlugin, {
 *   routes: {
 *     '/': 'home',
 *     '/users': 'users',
 *     '/users/:id': 'userDetail'
 *   }
 * });
 *
 * // In HTML:
 * <div data-route="home">Home content</div>
 * <div data-route="users">Users list</div>
 */
export function routerPlugin(options = {}) {
  const { routes = {}, mode = 'hash' } = options;

  return {
    name: 'router',

    install(ctx) {
      const state = ctx.createReactiveState({
        currentRoute: '',
        params: {}
      });

      // Match route and extract params
      function matchRoute(path) {
        for (const [pattern, name] of Object.entries(routes)) {
          const regex = new RegExp(
            '^' + pattern.replace(/:(\w+)/g, '(?<$1>[^/]+)') + '$'
          );
          const match = path.match(regex);
          if (match) {
            return { name, params: match.groups || {} };
          }
        }
        return { name: '404', params: {} };
      }

      // Update route
      function updateRoute() {
        const path = mode === 'hash'
          ? window.location.hash.slice(1) || '/'
          : window.location.pathname;

        const { name, params } = matchRoute(path);
        state.currentRoute = name;
        state.params = params;

        // Show/hide route elements
        document.querySelectorAll('[data-route]').forEach(el => {
          el.style.display = el.dataset.route === name ? '' : 'none';
        });
      }

      // Navigation
      window.rnxRouter = {
        push(path) {
          if (mode === 'hash') {
            window.location.hash = path;
          } else {
            history.pushState({}, '', path);
            updateRoute();
          }
        },
        replace(path) {
          if (mode === 'hash') {
            window.location.replace('#' + path);
          } else {
            history.replaceState({}, '', path);
            updateRoute();
          }
        },
        state
      };

      // Listen for changes
      window.addEventListener(mode === 'hash' ? 'hashchange' : 'popstate', updateRoute);

      // Initial route
      updateRoute();
    }
  };
}
```

#### Toast Plugin

```javascript
// plugins/toast.js

/**
 * Toast notifications plugin
 *
 * Usage:
 * rnx.toast.success('Saved!');
 * rnx.toast.error('Something went wrong');
 * rnx.toast.info('Processing...');
 */
export function toastPlugin(options = {}) {
  const {
    position = 'top-right',
    duration = 3000,
    maxToasts = 5
  } = options;

  return {
    name: 'toast',

    install() {
      // Create container
      const container = document.createElement('div');
      container.className = `rnx-toast-container rnx-toast-${position}`;
      container.setAttribute('aria-live', 'polite');
      document.body.appendChild(container);

      const toasts = [];

      function show(message, type = 'info', customDuration) {
        const toast = document.createElement('div');
        toast.className = `rnx-toast rnx-toast-${type}`;
        toast.setAttribute('role', 'alert');

        const icons = {
          success: 'check-circle-fill',
          error: 'x-circle-fill',
          warning: 'exclamation-triangle-fill',
          info: 'info-circle-fill'
        };

        toast.innerHTML = `
          <i class="bi bi-${icons[type] || icons.info}"></i>
          <span class="rnx-toast-message">${rnx.escapeHtml(message)}</span>
          <button class="rnx-toast-close" aria-label="Close">
            <i class="bi bi-x"></i>
          </button>
        `;

        // Close button
        toast.querySelector('.rnx-toast-close').addEventListener('click', () => {
          remove(toast);
        });

        // Add to container
        container.appendChild(toast);
        toasts.push(toast);

        // Animate in
        requestAnimationFrame(() => {
          toast.classList.add('rnx-toast-show');
        });

        // Remove oldest if too many
        while (toasts.length > maxToasts) {
          remove(toasts[0]);
        }

        // Auto-remove
        const timeout = customDuration ?? duration;
        if (timeout > 0) {
          setTimeout(() => remove(toast), timeout);
        }

        return toast;
      }

      function remove(toast) {
        toast.classList.remove('rnx-toast-show');
        toast.classList.add('rnx-toast-hide');

        setTimeout(() => {
          if (toast.parentNode) {
            toast.parentNode.removeChild(toast);
          }
          const idx = toasts.indexOf(toast);
          if (idx > -1) toasts.splice(idx, 1);
        }, 300);
      }

      // Public API
      window.rnx.toast = {
        show,
        success: (msg, dur) => show(msg, 'success', dur),
        error: (msg, dur) => show(msg, 'error', dur),
        warning: (msg, dur) => show(msg, 'warning', dur),
        info: (msg, dur) => show(msg, 'info', dur),
        clear: () => toasts.forEach(remove)
      };
    }
  };
}
```

#### Storage Plugin

```javascript
// plugins/storage.js

/**
 * Persistent state storage plugin
 *
 * Usage:
 * const state = rnx.createReactiveState({
 *   user: null,
 *   theme: 'light'
 * });
 *
 * rnx.storage.persist(state, 'app-state', ['theme']);
 * // Now state.theme will be saved to localStorage and restored on load
 */
export function storagePlugin(options = {}) {
  const {
    storage = localStorage,
    prefix = 'rnx_'
  } = options;

  return {
    name: 'storage',

    install() {
      window.rnx.storage = {
        /**
         * Persist specific state paths to storage
         */
        persist(state, key, paths = null) {
          const storageKey = prefix + key;

          // Load initial values
          try {
            const saved = storage.getItem(storageKey);
            if (saved) {
              const data = JSON.parse(saved);
              for (const [path, value] of Object.entries(data)) {
                setNestedValue(state, path, value);
              }
            }
          } catch (e) {
            console.warn('[rnxJS Storage] Failed to load:', e);
          }

          // Subscribe to changes
          const pathsToWatch = paths || Object.keys(state);

          pathsToWatch.forEach(path => {
            state.subscribe(path, () => {
              try {
                const data = {};
                pathsToWatch.forEach(p => {
                  data[p] = getNestedValue(state, p);
                });
                storage.setItem(storageKey, JSON.stringify(data));
              } catch (e) {
                console.warn('[rnxJS Storage] Failed to save:', e);
              }
            });
          });
        },

        /**
         * Clear persisted data
         */
        clear(key) {
          storage.removeItem(prefix + key);
        }
      };
    }
  };
}
```

---

## Task 5.3: Community Building

### GitHub Presence

```markdown
<!-- .github/ISSUE_TEMPLATE/bug_report.md -->
---
name: Bug Report
about: Report a bug in rnxJS
labels: bug
---

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Create state with '...'
2. Add data-bind to '....'
3. See error

**Expected behavior**
What you expected to happen.

**Code Sample**
```html
<!-- Minimal reproduction -->
```

**Environment:**
- rnxJS version:
- Browser:
- OS:
```

### Discord/Community

```
rnxJS Discord Structure:
├── #announcements (read-only)
├── #getting-started
├── #help
├── #showcase
├── #plugins
├── #backend-django
├── #backend-laravel
├── #backend-rails
├── #feature-requests
└── #contributors
```

### Blog/Content Strategy

| Content Type | Frequency | Topics |
|--------------|-----------|--------|
| Tutorial | Weekly | "Build X with rnxJS" |
| Comparison | Monthly | "rnxJS vs React for Admin Panels" |
| Case Study | Monthly | Real user success stories |
| Release Notes | Per release | What's new, migration |
| Video | Bi-weekly | YouTube tutorials |

### Content Ideas

1. "Build a Full CRUD App in 30 Minutes"
2. "Why We Chose rnxJS Over React"
3. "Migrating from jQuery: A Step-by-Step Guide"
4. "10 Components Every Admin Panel Needs"
5. "rnxJS for Django Developers"
6. "Performance Comparison: rnxJS vs Vue vs React"

---

## Task 5.4: Distribution Channels

### NPM Package Optimization

```json
// package.json additions
{
  "sideEffects": false,
  "exports": {
    ".": {
      "import": "./dist/rnx.esm.js",
      "require": "./dist/rnx.cjs.js",
      "types": "./dist/types/index.d.ts"
    },
    "./components/*": {
      "import": "./dist/components/*.js"
    },
    "./plugins/*": {
      "import": "./dist/plugins/*.js"
    },
    "./css/*": "./css/*"
  },
  "keywords": [
    "javascript",
    "framework",
    "reactive",
    "bootstrap",
    "django",
    "laravel",
    "rails",
    "admin-dashboard",
    "no-build",
    "vanilla-js",
    "data-binding",
    "components",
    "ui-library"
  ]
}
```

### CDN Distribution

- **unpkg**: `https://unpkg.com/@arnelirobles/rnxjs`
- **jsDelivr**: `https://cdn.jsdelivr.net/npm/@arnelirobles/rnxjs`
- **cdnjs**: Submit for inclusion

### Package Registries

1. **npm** (primary)
2. **Yarn** (automatic via npm)
3. **pnpm** (automatic via npm)
4. **Deno** (create deno.land/x entry)
5. **Packagist** (for Laravel integration)
6. **PyPI** (for Django integration)
7. **RubyGems** (for Rails integration)

---

## Task 5.5: Enterprise Support

### Support Tiers

| Tier | Price | Features |
|------|-------|----------|
| **Community** | Free | GitHub issues, Discord |
| **Pro** | $99/mo | Email support, 48h response |
| **Enterprise** | $499/mo | Slack channel, 4h response, training |
| **Custom** | Contact | On-site, custom components, SLA |

### Enterprise Features (Paid)

1. **Priority Support**: Direct Slack/email access
2. **Custom Components**: Built to specification
3. **Training**: Live workshops for teams
4. **Migration Assistance**: Help moving from jQuery/React
5. **Security Audits**: Regular vulnerability assessments
6. **SLA**: Uptime and response time guarantees

---

## Task 5.6: Metrics & Analytics

### Success Metrics

| Metric | Current | 6-Month Goal | 12-Month Goal |
|--------|---------|--------------|---------------|
| npm weekly downloads | ? | 1,000 | 5,000 |
| GitHub stars | ? | 1,000 | 5,000 |
| Discord members | 0 | 200 | 1,000 |
| Production users | 0 | 20 | 100 |
| Enterprise customers | 0 | 2 | 10 |
| Contributors | 1 | 10 | 30 |

### Analytics Setup

```javascript
// Optional analytics for framework usage (opt-in only)
const analytics = {
  init(options = {}) {
    if (!options.enabled) return;

    // Track framework usage (anonymized)
    fetch('https://analytics.rnxjs.dev/usage', {
      method: 'POST',
      body: JSON.stringify({
        version: RNX_VERSION,
        components: Object.keys(registeredComponents),
        plugins: Array.from(plugins.plugins.keys())
      })
    }).catch(() => {}); // Silent fail
  }
};
```

---

## Acceptance Criteria

### Must Have

- [ ] Django integration package published
- [ ] Laravel integration package published
- [ ] Plugin system with 3+ plugins (router, toast, storage)
- [ ] GitHub issue/PR templates
- [ ] Basic documentation site live
- [ ] unpkg and jsDelivr working

### Should Have

- [ ] Rails integration package
- [ ] Express middleware
- [ ] Discord community set up
- [ ] 5+ blog posts/tutorials
- [ ] YouTube channel with tutorials

### Nice to Have

- [ ] Enterprise support offering
- [ ] Contributor guidelines
- [ ] Showcase of production sites
- [ ] Conference talk/presentation

---

## Definition of Done

1. Backend integrations work in real projects
2. Plugin system is extensible and documented
3. Community channels are active
4. Framework is discoverable via search
5. At least 10 real production users
6. Positive community feedback and contributions
