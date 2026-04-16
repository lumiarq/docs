---
title: Veil Template Engine
description: Server-side HTML templating with .veil files in LumiARQ
section: Digging Deeper
order: 8
draft: false
---

# Veil Template Engine

- [Introduction](#introduction)
- [File Location Convention](#file-location-convention)
- [Rendering Veil Templates](#rendering-veil-templates)
- [Template Syntax](#template-syntax)
  - [Escaped Output](#escaped-output)
  - [Unescaped HTML Output](#unescaped-html-output)
  - [Conditionals](#conditionals)
  - [Loops](#loops)
  - [Includes](#includes)
  - [Layouts and Blocks](#layouts-and-blocks)
- [Passing Data to Views](#passing-data-to-views)
- [Layouts and Partials](#layouts-and-partials)
- [Alpine.js Integration](#alpinejs-integration)
- [Caching Templates](#caching-templates)
- [Scaffold Commands](#scaffold-commands)

<a name="introduction"></a>
## Introduction

Veil is LumiARQ's server-side template engine. It processes `.veil` files — plain HTML files with a lightweight template syntax — and returns rendered HTML strings from your route handlers. Veil supports layouts, partials, conditionals, loops, and Alpine.js integration out of the box.

Veil templates are compiled to optimised JavaScript functions at build time (`lumis build`) and can be cached for near-zero per-request overhead in production. In development, templates are compiled on demand when they change.

<a name="file-location-convention"></a>
## File Location Convention

Veil templates live inside the `views/` folder of the module that owns them:

```
src/modules/
  Billing/
    views/
      invoices/
        index.veil
        show.veil
      partials/
        invoice-row.veil
  Auth/
    views/
      login.veil
      register.veil
  Shared/
    views/
      layouts/
        app.veil
        guest.veil
      partials/
        flash.veil
        nav.veil
```

Shared layouts and partials used across modules belong in `src/modules/Shared/views/`.

<a name="rendering-veil-templates"></a>
## Rendering Veil Templates

Import `renderView` from `@lumiarq/framework/veil` and call it from any route handler. The first argument is the template path (relative to `src/modules/`, without the `.veil` extension). The second argument is the data object passed to the template.

```typescript
// src/modules/Billing/http/handlers/list-invoices.handler.ts
import { defineHandler } from '@lumiarq/framework'
import { renderView } from '@lumiarq/framework/veil'
import { GetInvoicesQuery } from '../../logic/queries/get-invoices.query'

export const ListInvoicesHandler = defineHandler(async (ctx) => {
  const invoices = await GetInvoicesQuery.run(ctx.auth.userId)

  return renderView('Billing/views/invoices/index', {
    invoices,
    title: 'Your Invoices',
  })
})
```

`renderView` returns a `Response` with `Content-Type: text/html` and the rendered string as the body.

<a name="template-syntax"></a>
## Template Syntax

<a name="escaped-output"></a>
### Escaped Output

Use double curly braces to output a variable. The value is HTML-escaped automatically, preventing XSS:

```html
<h1>Hello, {{ user.name }}</h1>
<p>Your email: {{ user.email }}</p>
```

<a name="unescaped-html-output"></a>
### Unescaped HTML Output

Use triple curly braces when you intentionally want to render raw HTML — for example, pre-rendered rich text from a trusted source:

```html
<div class="prose">
  {{{ article.bodyHtml }}}
</div>
```

> **Warning** — Never use `{{{ }}}` with user-supplied input. Only use it for content you control.

<a name="conditionals"></a>
### Conditionals

```html
{% if invoice.status === 'paid' %}
  <span class="badge-green">Paid</span>
{% elseif invoice.status === 'overdue' %}
  <span class="badge-red">Overdue</span>
{% else %}
  <span class="badge-gray">Pending</span>
{% endif %}
```

<a name="loops"></a>
### Loops

Iterate over arrays with `{% for item in items %}`:

```html
<ul>
  {% for invoice in invoices %}
    <li>
      <a href="/billing/invoices/{{ invoice.id }}">
        {{ invoice.number }} — {{ invoice.totalFormatted }}
      </a>
    </li>
  {% endfor %}
</ul>
```

Access the loop index with the special `loop` variable:

```html
{% for item in items %}
  <tr class="{{ loop.index % 2 === 0 ? 'bg-gray-50' : '' }}">
    <td>{{ loop.index + 1 }}</td>
    <td>{{ item.name }}</td>
  </tr>
{% endfor %}
```

<a name="includes"></a>
### Includes

Pull in a partial template with `{% include %}`. The included template has access to the same data as the parent:

```html
{% include 'Shared/views/partials/flash' %}

<table>
  {% for invoice in invoices %}
    {% include 'Billing/views/partials/invoice-row' %}
  {% endfor %}
</table>
```

Pass extra data to a partial using `with`:

```html
{% include 'Shared/views/partials/nav' with { active: 'billing' } %}
```

<a name="layouts-and-blocks"></a>
### Layouts and Blocks

Extend a layout with `{% extends %}` and fill named content areas with `{% block %}`:

```html
{# src/modules/Billing/views/invoices/index.veil #}
{% extends 'Shared/views/layouts/app' %}

{% block title %}Invoices — {{ appName }}{% endblock %}

{% block content %}
  <h1>Your Invoices</h1>

  {% if invoices.length === 0 %}
    <p>No invoices yet.</p>
  {% else %}
    {% for invoice in invoices %}
      {% include 'Billing/views/partials/invoice-row' %}
    {% endfor %}
  {% endif %}
{% endblock %}
```

The layout template defines the `{% block %}` slots that child templates fill:

```html
{# src/modules/Shared/views/layouts/app.veil #}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}{{ appName }}{% endblock %}</title>
  <link rel="stylesheet" href="/app.css">
</head>
<body>
  {% include 'Shared/views/partials/nav' %}

  <main>
    {% block content %}{% endblock %}
  </main>

  {% block scripts %}{% endblock %}
  <script src="/app.js"></script>
</body>
</html>
```

A template can only extend one layout. Blocks can be nested — an inner template's `{% block %}` overrides the layout's default content.

<a name="passing-data-to-views"></a>
## Passing Data to Views

All data is passed as the second argument to `renderView`. The object is available as top-level variables inside the template:

```typescript
return renderView('Billing/views/invoices/show', {
  invoice,           // available as {{ invoice.id }}, {{ invoice.total }}, etc.
  customer,          // available as {{ customer.name }}
  appName: 'MyApp',  // available as {{ appName }}
})
```

There is no global template context injected automatically. If you need values available in every view (like `appName`, `currentUser`, or flash messages), pass them explicitly or use a view composer helper:

```typescript
// src/shared/view-data.ts
import { env } from '../bootstrap/env.js'

export function globalViewData(ctx: RequestContext) {
  return {
    appName: env.APP_NAME,
    currentUser: ctx.auth?.user ?? null,
    flash: ctx.session?.flash ?? {},
  }
}
```

```typescript
return renderView('Billing/views/invoices/index', {
  ...globalViewData(ctx),
  invoices,
})
```

<a name="layouts-and-partials"></a>
## Layouts and Partials

Layouts live in `src/modules/Shared/views/layouts/` by convention. A typical project has two layouts:

| Layout | Used for |
|---|---|
| `layouts/app.veil` | Authenticated pages (includes nav, sidebar) |
| `layouts/guest.veil` | Public/auth pages (login, register) |

Partials are reusable fragments included with `{% include %}`. Keep partials small and focused — one concern per file. The `partials/` subdirectory is just a convention; you can organise them however makes sense for your module.

<a name="alpinejs-integration"></a>
## Alpine.js Integration

Veil supports Alpine.js attributes natively. Alpine.js is optional and only needed when you want client-side interactivity on server-rendered pages.

### Setup

1. Install Alpine.js:

```bash
pnpm add alpinejs
```

2. Start Alpine hydration in your layout's client script:

```typescript
import { start } from '@lumiarq/framework/veil';

await start();
```

### Registering Components

Register Alpine components before calling `start`. Components defined here are available as `x-data="componentName"` in any Veil template:

```typescript
import { registerComponents, start } from '@lumiarq/framework/veil';

registerComponents({
  counter: { count: 0 },
  modal: () => ({ open: false, toggle() { this.open = !this.open } }),
  dropdown: () => ({ expanded: false }),
});

await start();
```

### Using Alpine in Templates

```html
<div x-data="modal">
  <button @click="toggle">Open modal</button>

  <div x-show="open" class="modal-overlay">
    <div class="modal-box">
      <button @click="toggle">Close</button>
      {% block modalContent %}{% endblock %}
    </div>
  </div>
</div>
```

### Troubleshooting

If you see "Alpine.js is not installed", install Alpine and rerun your app build:

```bash
pnpm add alpinejs
pnpm lumis build --target node
```

<a name="caching-templates"></a>
## Caching Templates

In production, Veil pre-compiles all templates to optimised functions. The compiled output lives in `storage/framework/views/`.

**Commands:**

```bash
pnpm lumis view:cache    # compile and cache all templates
pnpm lumis view:clear    # delete all compiled templates
```

`lumis build` always runs `view:cache` automatically before bundling. You do not need to call it manually in a standard build pipeline.

**Config option:**

Set `veil.cache` in `config/app.ts` to control template caching:

```typescript
export default {
  // ...
  veil: {
    cache: env.APP_ENV === 'production',  // compile templates to disk
  },
} satisfies AppConfig
```

When `veil.cache` is `false` (development), templates are compiled on demand whenever the file changes, giving you instant feedback without restarting the server.

<a name="scaffold-commands"></a>
## Scaffold Commands

Generate a new Veil view file with `lumis make:view`:

```bash
pnpm lumis make:view Billing invoices/index
pnpm lumis make:view Billing invoices/show
pnpm lumis make:view Shared layouts/app
pnpm lumis make:view Shared partials/nav
```

This creates `src/modules/<Module>/views/<name>.veil` with a basic stub that extends the default layout.

For email templates, use `make:view` with a `mail/` path:

```bash
pnpm lumis make:view Auth mail/welcome-email
```

