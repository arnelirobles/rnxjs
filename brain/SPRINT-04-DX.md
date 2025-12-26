# Sprint 4: Developer Experience

**Priority**: HIGH
**Duration**: 3-4 weeks
**Goal**: Achieve the "shit, that's it? I'm done?" moment

---

## The North Star

> **Developer Experience Goal**: A backend developer should be able to build a complete CRUD dashboard in under 30 minutes, with zero configuration, and feel like they cheated because it was too easy.

### The "Shit, That's It?" Checklist

- [ ] Copy 1 script tag
- [ ] Write HTML like it's 2005 (but reactive)
- [ ] Add `data-bind` to an input
- [ ] Watch it magically work
- [ ] Add a component with one tag
- [ ] Wonder why React exists

---

## Task 4.1: Expanded Component Library

### Current: 34 Components
### Target: 60+ Components

### Component Wishlist - Tier 1 (Must Have)

| Component | Description | Use Case |
|-----------|-------------|----------|
| **DataTable** | Sortable, filterable, paginated table | Every admin panel |
| **DatePicker** | Calendar date selection | Forms |
| **TimePicker** | Time selection | Scheduling |
| **DateRangePicker** | From/to date selection | Reports |
| **Autocomplete** | Search with suggestions | Search, selection |
| **FileUpload** | Drag & drop file upload | File management |
| **ImageUpload** | Image preview + crop | User profiles |
| **RichTextEditor** | Basic WYSIWYG | Content editing |
| **CodeEditor** | Syntax highlighted code input | Dev tools |
| **TreeView** | Hierarchical data display | File browsers |
| **Sidebar** | Collapsible sidebar nav | Layouts |
| **Breadcrumb** | Navigation breadcrumbs | Navigation |
| **Stepper** | Multi-step form wizard | Onboarding |
| **Timeline** | Chronological events | Activity feeds |
| **StatCard** | Number + label + trend | Dashboards |
| **Chart** | Basic charts (bar, line, pie) | Analytics |
| **Avatar** | User avatar with fallback | User lists |
| **AvatarGroup** | Stacked avatars | Team display |
| **EmptyState** | No data placeholder | Lists |
| **ErrorState** | Error display | Error handling |
| **LoadingState** | Full-page loading | Initial load |
| **Skeleton** | Content placeholder | Loading |
| **Tooltip** | Hover information | UX |
| **Popover** | Click information | Forms |
| **Dropdown** | Click menu | Actions |
| **ContextMenu** | Right-click menu | Power users |
| **Notification** | Toast notifications | Feedback |
| **ConfirmDialog** | Yes/No confirmation | Destructive actions |
| **CommandPalette** | Cmd+K search | Power users |

### Component Wishlist - Tier 2 (Nice to Have)

| Component | Description | Use Case |
|-----------|-------------|----------|
| **Kanban** | Drag & drop board | Project management |
| **Calendar** | Event calendar | Scheduling |
| **Scheduler** | Time slot scheduling | Appointments |
| **GanttChart** | Project timeline | Project management |
| **OrgChart** | Organization hierarchy | HR tools |
| **DiffViewer** | Text comparison | Version control |
| **JsonViewer** | JSON tree display | API debugging |
| **LogViewer** | Log display + filtering | DevOps |
| **Terminal** | Terminal emulator | DevOps |
| **Map** | Location display | Geo apps |

### Implementation: DataTable (Most Requested)

