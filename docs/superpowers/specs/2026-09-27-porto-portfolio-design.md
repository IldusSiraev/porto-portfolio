# Porto portfolio site — design

## Context

This repo (`porto-portfolio`) is a fork of `SirilCMS` (remote `upstream`), created for one client: a single-page portfolio site matching the `Porto` Tailwind HTML template (see `html_templates/Porto/index.html` in the upstream repo — not copied into this repo, kept only as the visual reference during design).

Deployment model: this fork runs as its own standalone instance (own Postgres, own `apps/web`/`apps/admin`, own `.env`/`PAYLOAD_SECRET`) on the client's VPS/domain. It is not part of a shared multi-tenant SirilCMS instance. Framework updates flow one way, on demand: `git fetch upstream && git merge upstream/main`.

All new block/theme contracts are added generically (not client-specific naming) so they remain mergeable with upstream and reusable if this fork's improvements are ever upstreamed.

## Section-to-block mapping

| Porto section | Block | Change |
|---|---|---|
| Hero | `hero` / `default` | none — reuse as-is |
| Features (3 cards) | `features` / `default` | none — reuse as-is |
| Portfolio (numbered list) | `portfolio-grid` / new variant `list` | add variant; extend shared `items` field with optional `description` |
| Education (card grid) | new type `education` | add |
| Quote / skills statement | new type `statement` | add |
| Experience (3-column table) | new type `experience` | add |
| Brands (logo grid) | `gallery` / `default` | none — reuse as-is |
| Testimonials | `testimonials` / `default` | none — reuse as-is |
| Footer (social icons) | `contact` | extend with optional `socialLinks` |

## Block contract changes (`packages/blocks-definitions/src/registry.ts`)

Append-only; no existing field/variant renamed or removed.

### `portfolio-grid` — extend

- Add `description` (richtext, optional) to the existing `items` object-array subfields (used by both variants, so `default`'s cards can optionally show a caption too).
- Add variant `{ id: 'list', name: 'Нумерованный список', layout: 'list', fields: [title, items] }` reusing the same `items` field name/shape as `default` (title, image, link, description) — Vue variant component just lays it out differently (numbered rows) and ignores `image`/`link` when absent.

### `education` — new block type

```
{ type: 'education', name: 'Образование', variants: [
  { id: 'default', name: 'Сетка', layout: 'grid', fields: [
    { name: 'title', type: 'text', label: {ru:'Заголовок', en:'Title'} },
    { name: 'subtitle', type: 'text', label: {ru:'Подзаголовок', en:'Subtitle'} },
    { name: 'items', type: 'object-array', label: {ru:'Места', en:'Places'}, maxItems: 12, subfields: [
      { name: 'period', type: 'text', label: {ru:'Период', en:'Period'}, required: true },
      { name: 'description', type: 'richtext', label: {ru:'Описание', en:'Description'} },
      { name: 'linkText', type: 'text', label: {ru:'Текст ссылки (необяз.)', en:'Link text (optional)'} },
      { name: 'linkUrl', type: 'link', label: {ru:'Ссылка (необяз.)', en:'Link (optional)'} },
    ] },
  ] },
]}
```

### `statement` — new block type

```
{ type: 'statement', name: 'Тезис', variants: [
  { id: 'default', name: 'Центр', layout: 'center', fields: [
    { name: 'quote', type: 'richtext', label: {ru:'Крупный текст', en:'Statement'}, required: true },
    { name: 'body', type: 'richtext', label: {ru:'Пояснение', en:'Supporting text'} },
  ] },
]}
```

### `experience` — new block type

```
{ type: 'experience', name: 'Опыт работы', variants: [
  { id: 'default', name: 'Список', layout: 'list', fields: [
    { name: 'title', type: 'text', label: {ru:'Заголовок', en:'Title'} },
    { name: 'subtitle', type: 'text', label: {ru:'Подзаголовок', en:'Subtitle'} },
    { name: 'entries', type: 'object-array', label: {ru:'Записи', en:'Entries'}, maxItems: 20, subfields: [
      { name: 'company', type: 'text', label: {ru:'Компания', en:'Company'}, required: true },
      { name: 'position', type: 'text', label: {ru:'Должность', en:'Position'} },
      { name: 'year', type: 'text', label: {ru:'Год', en:'Year'} },
    ] },
  ] },
]}
```

### `contact` — extend

Add optional `socialLinks` object-array after existing fields:

```
{ name: 'socialLinks', type: 'object-array', label: {ru:'Соцсети', en:'Social links'}, maxItems: 8, subfields: [
  { name: 'platform', type: 'select', label: {ru:'Платформа', en:'Platform'}, options: [
    {value:'twitter',label:'Twitter'}, {value:'facebook',label:'Facebook'}, {value:'instagram',label:'Instagram'},
    {value:'dribbble',label:'Dribbble'}, {value:'codepen',label:'Codepen'}, {value:'email',label:'Email'}, {value:'other',label:'Other'},
  ] },
  { name: 'url', type: 'link', label: {ru:'Ссылка', en:'URL'}, required: true },
] }
```

## Theme `porto`

New entry in `THEMES` (`packages/blocks-definitions/src/registry.ts`):

```
{ id: 'porto', name: 'Porto (light neutral)',
  tokens: { '--c-primary': '#374151', '--c-bg': '#f3f4f6', '--c-text': '#111827', '--radius': '.375rem' },
  blocks: { ...all(), 'form-block': { enabled: true } } }
```

`apps/web/themes/porto/tokens.css`:

```css
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700;800&display=swap');
:root { --c-primary: #374151; --c-bg: #f3f4f6; --c-text: #111827; --radius: .375rem; }
body { font-family: 'Poppins', system-ui, sans-serif; color: var(--c-text); background: var(--c-bg); }
a { color: var(--c-primary); }
```

## Vue components (`apps/web/app/blocks/<type>/*.vue`)

New files, following the existing pattern (e.g. `apps/web/app/blocks/testimonials/Default.vue`) — inline `:style` from `getTheme(id).tokens`, no `<style>`/`<script>` blocks inside `<template>` for dynamic CSS (use `useHead` if needed):

- `apps/web/app/blocks/portfolio-grid/List.vue`
- `apps/web/app/blocks/education/Default.vue`
- `apps/web/app/blocks/statement/Default.vue`
- `apps/web/app/blocks/experience/Default.vue`
- `apps/web/app/blocks/contact/Default.vue` — extend to render `socialLinks` if present (icon + link row)

Missing/unknown variant still falls back to `Placeholder.vue` per existing `resolveVariant` behavior — no crash risk during incremental rollout.

## Migration & types

```
pnpm --filter @siril/admin migrate:generate
pnpm --filter @siril/admin migrate
pnpm --filter @siril/admin generate:types
```

## Seed content

New `Sites` record: client domain, `theme: 'porto'`. One page with sections in Porto's order: hero → features → portfolio-grid(list) → education → statement → experience → gallery → testimonials → contact(with socialLinks). Content is placeholder text/images until the client provides real copy — flagged clearly in the admin as draft.

## Verification

```
pnpm typecheck && pnpm lint && pnpm test
pnpm dev:all
```
Manually check every section renders on `http://localhost:3000` for the seeded page, and that the `porto` theme's CSS/tokens apply (background, font, radius).

## Out of scope

- Multi-tenant hosting, domain-based site resolution beyond what already exists in `apps/web/server/utils/site-resolve.ts`.
- Upstreaming these blocks back into `SirilCMS` — left as a future decision.
- Pixel-perfect port of every Tailwind utility class from the original template; visual fidelity is theme-token-driven like all other SirilCMS themes.
