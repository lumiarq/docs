# Veil Template Engine

## Alpine.js Integration

Veil supports Alpine.js attributes natively. Alpine.js is optional and only needed
when you want client-side interactivity.

### Setup

1. Install Alpine.js in your app:

```bash
pnpm add alpinejs
```

2. Start hydration in your layout script:

```ts
import { start } from '@lumiarq/framework/veil';

await start();
```

### Register Components

Register components before calling start:

```ts
import { registerComponents, start } from '@lumiarq/framework/veil';

registerComponents({
  counter: { count: 0 },
  modal: () => ({ open: false }),
});

await start();
```

### Troubleshooting

If you see "Alpine.js is not installed", install Alpine and rerun your app build.

```bash
pnpm add alpinejs
```