```javascript
// components/DataTable.js

/**
 * DataTable - The component every admin panel needs
 *
 * Features:
 * - Sortable columns
 * - Filterable
 * - Paginated
 * - Row selection
 * - Row actions
 * - Column resizing
 * - Export to CSV
 * - Keyboard navigation
 *
 * Usage:
 * <DataTable
 *   data="state.users"
 *   columns='[
 *     {"key": "name", "label": "Name", "sortable": true},
 *     {"key": "email", "label": "Email", "sortable": true},
 *     {"key": "role", "label": "Role", "filterable": true},
 *     {"key": "actions", "label": "", "type": "actions"}
 *   ]'
 *   page-size="10"
 *   selectable="true"
 *   on-row-click="handleRowClick"
 *   on-selection-change="handleSelection"
 * />
 */
export function DataTable({
  data = [],
  columns = [],
  pageSize = 10,
  selectable = false,
  searchable = true,
  exportable = true,
  loading = false,
  emptyText = 'No data available',
  onRowClick,
  onSelectionChange,
  rowActions = [],
  state
}) {
  // Parse columns if string
  const cols = typeof columns === 'string' ? JSON.parse(columns) : columns;

  // Internal state
  let currentPage = 0;
  let sortColumn = null;
  let sortDirection = 'asc';
  let searchQuery = '';
  let selectedRows = new Set();

  // Get data from state if path
  const getData = () => {
    if (typeof data === 'string' && state) {
      return data.split('.').reduce((o, k) => o?.[k], state) || [];
    }
    return data;
  };

  // Filter and sort data
  const getProcessedData = () => {
    let result = [...getData()];

    // Search filter
    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      result = result.filter(row =>
        cols.some(col =>
          String(row[col.key] || '').toLowerCase().includes(query)
        )
      );
    }

    // Sort
    if (sortColumn) {
      result.sort((a, b) => {
        const aVal = a[sortColumn] ?? '';
        const bVal = b[sortColumn] ?? '';
        const cmp = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
        return sortDirection === 'asc' ? cmp : -cmp;
      });
    }

    return result;
  };

  // Pagination
  const getPageData = () => {
    const all = getProcessedData();
    const start = currentPage * pageSize;
    return all.slice(start, start + pageSize);
  };

  const getTotalPages = () => Math.ceil(getProcessedData().length / pageSize);

  // Build table
  const container = document.createElement('div');
  container.className = 'rnx-datatable';

  const render = () => {
    const pageData = getPageData();
    const totalPages = getTotalPages();
    const total = getProcessedData().length;

    container.innerHTML = `
      <div class="datatable-header">
        ${searchable ? `
          <div class="datatable-search">
            <input
              type="text"
              class="form-control form-control-sm"
              placeholder="Search..."
              value="${escapeHtml(searchQuery)}"
              data-ref="search"
            />
          </div>
        ` : ''}
        ${exportable ? `
          <button class="btn btn-sm btn-outline-secondary" data-ref="export">
            <i class="bi bi-download"></i> Export
          </button>
        ` : ''}
      </div>

      <div class="table-responsive">
        <table class="table table-hover">
          <thead>
            <tr>
              ${selectable ? `
                <th class="datatable-checkbox">
                  <input type="checkbox" data-ref="selectAll" />
                </th>
              ` : ''}
              ${cols.map(col => `
                <th
                  class="${col.sortable ? 'sortable' : ''} ${sortColumn === col.key ? 'sorted ' + sortDirection : ''}"
                  data-column="${col.key}"
                >
                  ${escapeHtml(col.label)}
                  ${col.sortable ? '<i class="bi bi-chevron-expand"></i>' : ''}
                </th>
              `).join('')}
            </tr>
          </thead>
          <tbody>
            ${loading ? `
              <tr><td colspan="${cols.length + (selectable ? 1 : 0)}">
                <div class="text-center p-4">
                  <div class="spinner-border" role="status"></div>
                </div>
              </td></tr>
            ` : pageData.length === 0 ? `
              <tr><td colspan="${cols.length + (selectable ? 1 : 0)}">
                <div class="text-center text-muted p-4">${escapeHtml(emptyText)}</div>
              </td></tr>
            ` : pageData.map((row, idx) => `
              <tr data-row-index="${idx}" class="${selectedRows.has(row.id || idx) ? 'table-active' : ''}">
                ${selectable ? `
                  <td class="datatable-checkbox">
                    <input type="checkbox" data-row-id="${row.id || idx}" ${selectedRows.has(row.id || idx) ? 'checked' : ''} />
                  </td>
                ` : ''}
                ${cols.map(col => `
                  <td data-column="${col.key}">
                    ${col.type === 'actions' ? renderActions(row, rowActions) :
                      col.render ? col.render(row[col.key], row) :
                      escapeHtml(row[col.key] ?? '')}
                  </td>
                `).join('')}
              </tr>
            `).join('')}
          </tbody>
        </table>
      </div>

      <div class="datatable-footer">
        <div class="datatable-info">
          Showing ${currentPage * pageSize + 1}-${Math.min((currentPage + 1) * pageSize, total)} of ${total}
        </div>
        <div class="datatable-pagination">
          <button class="btn btn-sm btn-outline-secondary" data-ref="prevPage" ${currentPage === 0 ? 'disabled' : ''}>
            <i class="bi bi-chevron-left"></i>
          </button>
          <span class="mx-2">Page ${currentPage + 1} of ${totalPages || 1}</span>
          <button class="btn btn-sm btn-outline-secondary" data-ref="nextPage" ${currentPage >= totalPages - 1 ? 'disabled' : ''}>
            <i class="bi bi-chevron-right"></i>
          </button>
        </div>
      </div>
    `;

    attachEventListeners();
  };

  const renderActions = (row, actions) => {
    return `
      <div class="btn-group btn-group-sm">
        ${actions.map(action => `
          <button
            class="btn btn-${action.variant || 'outline-secondary'}"
            data-action="${action.name}"
            data-row-id="${row.id}"
            title="${escapeHtml(action.label)}"
          >
            <i class="bi bi-${action.icon}"></i>
          </button>
        `).join('')}
      </div>
    `;
  };

  const attachEventListeners = () => {
    // Search
    container.querySelector('[data-ref="search"]')?.addEventListener('input', (e) => {
      searchQuery = e.target.value;
      currentPage = 0;
      render();
    });

    // Sort
    container.querySelectorAll('th.sortable').forEach(th => {
      th.addEventListener('click', () => {
        const col = th.dataset.column;
        if (sortColumn === col) {
          sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
        } else {
          sortColumn = col;
          sortDirection = 'asc';
        }
        render();
      });
    });

    // Pagination
    container.querySelector('[data-ref="prevPage"]')?.addEventListener('click', () => {
      if (currentPage > 0) {
        currentPage--;
        render();
      }
    });

    container.querySelector('[data-ref="nextPage"]')?.addEventListener('click', () => {
      if (currentPage < getTotalPages() - 1) {
        currentPage++;
        render();
      }
    });

    // Row click
    if (onRowClick) {
      container.querySelectorAll('tbody tr').forEach(tr => {
        tr.addEventListener('click', (e) => {
          if (e.target.closest('button, input')) return;
          const idx = parseInt(tr.dataset.rowIndex);
          const row = getPageData()[idx];
          if (typeof onRowClick === 'function') {
            onRowClick(row);
          }
        });
      });
    }

    // Export
    container.querySelector('[data-ref="export"]')?.addEventListener('click', exportToCSV);
  };

  const exportToCSV = () => {
    const data = getProcessedData();
    const headers = cols.filter(c => c.type !== 'actions').map(c => c.label);
    const rows = data.map(row =>
      cols.filter(c => c.type !== 'actions').map(c => row[c.key] ?? '')
    );

    const csv = [headers, ...rows]
      .map(row => row.map(cell => `"${String(cell).replace(/"/g, '""')}"`).join(','))
      .join('\n');

    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'export.csv';
    a.click();
    URL.revokeObjectURL(url);
  };

  render();

  // Subscribe to data changes
  if (typeof data === 'string' && state?.subscribe) {
    state.subscribe(data, () => render());
  }

  return container;
}
```

### Implementation: StatCard

```javascript
// components/StatCard.js

/**
 * StatCard - Dashboard number display
 *
 * Usage:
 * <StatCard
 *   label="Total Users"
 *   value="state.stats.users"
 *   icon="people"
 *   trend="+12%"
 *   trend-direction="up"
 *   variant="primary"
 * />
 */
export function StatCard({
  label,
  value,
  icon,
  trend,
  trendDirection = 'neutral', // 'up' | 'down' | 'neutral'
  variant = 'light',
  format = 'number', // 'number' | 'currency' | 'percent'
  href,
  state
}) {
  // Get value from state if path
  const getValue = () => {
    if (typeof value === 'string' && state) {
      const v = value.split('.').reduce((o, k) => o?.[k], state);
      return formatValue(v, format);
    }
    return formatValue(value, format);
  };

  const formatValue = (v, format) => {
    if (v === null || v === undefined) return '—';
    if (format === 'currency') {
      return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(v);
    }
    if (format === 'percent') {
      return new Intl.NumberFormat('en-US', { style: 'percent' }).format(v / 100);
    }
    return new Intl.NumberFormat().format(v);
  };

  const trendColors = {
    up: 'text-success',
    down: 'text-danger',
    neutral: 'text-muted'
  };

  const trendIcons = {
    up: 'arrow-up',
    down: 'arrow-down',
    neutral: 'dash'
  };

  const card = document.createElement(href ? 'a' : 'div');
  card.className = `card stat-card bg-${variant} ${href ? 'stat-card-link' : ''}`;
  if (href) card.href = href;

  card.innerHTML = `
    <div class="card-body">
      <div class="d-flex justify-content-between align-items-start">
        <div>
          <h6 class="stat-label text-muted mb-1">${escapeHtml(label)}</h6>
          <h2 class="stat-value mb-0" data-ref="value">${getValue()}</h2>
          ${trend ? `
            <small class="${trendColors[trendDirection]}">
              <i class="bi bi-${trendIcons[trendDirection]}"></i>
              ${escapeHtml(trend)}
            </small>
          ` : ''}
        </div>
        ${icon ? `
          <div class="stat-icon">
            <i class="bi bi-${icon}"></i>
          </div>
        ` : ''}
      </div>
    </div>
  `;

  // Subscribe to value changes
  if (typeof value === 'string' && state?.subscribe) {
    state.subscribe(value, () => {
      const valueEl = card.querySelector('[data-ref="value"]');
      if (valueEl) valueEl.textContent = getValue();
    });
  }

  return card;
}
```

### Implementation: Skeleton

```javascript
// components/Skeleton.js

/**
 * Skeleton - Loading placeholder
 *
 * Usage:
 * <Skeleton type="text" lines="3" />
 * <Skeleton type="avatar" size="lg" />
 * <Skeleton type="card" />
 * <Skeleton type="table" rows="5" cols="4" />
 */
export function Skeleton({
  type = 'text', // 'text' | 'avatar' | 'card' | 'table' | 'image'
  lines = 3,
  rows = 5,
  cols = 4,
  size = 'md',
  width,
  height,
  animated = true
}) {
  const animClass = animated ? 'skeleton-animated' : '';

  const templates = {
    text: () => `
      <div class="skeleton-text ${animClass}">
        ${Array(lines).fill(0).map((_, i) => `
          <div class="skeleton-line" style="width: ${i === lines - 1 ? '60%' : '100%'}"></div>
        `).join('')}
      </div>
    `,

    avatar: () => `
      <div class="skeleton-avatar skeleton-avatar-${size} ${animClass}"></div>
    `,

    card: () => `
      <div class="card skeleton-card ${animClass}">
        <div class="skeleton-image" style="height: 200px"></div>
        <div class="card-body">
          <div class="skeleton-line" style="width: 60%; height: 24px"></div>
          <div class="skeleton-line" style="width: 100%"></div>
          <div class="skeleton-line" style="width: 80%"></div>
        </div>
      </div>
    `,

    table: () => `
      <div class="skeleton-table ${animClass}">
        <div class="skeleton-row skeleton-header">
          ${Array(cols).fill(0).map(() => '<div class="skeleton-cell"></div>').join('')}
        </div>
        ${Array(rows).fill(0).map(() => `
          <div class="skeleton-row">
            ${Array(cols).fill(0).map(() => '<div class="skeleton-cell"></div>').join('')}
          </div>
        `).join('')}
      </div>
    `,

    image: () => `
      <div class="skeleton-image ${animClass}" style="width: ${width || '100%'}; height: ${height || '200px'}"></div>
    `
  };

  const container = document.createElement('div');
  container.className = 'skeleton-container';
  container.innerHTML = templates[type]?.() || templates.text();

  return container;
}
```

---

## Task 4.2: CLI Tool - `create-rnx`

### The Dream Experience

```bash
$ npx create-rnx my-dashboard

✨ Creating rnxJS project in ./my-dashboard

? What type of project?
  ❯ Admin Dashboard
    Landing Page
    Form Application
    Blank

? CSS Framework?
  ❯ Bootstrap 5 (Default)
    Tailwind CSS
    None

? Backend integration?
  ❯ Django
    Laravel
    Rails
    Express
    None (Static)

✓ Project created!

  cd my-dashboard
  open index.html

  # Or with live reload:
  npx serve

Happy coding! 🚀
```

### CLI Implementation

```javascript
// packages/create-rnx/index.js
#!/usr/bin/env node

import { input, select, confirm } from '@inquirer/prompts';
import fs from 'fs-extra';
import path from 'path';
import chalk from 'chalk';

const TEMPLATES = {
  dashboard: {
    name: 'Admin Dashboard',
    description: 'Full-featured admin panel with sidebar, stats, and data tables'
  },
  landing: {
    name: 'Landing Page',
    description: 'Marketing page with hero, features, and contact form'
  },
  form: {
    name: 'Form Application',
    description: 'Multi-step form with validation'
  },
  blank: {
    name: 'Blank',
    description: 'Minimal setup with just the essentials'
  }
};

async function main() {
  console.log(chalk.cyan('\n✨ Create rnxJS Project\n'));

  const projectName = process.argv[2] || await input({
    message: 'Project name:',
    default: 'my-rnx-app'
  });

  const template = await select({
    message: 'What type of project?',
    choices: Object.entries(TEMPLATES).map(([key, val]) => ({
      value: key,
      name: val.name,
      description: val.description
    }))
  });

  const cssFramework = await select({
    message: 'CSS Framework?',
    choices: [
      { value: 'bootstrap', name: 'Bootstrap 5 (Recommended)' },
      { value: 'tailwind', name: 'Tailwind CSS' },
      { value: 'none', name: 'None' }
    ]
  });

  const backend = await select({
    message: 'Backend integration?',
    choices: [
      { value: 'none', name: 'None (Static HTML)' },
      { value: 'django', name: 'Django' },
      { value: 'laravel', name: 'Laravel' },
      { value: 'rails', name: 'Rails' },
      { value: 'express', name: 'Express' }
    ]
  });

  const projectPath = path.resolve(process.cwd(), projectName);

  // Create project
  console.log(chalk.gray(`\nCreating project in ${projectPath}...\n`));

  await fs.ensureDir(projectPath);
  await copyTemplate(template, projectPath, { cssFramework, backend });

  // Success message
  console.log(chalk.green('\n✓ Project created!\n'));
  console.log('  Next steps:\n');
  console.log(chalk.cyan(`    cd ${projectName}`));
  console.log(chalk.cyan('    open index.html'));
  console.log('\n  Or with live reload:\n');
  console.log(chalk.cyan('    npx serve'));
  console.log(chalk.gray('\n  Happy coding! 🚀\n'));
}

async function copyTemplate(template, dest, options) {
  const templateDir = path.join(__dirname, 'templates', template);
  await fs.copy(templateDir, dest);

  // Customize based on options
  // ...
}

main().catch(console.error);
```

### Dashboard Template

```html
<!-- templates/dashboard/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dashboard</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css" rel="stylesheet">
  <script src="https://unpkg.com/@arnelirobles/rnxjs/dist/rnx.global.js"></script>
  <style>
    .sidebar { width: 250px; min-height: 100vh; }
    .main-content { flex: 1; }
  </style>
</head>
<body>
  <div class="d-flex">
    <!-- Sidebar -->
    <nav class="sidebar bg-dark text-white p-3">
      <h4 class="mb-4">Dashboard</h4>
      <ul class="nav flex-column">
        <li class="nav-item">
          <a class="nav-link text-white" href="#"><i class="bi bi-house me-2"></i> Home</a>
        </li>
        <li class="nav-item">
          <a class="nav-link text-white" href="#"><i class="bi bi-people me-2"></i> Users</a>
        </li>
        <li class="nav-item">
          <a class="nav-link text-white" href="#"><i class="bi bi-gear me-2"></i> Settings</a>
        </li>
      </ul>
    </nav>

    <!-- Main Content -->
    <main class="main-content bg-light p-4">
      <!-- Stats Row -->
      <div class="row g-4 mb-4">
        <div class="col-md-3">
          <StatCard label="Total Users" value="state.stats.users" icon="people" trend="+12%" trend-direction="up" variant="primary" />
        </div>
        <div class="col-md-3">
          <StatCard label="Revenue" value="state.stats.revenue" icon="currency-dollar" format="currency" variant="success" />
        </div>
        <div class="col-md-3">
          <StatCard label="Orders" value="state.stats.orders" icon="cart" variant="info" />
        </div>
        <div class="col-md-3">
          <StatCard label="Conversion" value="state.stats.conversion" icon="graph-up" format="percent" variant="warning" />
        </div>
      </div>

      <!-- Data Table -->
      <Card title="Recent Users">
        <DataTable
          data="state.users"
          columns='[
            {"key": "name", "label": "Name", "sortable": true},
            {"key": "email", "label": "Email", "sortable": true},
            {"key": "role", "label": "Role"},
            {"key": "status", "label": "Status"},
            {"key": "actions", "label": "", "type": "actions"}
          ]'
          row-actions='[
            {"name": "edit", "icon": "pencil", "label": "Edit"},
            {"name": "delete", "icon": "trash", "label": "Delete", "variant": "outline-danger"}
          ]'
          page-size="10"
          searchable="true"
        />
      </Card>
    </main>
  </div>

  <script>
    const { createReactiveState, loadComponents, autoRegisterComponents } = rnx;

    // Create state
    const state = createReactiveState({
      stats: {
        users: 1234,
        revenue: 45678,
        orders: 89,
        conversion: 3.2
      },
      users: [
        { id: 1, name: 'Alice Johnson', email: 'alice@example.com', role: 'Admin', status: 'Active' },
        { id: 2, name: 'Bob Smith', email: 'bob@example.com', role: 'User', status: 'Active' },
        // Add more...
      ]
    });

    // Initialize
    autoRegisterComponents();
    loadComponents(document, state);

    // That's it! 🎉
  </script>
</body>
</html>
```

---

## Task 4.3: Component Creation Guide

### The Simplest Component

```javascript
// A component is just a function that returns an element

function HelloWorld({ name }) {
  const div = document.createElement('div');
  div.textContent = `Hello, ${name}!`;
  return div;
}

// Register it
rnx.registerComponent('HelloWorld', HelloWorld);

// Use it
// <HelloWorld name="Developer" />
```

### Component with State

```javascript
function Counter({ initial = 0 }) {
  let count = parseInt(initial);

  const container = document.createElement('div');
  container.className = 'counter';

  const render = () => {
    container.innerHTML = `
      <span class="count">${count}</span>
      <button class="btn btn-sm btn-primary" data-action="increment">+</button>
      <button class="btn btn-sm btn-secondary" data-action="decrement">-</button>
    `;

    container.querySelector('[data-action="increment"]')
      .addEventListener('click', () => { count++; render(); });
    container.querySelector('[data-action="decrement"]')
      .addEventListener('click', () => { count--; render(); });
  };

  render();
  return container;
}
```

### Component with Reactive State Binding

```javascript
function UserForm({ state }) {
  const form = document.createElement('form');
  form.innerHTML = `
    <div class="mb-3">
      <label class="form-label">Name</label>
      <input type="text" class="form-control" data-bind="user.name" />
    </div>
    <div class="mb-3">
      <label class="form-label">Email</label>
      <input type="email" class="form-control" data-bind="user.email" data-rule="required|email" />
      <div class="text-danger" data-bind="errors.user.email"></div>
    </div>
    <button type="submit" class="btn btn-primary">Save</button>
  `;

  // The magic: data-bind automatically syncs with state!
  // No useState, no setState, no re-renders to manage

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    await saveUser(state.user);
  });

  return form;
}
```

### Component Best Practices

```javascript
/**
 * Best Practice Example: A well-structured component
 *
 * - Props have defaults
 * - Escaped HTML for user content
 * - Clean event handling
 * - Proper cleanup
 */
function Card({
  title = '',
  subtitle = '',
  children = [],
  variant = 'light',
  collapsible = false,
  onClose
}) {
  const card = document.createElement('div');
  card.className = `card bg-${variant}`;

  // Use escapeHtml for any user-provided content
  card.innerHTML = `
    <div class="card-header d-flex justify-content-between align-items-center">
      <div>
        <h5 class="card-title mb-0">${rnx.escapeHtml(title)}</h5>
        ${subtitle ? `<small class="text-muted">${rnx.escapeHtml(subtitle)}</small>` : ''}
      </div>
      <div class="card-actions">
        ${collapsible ? '<button class="btn btn-sm" data-action="collapse"><i class="bi bi-chevron-up"></i></button>' : ''}
        ${onClose ? '<button class="btn btn-sm" data-action="close"><i class="bi bi-x"></i></button>' : ''}
      </div>
    </div>
    <div class="card-body" data-slot="body"></div>
  `;

  // Insert children into slot
  const bodySlot = card.querySelector('[data-slot="body"]');
  children.forEach(child => {
    if (child instanceof Node) {
      bodySlot.appendChild(child);
    } else {
      bodySlot.innerHTML += child;
    }
  });

  // Event handling
  let isCollapsed = false;

  card.querySelector('[data-action="collapse"]')?.addEventListener('click', () => {
    isCollapsed = !isCollapsed;
    bodySlot.style.display = isCollapsed ? 'none' : '';
    card.querySelector('[data-action="collapse"] i').className =
      `bi bi-chevron-${isCollapsed ? 'down' : 'up'}`;
  });

  card.querySelector('[data-action="close"]')?.addEventListener('click', () => {
    if (typeof onClose === 'function') onClose();
    card.remove();
  });

  return card;
}
```

---

## Task 4.4: Documentation Site

### Structure

```
docs/
├── index.html          # Landing page
├── getting-started/
│   ├── installation.html
│   ├── quick-start.html
│   └── first-component.html
├── guide/
│   ├── reactivity.html
│   ├── data-binding.html
│   ├── components.html
│   ├── validation.html
│   ├── theming.html
│   └── i18n.html
├── components/
│   ├── index.html      # Component gallery
│   ├── button.html
│   ├── input.html
│   └── ... (one per component)
├── examples/
│   ├── dashboard.html
│   ├── crud-app.html
│   ├── form-wizard.html
│   └── real-world/
├── api/
│   ├── reactive-state.html
│   ├── data-binder.html
│   └── component-loader.html
└── migration/
    ├── from-jquery.html
    ├── from-vue.html
    └── from-react.html
```

### Interactive Examples

Each component page should have:
1. Live preview
2. Copy-paste code
3. Prop table
4. Use case examples

```html
<!-- Component documentation template -->
<div class="component-doc">
  <h1>Button</h1>
  <p class="lead">A versatile button component with variants and icons.</p>

  <!-- Live Preview -->
  <div class="preview-container">
    <div class="preview">
      <Button variant="primary" label="Click me" />
      <Button variant="secondary" label="Secondary" />
      <Button variant="outline" label="Outline" />
    </div>
    <div class="code-panel">
      <pre><code>&lt;Button variant="primary" label="Click me" /&gt;</code></pre>
      <button class="copy-btn" onclick="copyCode(this)">Copy</button>
    </div>
  </div>

  <!-- Props Table -->
  <h2>Props</h2>
  <table class="table">
    <thead>
      <tr>
        <th>Prop</th>
        <th>Type</th>
        <th>Default</th>
        <th>Description</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>variant</code></td>
        <td>string</td>
        <td>'primary'</td>
        <td>Button style: primary, secondary, outline, text</td>
      </tr>
      <!-- ... -->
    </tbody>
  </table>

  <!-- Examples -->
  <h2>Examples</h2>

  <h3>Button with Icon</h3>
  <div class="preview-container">
    <div class="preview">
      <Button variant="primary" icon="save" label="Save" />
    </div>
    <pre><code>&lt;Button variant="primary" icon="save" label="Save" /&gt;</code></pre>
  </div>
</div>
```

---

## Task 4.5: DevTools Browser Extension

### Features

1. **State Inspector**: View and edit reactive state in real-time
2. **Component Tree**: Visualize component hierarchy
3. **Binding Debugger**: See all data-bind connections
4. **Performance Monitor**: Track updates and render times
5. **Event Logger**: Log all state changes

### Implementation Sketch

```javascript
// devtools/panel.js

class RnxDevTools {
  constructor() {
    this.port = chrome.runtime.connect({ name: 'rnx-devtools' });
    this.init();
  }

  init() {
    // Listen for state updates from page
    this.port.onMessage.addListener((message) => {
      switch (message.type) {
        case 'STATE_UPDATE':
          this.updateStatePanel(message.state);
          break;
        case 'COMPONENT_TREE':
          this.updateComponentTree(message.tree);
          break;
        case 'BINDING_MAP':
          this.updateBindings(message.bindings);
          break;
      }
    });

    // Request initial data
    this.port.postMessage({ type: 'GET_STATE' });
  }

  updateStatePanel(state) {
    const container = document.getElementById('state-tree');
    container.innerHTML = '';
    this.renderStateTree(state, container, '');
  }

  renderStateTree(obj, container, path) {
    for (const [key, value] of Object.entries(obj)) {
      const currentPath = path ? `${path}.${key}` : key;
      const item = document.createElement('div');
      item.className = 'state-item';

      if (typeof value === 'object' && value !== null) {
        item.innerHTML = `
          <span class="key expandable">${key}:</span>
          <span class="type">${Array.isArray(value) ? 'Array' : 'Object'}</span>
        `;
        const children = document.createElement('div');
        children.className = 'children collapsed';
        this.renderStateTree(value, children, currentPath);
        item.appendChild(children);

        item.querySelector('.key').addEventListener('click', () => {
          children.classList.toggle('collapsed');
        });
      } else {
        item.innerHTML = `
          <span class="key">${key}:</span>
          <input type="text" class="value-editor" value="${value}" data-path="${currentPath}" />
        `;

        item.querySelector('.value-editor').addEventListener('change', (e) => {
          this.port.postMessage({
            type: 'SET_STATE',
            path: currentPath,
            value: e.target.value
          });
        });
      }

      container.appendChild(item);
    }
  }
}
```

---

## Task 4.6: Migration Guides

### From jQuery

```markdown
# Migrating from jQuery to rnxJS

## Why Migrate?

| jQuery Pain Point | rnxJS Solution |
|-------------------|----------------|
| Manual DOM updates | Automatic reactivity |
| Spaghetti event handlers | Declarative data-bind |
| No state management | Built-in reactive state |
| Plugin dependency hell | Zero dependencies |

## Before & After

### Updating Text

**jQuery:**
```javascript
$('#username').text(user.name);
$('#email').text(user.email);

// When data changes, manually update again
user.name = 'New Name';
$('#username').text(user.name); // Easy to forget!
```

**rnxJS:**
```html
<span data-bind="user.name"></span>
<span data-bind="user.email"></span>
```
```javascript
state.user.name = 'New Name'; // UI updates automatically!
```

### Form Handling

**jQuery:**
```javascript
$('#myForm').on('submit', function(e) {
  e.preventDefault();
  const data = {
    name: $('#name').val(),
    email: $('#email').val()
  };
  // Validate manually...
  // Submit...
});
```

**rnxJS:**
```html
<input data-bind="form.name" data-rule="required" />
<input data-bind="form.email" data-rule="required|email" />
```
```javascript
// state.form is always in sync!
// state.errors has validation results!
submitForm(state.form);
```

## Migration Steps

1. Add rnxJS script tag (keep jQuery for now)
2. Create reactive state for your data
3. Replace `$('#id').text()` with `data-bind`
4. Replace `$('#id').val()` with `data-bind` on inputs
5. Remove jQuery event handlers, use native or rnxJS
6. Remove jQuery entirely when done
```

---

## Acceptance Criteria

### Must Have

- [ ] 15+ new components (DataTable, StatCard, Skeleton, etc.)
- [ ] CLI tool working: `npx create-rnx`
- [ ] Dashboard template with all components
- [ ] Component creation documentation
- [ ] Migration guide from jQuery

### Should Have

- [ ] Documentation site structure
- [ ] DevTools extension (basic version)
- [ ] Form wizard template
- [ ] Landing page template

### Nice to Have

- [ ] Video tutorials
- [ ] Interactive playground
- [ ] Component Storybook

---

## "Shit, That's It?" Moments

1. **Setup**: One script tag, done
2. **Reactivity**: `data-bind="user.name"`, done
3. **Validation**: `data-rule="required|email"`, done
4. **Dashboard**: `npx create-rnx my-dashboard`, done
5. **Table**: `<DataTable data="state.users" />`, done

---

## Definition of Done

1. New developer can build CRUD dashboard in <30 minutes
2. No questions about "how do I..." for common use cases
3. Zero configuration required for basic usage
4. All new components documented with examples
5. CLI creates working projects
6. At least 3 template options available
