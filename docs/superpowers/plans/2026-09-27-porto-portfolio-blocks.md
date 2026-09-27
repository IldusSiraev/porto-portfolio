# Porto Portfolio Blocks/Theme Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the block types, block variant, theme, migration, Vue components and seed content needed to reproduce the Porto portfolio template's 9 sections (Hero, Features, Portfolio, Education, Quote/Statement, Experience, Brands, Testimonials, Contact/Footer) as a SirilCMS site.

**Architecture:** Extend the append-only `BLOCKS`/`THEMES` SSOT in `packages/blocks-definitions/src/registry.ts` (3 new block types, 1 new variant, 1 field addition, 1 new theme), regenerate the Payload schema via migration, add matching Vue block components under `apps/web/app/blocks`, then seed one `Sites`/`Pages` record with placeholder Porto content via a dedicated one-off script that does not touch the shared `seed.ts`.

**Tech Stack:** TypeScript, Payload 3 (Next.js 16, Postgres), Nuxt 4 (Vue 3), Vitest, pnpm workspaces.

**Spec:** [docs/superpowers/specs/2026-09-27-porto-portfolio-design.md](../specs/2026-09-27-porto-portfolio-design.md)

## Global Constraints

- Append-only: never rename or remove an existing block type, variant, or field — this fork tracks `upstream` (`git remote -v` shows `upstream = git@github.com:IldusSiraev/SirilCMS.git`) and merges from it later.
- Every new/changed field needs bilingual `label: { ru, en }` (enforced by `packages/blocks-definitions/src/registry.test.ts`, which iterates all of `BLOCKS`).
- Any change to `packages/blocks-definitions/src/registry.ts` block shapes requires a Payload migration (`pnpm --filter @siril/admin migrate:generate` → `migrate` → `generate:types`) — see CLAUDE.md "Commands".
- No `<style>`/`<script>` blocks inside a Vue `<template>` — hard `vite:vue` build error (CLAUDE.md "Known gotchas").
- Richtext fields get an automatic sibling `<name>Html` field via `withRichTextHtml` (`apps/admin/src/utils/with-richtext-html.ts`) — Vue components render `block.<name>Html` through `<RichText :value="..." />`, never the raw richtext value.
- Variant Vue filenames are `apps/web/app/blocks/<type>/<PascalCase variant id>.vue` — no manual registration (`apps/web/app/blocks/registry.ts` globs them).
- When a block field name (e.g. `items`) is reused across two variants of the same block type, both variants must declare **identical** subfields for it — `toPayloadBlockFields` in `packages/blocks-definitions/src/to-payload.ts:38` unions fields by `name` using a `Map`, so the last-declared variant's shape silently wins if they differ.

---

### Task 1: `portfolio-grid` — add `list` variant + `description` field

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts:86-95`
- Create: `packages/blocks-definitions/src/registry.porto.test.ts`

**Interfaces:**
- Produces: `getBlock('portfolio-grid')` now returns a `BlockDef` with `variants: [{id:'default',...}, {id:'list',...}]`, both variants' `items` object-array subfields = `['title','image','link','description']`.

- [ ] **Step 1: Write the failing test**

Create `packages/blocks-definitions/src/registry.porto.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { getBlock } from './registry'

describe('portfolio-grid — вариант list + description', () => {
  const block = getBlock('portfolio-grid')!
  it('есть варианты default и list', () => {
    expect(block.variants.map(v => v.id)).toEqual(['default', 'list'])
  })
  it('items — одинаковый набор subfields у обоих вариантов', () => {
    const names = (id: string) => block.variants.find(v => v.id === id)!.fields
      .find(f => f.name === 'items')!.subfields!.map(f => f.name)
    expect(names('default')).toEqual(['title', 'image', 'link', 'description'])
    expect(names('list')).toEqual(names('default'))
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `block.variants.map(...)` is `['default']`, not `['default', 'list']`.

- [ ] **Step 3: Implement — extend registry.ts**

In `packages/blocks-definitions/src/registry.ts`, replace the `portfolio-grid` entry (currently lines 86-95):

```ts
  { type: 'portfolio-grid', name: 'Портфолио', variants: [
    { id: 'default', name: 'Сетка', layout: 'grid', fields: [
      { name: 'title', type: 'text', label: { ru: 'Заголовок', en: 'Title' } },
      { name: 'items', type: 'object-array', label: { ru: 'Работы', en: 'Works' }, maxItems: 24, subfields: [
        { name: 'title', type: 'text', label: { ru: 'Название', en: 'Name' }, required: true },
        { name: 'image', type: 'image', label: { ru: 'Обложка', en: 'Cover' } },
        { name: 'link', type: 'link', label: { ru: 'Ссылка (необяз.)', en: 'Link (optional)' } },
        { name: 'description', type: 'richtext', label: { ru: 'Описание (необяз.)', en: 'Description (optional)' } },
      ] },
    ] },
    { id: 'list', name: 'Нумерованный список', layout: 'list', fields: [
      { name: 'title', type: 'text', label: { ru: 'Заголовок', en: 'Title' } },
      { name: 'items', type: 'object-array', label: { ru: 'Работы', en: 'Works' }, maxItems: 24, subfields: [
        { name: 'title', type: 'text', label: { ru: 'Название', en: 'Name' }, required: true },
        { name: 'image', type: 'image', label: { ru: 'Обложка', en: 'Cover' } },
        { name: 'link', type: 'link', label: { ru: 'Ссылка (необяз.)', en: 'Link (optional)' } },
        { name: 'description', type: 'richtext', label: { ru: 'Описание (необяз.)', en: 'Description (optional)' } },
      ] },
    ] },
  ] },
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS, including the existing `registry.test.ts` label check (new fields already have `ru`/`en` labels).

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(blocks): add portfolio-grid list variant + description field"
```

---

### Task 2: New block type `education`

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts` (insert after the `portfolio-grid` entry from Task 1)
- Modify: `packages/blocks-definitions/src/registry.porto.test.ts`

**Interfaces:**
- Produces: `getBlock('education')` → `BlockDef` with one `default` variant (`layout: 'grid'`), fields `title`, `subtitle`, `items[]{period, description, linkText, linkUrl}`.

- [ ] **Step 1: Write the failing test**

Append to `packages/blocks-definitions/src/registry.porto.test.ts`:

```ts
describe('education — новый блок', () => {
  it('есть с полями period/description/linkText/linkUrl', () => {
    const block = getBlock('education')!
    expect(block.variants[0]!.layout).toBe('grid')
    const items = block.variants[0]!.fields.find(f => f.name === 'items')!
    expect(items.subfields!.map(f => f.name)).toEqual(['period', 'description', 'linkText', 'linkUrl'])
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `getBlock('education')` is `undefined`, so `.variants` throws.

- [ ] **Step 3: Implement — add block to registry.ts**

Insert right after the `portfolio-grid` block entry:

```ts
  { type: 'education', name: 'Образование', variants: [
    { id: 'default', name: 'Сетка', layout: 'grid', fields: [
      { name: 'title', type: 'text', label: { ru: 'Заголовок', en: 'Title' } },
      { name: 'subtitle', type: 'text', label: { ru: 'Подзаголовок', en: 'Subtitle' } },
      { name: 'items', type: 'object-array', label: { ru: 'Места', en: 'Places' }, maxItems: 12, subfields: [
        { name: 'period', type: 'text', label: { ru: 'Период', en: 'Period' }, required: true },
        { name: 'description', type: 'richtext', label: { ru: 'Описание', en: 'Description' } },
        { name: 'linkText', type: 'text', label: { ru: 'Текст ссылки (необяз.)', en: 'Link text (optional)' } },
        { name: 'linkUrl', type: 'link', label: { ru: 'Ссылка (необяз.)', en: 'Link (optional)' } },
      ] },
    ] },
  ] },
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(blocks): add education block type"
```

---

### Task 3: New block type `statement`

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts` (insert after `education`)
- Modify: `packages/blocks-definitions/src/registry.porto.test.ts`

**Interfaces:**
- Produces: `getBlock('statement')` → `BlockDef` with one `default` variant (`layout: 'center'`), fields `quote` (richtext, required), `body` (richtext, optional).

- [ ] **Step 1: Write the failing test**

Append to `packages/blocks-definitions/src/registry.porto.test.ts`:

```ts
describe('statement — новый блок', () => {
  it('есть с полями quote (required) и body', () => {
    const block = getBlock('statement')!
    expect(block.variants[0]!.layout).toBe('center')
    const fields = block.variants[0]!.fields
    expect(fields.find(f => f.name === 'quote')).toMatchObject({ type: 'richtext', required: true })
    expect(fields.find(f => f.name === 'body')).toMatchObject({ type: 'richtext' })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `getBlock('statement')` is `undefined`.

- [ ] **Step 3: Implement — add block to registry.ts**

Insert right after the `education` block entry:

```ts
  { type: 'statement', name: 'Тезис', variants: [
    { id: 'default', name: 'Центр', layout: 'center', fields: [
      { name: 'quote', type: 'richtext', label: { ru: 'Крупный текст', en: 'Statement' }, required: true },
      { name: 'body', type: 'richtext', label: { ru: 'Пояснение', en: 'Supporting text' } },
    ] },
  ] },
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(blocks): add statement block type"
```

---

### Task 4: New block type `experience`

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts` (insert after `statement`)
- Modify: `packages/blocks-definitions/src/registry.porto.test.ts`

**Interfaces:**
- Produces: `getBlock('experience')` → `BlockDef` with one `default` variant (`layout: 'list'`), fields `title`, `subtitle`, `entries[]{company, position, year}`.

- [ ] **Step 1: Write the failing test**

Append to `packages/blocks-definitions/src/registry.porto.test.ts`:

```ts
describe('experience — новый блок', () => {
  it('есть с entries company/position/year', () => {
    const block = getBlock('experience')!
    expect(block.variants[0]!.layout).toBe('list')
    const entries = block.variants[0]!.fields.find(f => f.name === 'entries')!
    expect(entries.subfields!.map(f => f.name)).toEqual(['company', 'position', 'year'])
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `getBlock('experience')` is `undefined`.

- [ ] **Step 3: Implement — add block to registry.ts**

Insert right after the `statement` block entry:

```ts
  { type: 'experience', name: 'Опыт работы', variants: [
    { id: 'default', name: 'Список', layout: 'list', fields: [
      { name: 'title', type: 'text', label: { ru: 'Заголовок', en: 'Title' } },
      { name: 'subtitle', type: 'text', label: { ru: 'Подзаголовок', en: 'Subtitle' } },
      { name: 'entries', type: 'object-array', label: { ru: 'Записи', en: 'Entries' }, maxItems: 20, subfields: [
        { name: 'company', type: 'text', label: { ru: 'Компания', en: 'Company' }, required: true },
        { name: 'position', type: 'text', label: { ru: 'Должность', en: 'Position' } },
        { name: 'year', type: 'text', label: { ru: 'Год', en: 'Year' } },
      ] },
    ] },
  ] },
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(blocks): add experience block type"
```

---

### Task 5: `contact` — add `socialLinks`

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts:111-123` (exact line numbers shift after Tasks 1-4's insertions — locate by `type: 'contact'`)
- Modify: `packages/blocks-definitions/src/registry.porto.test.ts`

**Interfaces:**
- Produces: `getBlock('contact')` variant `default` gains field `socialLinks[]{platform(select), url(link, required)}`.

- [ ] **Step 1: Write the failing test**

Append to `packages/blocks-definitions/src/registry.porto.test.ts`:

```ts
describe('contact — socialLinks добавлено', () => {
  it('есть поле socialLinks с subfields platform/url', () => {
    const block = getBlock('contact')!
    const social = block.variants[0]!.fields.find(f => f.name === 'socialLinks')!
    expect(social.subfields!.map(f => f.name)).toEqual(['platform', 'url'])
    expect(social.subfields!.find(f => f.name === 'url')).toMatchObject({ required: true })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `social` is `undefined`, `.subfields` throws.

- [ ] **Step 3: Implement — extend contact block**

In `packages/blocks-definitions/src/registry.ts`, find the `contact` block's `default` variant fields array (ends with the `mapLink` field) and add `socialLinks` right after `mapLink`:

```ts
      { name: 'mapLink', type: 'link', label: { ru: 'Ссылка на карту', en: 'Map link' } },
      { name: 'socialLinks', type: 'object-array', label: { ru: 'Соцсети', en: 'Social links' }, maxItems: 8, subfields: [
        { name: 'platform', type: 'select', label: { ru: 'Платформа', en: 'Platform' }, options: [
          { value: 'twitter', label: 'Twitter' },
          { value: 'facebook', label: 'Facebook' },
          { value: 'instagram', label: 'Instagram' },
          { value: 'dribbble', label: 'Dribbble' },
          { value: 'codepen', label: 'Codepen' },
          { value: 'email', label: 'Email' },
          { value: 'other', label: 'Other' },
        ] },
        { name: 'url', type: 'link', label: { ru: 'Ссылка', en: 'URL' }, required: true },
      ] },
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(blocks): add socialLinks to contact block"
```

---

### Task 6: New theme `porto`

**Files:**
- Modify: `packages/blocks-definitions/src/registry.ts` (near `THEMES` export, after the `mono` const)
- Modify: `packages/blocks-definitions/src/registry.porto.test.ts`
- Create: `apps/web/themes/porto/tokens.css`

**Interfaces:**
- Consumes: `BLOCKS` (already imported in `registry.ts`), the local `all()` helper (`registry.ts:143`).
- Produces: `getTheme('porto')` → `ThemeDef` with the tokens below and every block in `BLOCKS` enabled.

- [ ] **Step 1: Write the failing test**

Append to `packages/blocks-definitions/src/registry.porto.test.ts` (add `BLOCKS` and `getTheme` to the existing import from `./registry`):

```ts
import { BLOCKS, getBlock, getTheme } from './registry'
```

```ts
describe('porto — новая тема', () => {
  it('токены заданы, все блоки включены', () => {
    const theme = getTheme('porto')
    expect(theme.id).toBe('porto')
    expect(theme.tokens).toMatchObject({
      '--c-primary': '#374151', '--c-bg': '#f3f4f6', '--c-text': '#111827', '--radius': '.375rem',
    })
    for (const b of BLOCKS) expect(theme.blocks[b.type]?.enabled).toBe(true)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @siril/blocks-definitions test -- registry.porto`
Expected: FAIL — `getTheme('porto')` falls back to `defaultTheme` (per `getTheme`'s `?? defaultTheme`), so `theme.id` is `'default'`, not `'porto'`.

- [ ] **Step 3: Implement — add porto theme**

In `packages/blocks-definitions/src/registry.ts`, after the `mono` const definition and before `export const THEMES: ThemeDef[] = [defaultTheme, mono]`:

```ts
const porto: ThemeDef = {
  id: 'porto', name: 'Porto (light neutral)',
  tokens: { '--c-primary': '#374151', '--c-bg': '#f3f4f6', '--c-text': '#111827', '--radius': '.375rem' },
  blocks: { ...all(), 'form-block': { enabled: true } },
}
```

Change the export line to:

```ts
export const THEMES: ThemeDef[] = [defaultTheme, mono, porto]
```

Create `apps/web/themes/porto/tokens.css`:

```css
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700;800&display=swap');
:root { --c-primary: #374151; --c-bg: #f3f4f6; --c-text: #111827; --radius: .375rem; }
body { font-family: 'Poppins', system-ui, sans-serif; color: var(--c-text); background: var(--c-bg); }
a { color: var(--c-primary); }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @siril/blocks-definitions test`
Expected: PASS (all `registry.porto.test.ts` describe blocks green, plus existing `registry.test.ts`/`resolve.test.ts`/`to-payload.test.ts` unaffected).

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-definitions/src/registry.ts apps/web/themes/porto/tokens.css packages/blocks-definitions/src/registry.porto.test.ts
git commit -m "feat(theme): add porto theme"
```

---

### Task 7: Payload migration + generated types

No new block/theme logic here — this task persists Tasks 1-6's `registry.ts` changes into the Postgres schema and regenerates `payload-types.ts`, both required before the seed script (Task 13) or admin UI can use the new fields.

**Files:**
- Create: `apps/admin/src/migrations/<generated-timestamp>_*.ts` (name chosen by the Payload CLI — do not hand-write it)
- Modify: `apps/admin/payload-types.ts` (regenerated, not hand-edited)

**Interfaces:**
- Consumes: `BLOCKS`/`THEMES` from `packages/blocks-definitions/src/registry.ts` (via `apps/admin/src/collections/pages.ts:59-62` and `apps/admin/src/collections/sites.ts`'s theme `select`).

- [ ] **Step 1: Ensure local Postgres is running**

Run: `pnpm dev:pg`
Expected: exits 0 (starts the dev Postgres container, or confirms an external one on `:5433` is reachable).

- [ ] **Step 2: Generate the migration**

Run: `pnpm --filter @siril/admin migrate:generate`
Expected: prints the created migration file path under `apps/admin/src/migrations/`.

- [ ] **Step 3: Inspect the generated migration**

Open the newly created file under `apps/admin/src/migrations/` and confirm it contains `CREATE TABLE`/`ALTER TABLE` statements referencing `pages_blocks_education`, `pages_blocks_statement`, `pages_blocks_experience`, and column additions on `pages_blocks_portfolio_grid_items` (`description`) and `pages_blocks_contact_social_links` (new table) — if any of these are missing, the corresponding registry.ts change from Tasks 1-5 did not save; re-check before continuing.

- [ ] **Step 4: Apply the migration**

Run: `pnpm --filter @siril/admin migrate`
Expected: exits 0, no pending migrations left (`pnpm --filter @siril/admin migrate` a second time reports "No migrations to run").

- [ ] **Step 5: Regenerate Payload types**

Run: `pnpm --filter @siril/admin generate:types`
Expected: `apps/admin/payload-types.ts` is rewritten; `git diff apps/admin/payload-types.ts` shows new types for `PagesBlockEducation`, `PagesBlockStatement`, `PagesBlockExperience`, and updated `PagesBlockPortfolioGrid`/`PagesBlockContact`.

- [ ] **Step 6: Typecheck admin**

Run: `pnpm --filter @siril/admin typecheck`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add apps/admin/src/migrations apps/admin/payload-types.ts
git commit -m "chore(migration): add education/statement/experience blocks, portfolio-grid description, contact socialLinks"
```

---

### Task 8: `portfolio-grid` Vue components — `Default.vue` description + new `List.vue`

**Files:**
- Modify: `apps/web/app/blocks/portfolio-grid/Default.vue`
- Create: `apps/web/app/blocks/portfolio-grid/List.vue`

**Interfaces:**
- Consumes: `block.items[].descriptionHtml` (from Task 1's `description` richtext field, exposed by `withRichTextHtml` as `descriptionHtml`), `getTheme` from `@siril/blocks-definitions`, global `mediaUrl` and `RichText` (both Nuxt auto-imports already used by sibling block components).
- Produces: `List.vue` is resolved by `apps/web/app/blocks/registry.ts:13` for `portfolio-grid`/`list` (`getBlockComponent('portfolio-grid', 'list')` → `./portfolio-grid/List.vue`).

No Vue component in this codebase has a dedicated unit test (verified: `apps/web` has no `*.test.ts`/`*.spec.ts` matching `block`) — rendering is checked via Nuxt's type-checked template compiler (`nuxt typecheck`) and, in Task 14, a manual browser pass. Steps below follow that convention instead of a fabricated test harness.

- [ ] **Step 1: Update `Default.vue` to render the optional description**

Replace the contents of `apps/web/app/blocks/portfolio-grid/Default.vue`:

```vue
<template>
  <section class="portfolio-block" :style="tokens">
    <h2 class="portfolio-title">{{ block.title }}</h2>
    <div class="portfolio-grid">
      <div v-for="(i, idx) in items" :key="idx" class="portfolio-card">
        <img v-if="i.imageSrc" :src="i.imageSrc" :alt="i.title || ''" loading="lazy" class="portfolio-img">
        <div class="portfolio-name">
          <a v-if="i.link" :href="i.link" :target="isExternal(i.link) ? '_blank' : undefined" :rel="isExternal(i.link) ? 'noopener' : undefined">{{ i.title }}</a>
          <span v-else>{{ i.title }}</span>
        </div>
        <div v-if="i.descriptionHtml" class="portfolio-description"><RichText :value="i.descriptionHtml" /></div>
      </div>
    </div>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
const items = computed(() =>
  (props.block.items ?? []).map((i: any) => ({
    ...i,
    imageSrc: mediaUrl(i.image),
  })),
)
const isExternal = (l: string) => typeof l === 'string' && l.startsWith('http')
</script>
<style>
.portfolio-block { padding: 2rem; max-width: 72rem; margin: 0 auto; }
.portfolio-title { text-align: center; margin-bottom: 1.5rem; color: var(--c-primary, #111); }
.portfolio-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; }
.portfolio-card { border-radius: .5rem; overflow: hidden; background: var(--c-surface, #f5f5f5); }
.portfolio-img { width: 100%; aspect-ratio: 4/3; object-fit: cover; }
.portfolio-name { padding: .75rem; font-weight: 600; }
.portfolio-description { padding: 0 .75rem .75rem; font-size: .875rem; opacity: .85; }
@media (max-width: 640px) { .portfolio-grid { grid-template-columns: 1fr; } }
</style>
```

- [ ] **Step 2: Create `List.vue`**

```vue
<template>
  <section class="portfolio-list-block" :style="tokens">
    <h2 class="portfolio-list-title">{{ block.title }}</h2>
    <ol class="portfolio-list">
      <li v-for="(i, idx) in items" :key="idx" class="portfolio-list-row">
        <span class="portfolio-list-index">{{ String(idx + 1).padStart(2, '0') }}</span>
        <div class="portfolio-list-body">
          <a v-if="i.link" :href="i.link" :target="isExternal(i.link) ? '_blank' : undefined" :rel="isExternal(i.link) ? 'noopener' : undefined" class="portfolio-list-name">{{ i.title }}</a>
          <span v-else class="portfolio-list-name">{{ i.title }}</span>
          <div v-if="i.descriptionHtml" class="portfolio-list-description"><RichText :value="i.descriptionHtml" /></div>
        </div>
      </li>
    </ol>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
const items = computed(() => props.block.items ?? [])
const isExternal = (l: string) => typeof l === 'string' && l.startsWith('http')
</script>
<style>
.portfolio-list-block { padding: 2rem; max-width: 72rem; margin: 0 auto; }
.portfolio-list-title { text-align: center; margin-bottom: 1.5rem; color: var(--c-primary, #111); }
.portfolio-list { list-style: none; padding: 0; display: flex; flex-direction: column; gap: 1.5rem; }
.portfolio-list-row { display: flex; gap: 1.5rem; align-items: baseline; }
.portfolio-list-index { font-size: 1.5rem; opacity: .5; }
.portfolio-list-name { font-size: 1.25rem; font-weight: 600; }
.portfolio-list-description { margin-top: .5rem; font-size: .875rem; opacity: .85; }
@media (max-width: 640px) { .portfolio-list-row { flex-direction: column; gap: .5rem; } }
</style>
```

- [ ] **Step 3: Typecheck web**

Run: `pnpm --filter @siril/web typecheck`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add apps/web/app/blocks/portfolio-grid/Default.vue apps/web/app/blocks/portfolio-grid/List.vue
git commit -m "feat(web): render portfolio-grid description + list variant"
```

---

### Task 9: `education` Vue component

**Files:**
- Create: `apps/web/app/blocks/education/Default.vue`

**Interfaces:**
- Consumes: `block.title`, `block.subtitle`, `block.items[].{period, descriptionHtml, linkText, linkUrl}` (Task 2's contract).

- [ ] **Step 1: Create the component**

```vue
<template>
  <section class="education-block" :style="tokens">
    <h2 class="education-title">{{ block.title }}</h2>
    <p v-if="block.subtitle" class="education-subtitle">{{ block.subtitle }}</p>
    <div class="education-grid">
      <div v-for="(i, idx) in items" :key="idx" class="education-card">
        <h4 class="education-period">{{ i.period }}</h4>
        <div v-if="i.descriptionHtml" class="education-description"><RichText :value="i.descriptionHtml" /></div>
        <a v-if="i.linkUrl" :href="i.linkUrl" :target="isExternal(i.linkUrl) ? '_blank' : undefined" :rel="isExternal(i.linkUrl) ? 'noopener' : undefined" class="education-link">{{ i.linkText || i.linkUrl }}</a>
      </div>
    </div>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
const items = computed(() => props.block.items ?? [])
const isExternal = (l: string) => typeof l === 'string' && l.startsWith('http')
</script>
<style>
.education-block { padding: 2rem; max-width: 72rem; margin: 0 auto; }
.education-title { text-align: center; margin-bottom: .5rem; color: var(--c-primary, #111); }
.education-subtitle { text-align: center; margin-bottom: 1.5rem; opacity: .8; }
.education-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 1rem; }
.education-card { padding: 1.25rem; border-radius: .5rem; background: var(--c-surface, #f5f5f5); }
.education-period { font-weight: 600; margin-bottom: .5rem; }
.education-link { display: inline-block; margin-top: .5rem; font-weight: 600; }
@media (max-width: 640px) { .education-grid { grid-template-columns: 1fr; } }
</style>
```

- [ ] **Step 2: Typecheck web**

Run: `pnpm --filter @siril/web typecheck`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/web/app/blocks/education/Default.vue
git commit -m "feat(web): add education block component"
```

---

### Task 10: `statement` Vue component

**Files:**
- Create: `apps/web/app/blocks/statement/Default.vue`

**Interfaces:**
- Consumes: `block.quoteHtml` (required), `block.bodyHtml` (optional) — Task 3's contract.

- [ ] **Step 1: Create the component**

```vue
<template>
  <section class="statement-block" :style="tokens">
    <div class="statement-quote"><RichText :value="block.quoteHtml" /></div>
    <div v-if="block.bodyHtml" class="statement-body"><RichText :value="block.bodyHtml" /></div>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
</script>
<style>
.statement-block { padding: 3rem 2rem; max-width: 60rem; margin: 0 auto; text-align: center; }
.statement-quote { font-size: 2rem; line-height: 1.3; opacity: .4; margin-bottom: 1.5rem; }
.statement-body { font-size: 1rem; color: var(--c-text, #111); }
</style>
```

- [ ] **Step 2: Typecheck web**

Run: `pnpm --filter @siril/web typecheck`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/web/app/blocks/statement/Default.vue
git commit -m "feat(web): add statement block component"
```

---

### Task 11: `experience` Vue component

**Files:**
- Create: `apps/web/app/blocks/experience/Default.vue`

**Interfaces:**
- Consumes: `block.title`, `block.subtitle`, `block.entries[].{company, position, year}` — Task 4's contract. The three columns render the same `entries` array three times; order is guaranteed identical since all three `v-for`s iterate the same source array.

- [ ] **Step 1: Create the component**

```vue
<template>
  <section class="experience-block" :style="tokens">
    <h2 class="experience-title">{{ block.title }}</h2>
    <p v-if="block.subtitle" class="experience-subtitle">{{ block.subtitle }}</p>
    <div class="experience-columns">
      <div class="experience-column">
        <h6 class="experience-heading">Company</h6>
        <p v-for="(e, idx) in entries" :key="'company-' + idx">{{ e.company }}</p>
      </div>
      <div class="experience-column">
        <h6 class="experience-heading">Position</h6>
        <p v-for="(e, idx) in entries" :key="'position-' + idx">{{ e.position }}</p>
      </div>
      <div class="experience-column">
        <h6 class="experience-heading">Year</h6>
        <p v-for="(e, idx) in entries" :key="'year-' + idx">{{ e.year }}</p>
      </div>
    </div>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
const entries = computed(() => props.block.entries ?? [])
</script>
<style>
.experience-block { padding: 2rem; max-width: 72rem; margin: 0 auto; }
.experience-title { margin-bottom: .5rem; color: var(--c-primary, #111); }
.experience-subtitle { margin-bottom: 1.5rem; opacity: .8; }
.experience-columns { display: flex; flex-wrap: wrap; justify-content: space-between; gap: 2rem; }
.experience-column { display: flex; flex-direction: column; gap: 1rem; min-width: 10rem; }
.experience-heading { text-transform: uppercase; opacity: .6; font-size: .8rem; margin-bottom: .5rem; }
</style>
```

- [ ] **Step 2: Typecheck web**

Run: `pnpm --filter @siril/web typecheck`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/web/app/blocks/experience/Default.vue
git commit -m "feat(web): add experience block component"
```

---

### Task 12: `contact` Vue component — render `socialLinks`

**Files:**
- Modify: `apps/web/app/blocks/contact/Default.vue`

**Interfaces:**
- Consumes: `block.socialLinks[].{platform, url}` — Task 5's contract.

- [ ] **Step 1: Update the component**

Replace the contents of `apps/web/app/blocks/contact/Default.vue`:

```vue
<template>
  <section class="contact-block" :style="tokens">
    <h2 class="contact-title">{{ block.title }}</h2>
    <ul class="contact-list">
      <li v-if="block.email"><a :href="'mailto:' + block.email">{{ block.email }}</a></li>
      <li v-if="block.phone"><a :href="'tel:' + block.phone">{{ block.phone }}</a></li>
      <li v-if="block.address">{{ block.address }}</li>
      <!-- hours заменил workHours (см. docs/developer.md §«Версионирование контрактов блоков») —
           workHours как фоллбэк на переходный период, для контента, ещё не тронутого миграцией. -->
      <li v-if="block.hours || block.workHours">{{ block.hours || block.workHours }}</li>
      <li v-if="block.mapLink"><a :href="block.mapLink" target="_blank" rel="noopener">Открыть на карте</a></li>
    </ul>
    <div v-if="socialLinks.length" class="contact-social">
      <a v-for="(s, idx) in socialLinks" :key="idx" :href="s.url" target="_blank" rel="noopener" class="contact-social-link">{{ s.platform }}</a>
    </div>
  </section>
</template>
<script setup lang="ts">
import { getTheme } from '@siril/blocks-definitions'
const props = defineProps<{ block: any; themeId: string }>()
const theme = getTheme(props.themeId)
const tokens = Object.fromEntries(Object.entries(theme.tokens))
const socialLinks = computed(() => props.block.socialLinks ?? [])
</script>
<style>
.contact-block { padding: 2rem; max-width: 72rem; margin: 0 auto; }
.contact-title { text-align: center; margin-bottom: 1.5rem; color: var(--c-primary, #111); }
.contact-list { list-style: none; padding: 0; max-width: 32rem; margin: 0 auto; display: flex; flex-direction: column; gap: .5rem; }
.contact-social { display: flex; justify-content: center; gap: 1rem; margin-top: 1.5rem; }
.contact-social-link { text-transform: capitalize; font-weight: 600; }
</style>
```

- [ ] **Step 2: Typecheck web**

Run: `pnpm --filter @siril/web typecheck`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/web/app/blocks/contact/Default.vue
git commit -m "feat(web): render contact socialLinks"
```

---

### Task 13: Seed script for the Porto site

Deliberately a separate script from `apps/admin/src/scripts/seed.ts` (the shared platform demo seed) so this client-specific content never conflicts with `git merge upstream/main` on that shared file.

**Files:**
- Create: `apps/admin/src/scripts/seed-porto.ts`
- Modify: `apps/admin/package.json:10` (add `seed:porto` script, next to `seed`)
- Modify: `package.json:8` (add root `seed:porto` script, next to `seed`)

**Interfaces:**
- Consumes: Payload local API (`getPayload`, `payload.create`) exactly as `seed.ts` does; collections `sites`/`pages` with the `sections` blocks shapes produced by Tasks 1-5 (post-migration from Task 7).

- [ ] **Step 1: Add the npm scripts**

In `apps/admin/package.json`, add after the `"seed"` line:

```json
    "seed:porto": "node --import ./scripts/ts-loader/index.mjs node_modules/payload/bin.js run src/scripts/seed-porto.ts --disable-transpile",
```

In the root `package.json`, add after the `"seed"` line:

```json
    "seed:porto": "pnpm --filter @siril/admin seed:porto",
```

- [ ] **Step 2: Create `apps/admin/src/scripts/seed-porto.ts`**

```ts
import { getPayload } from 'payload'
import config from '../../payload.config'

function richTextParagraph(text: string) {
  return {
    root: {
      type: 'root',
      children: [{
        type: 'paragraph',
        children: [{ type: 'text', text, detail: 0, format: 0, mode: 'normal', style: '', version: 1 }],
        direction: 'ltr' as const, format: '' as const, indent: 0, version: 1,
      }],
      direction: 'ltr' as const, format: '' as const, indent: 0, version: 1,
    },
  }
}

async function seedPorto() {
  const payload = await getPayload({ config })
  try {
    const existing = await payload.find({ collection: 'sites', where: { slug: { equals: 'porto' } }, overrideAccess: true })
    if (existing.totalDocs > 0) {
      console.log('seed-porto: skip (site "porto" exists)')
      return
    }
    console.log('seed-porto: creating…')

    const site = await payload.create({
      collection: 'sites',
      overrideAccess: true,
      data: {
        name: 'Porto Portfolio',
        slug: 'porto',
        domain: process.env.NUXT_PUBLIC_SITE_DOMAIN ?? 'localhost:3000',
        theme: 'porto',
      },
    })

    await payload.create({
      collection: 'pages',
      overrideAccess: true,
      data: {
        site: site.id,
        title: 'Home',
        slug: 'home',
        _status: 'published',
        sections: [
          {
            blockType: 'hero',
            variant: 'default',
            title: 'Senior Backend Developer',
            subtitle: richTextParagraph('I have a passion for software. I enjoy creating tools that make life easier for people.'),
            buttonText: 'Hire me',
            buttonLink: '#contact',
          },
          {
            blockType: 'features',
            variant: 'default',
            title: 'Why work with me',
            items: [
              { value: 'High experience' },
              { value: 'Useful sandboxes' },
              { value: 'Success side projects' },
            ],
          },
          {
            blockType: 'portfolio-grid',
            variant: 'list',
            title: 'Portfolio',
            items: [
              { title: 'Demo API Generator', description: richTextParagraph('A dummy data free and documented API generator to facilitate the process of testing the front-end portion of projects.') },
              { title: 'Demo API Generator', description: richTextParagraph('A dummy data free and documented API generator to facilitate the process of testing the front-end portion of projects.') },
              { title: 'Demo API Generator', description: richTextParagraph('A dummy data free and documented API generator to facilitate the process of testing the front-end portion of projects.') },
            ],
          },
          {
            blockType: 'education',
            variant: 'default',
            title: 'Education',
            subtitle: 'Below is a summary of the places I studied',
            items: Array.from({ length: 6 }, () => ({
              period: '2015 – 2016',
              description: richTextParagraph('Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.'),
              linkText: 'See the place here',
              linkUrl: '#',
            })),
          },
          {
            blockType: 'statement',
            variant: 'default',
            quote: richTextParagraph('Please do not measure your skills in percentages!'),
            body: richTextParagraph('In my many years of experience, I use Laravel for backend projects and Vue.js for front-end projects. I’m an avid programmer, so I create designs based on the weekend Figma design.'),
          },
          {
            blockType: 'experience',
            variant: 'default',
            title: 'Experience',
            subtitle: 'Below is a summary of the places I worked',
            entries: Array.from({ length: 5 }, () => ({
              company: 'Massa Fames / New York',
              position: 'Junior Front-End Developer',
              year: '2016',
            })),
          },
          {
            blockType: 'gallery',
            variant: 'default',
            title: 'Brands',
          },
          {
            blockType: 'testimonials',
            variant: 'default',
            title: 'Testimonial',
            items: Array.from({ length: 4 }, () => ({
              quote: richTextParagraph('Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.'),
              author: 'Stephan Clark',
              role: 'CEO at EarlyBird',
            })),
          },
          {
            blockType: 'contact',
            variant: 'default',
            title: 'Contact',
            email: 'hello@example.com',
            socialLinks: [
              { platform: 'twitter', url: '#' },
              { platform: 'dribbble', url: '#' },
              { platform: 'facebook', url: '#' },
              { platform: 'codepen', url: '#' },
              { platform: 'email', url: 'mailto:hello@example.com' },
              { platform: 'instagram', url: '#' },
            ],
          },
        ],
      },
    })

    console.log('seed-porto: done.')
  } finally {
    await payload.db?.destroy?.()
  }
}

await seedPorto()
```

- [ ] **Step 3: Run the seed script**

Run: `pnpm seed:porto`
Expected: prints `seed-porto: creating…` then `seed-porto: done.` with no thrown errors (a validation error here means a section's field names don't match the migrated schema — recheck Tasks 1-5/7).

- [ ] **Step 4: Verify in the admin UI**

Run: `pnpm --filter @siril/admin dev`, open `http://localhost:3001/admin`, log in as `owner@demo.ru` / `admin123`, open the "Porto Portfolio" site's "Home" page.
Expected: the "Home" page shows 9 sections in the order listed above, each with content, no empty/error blocks.

- [ ] **Step 5: Commit**

```bash
git add apps/admin/src/scripts/seed-porto.ts apps/admin/package.json package.json
git commit -m "feat(seed): add seed-porto script with placeholder Porto content"
```

---

### Task 14: Full verification pass

**Files:** none (verification only)

- [ ] **Step 1: Run the full test suite**

Run: `pnpm test`
Expected: all workspaces pass, including `registry.porto.test.ts` from Tasks 1-6.

- [ ] **Step 2: Run lint**

Run: `pnpm lint`
Expected: PASS, no errors in the new/changed `.ts`/`.vue` files.

- [ ] **Step 3: Run typecheck across the monorepo**

Run: `pnpm typecheck`
Expected: PASS (`@siril/admin`, `@siril/blocks-definitions` via `tsc --noEmit`; `@siril/web` via `nuxt typecheck`).

- [ ] **Step 4: Start the dev servers**

Run: `pnpm dev:all`
Expected: `apps/web` on `:3000`, `apps/admin` on `:3001`, both start without errors.

- [ ] **Step 5: Manual browser check**

Open `http://localhost:3000/` (the seeded "porto" site's `home` page — resolved via `pickSiteByHost`, works standalone since it is the only `Sites` doc besides any pre-existing `default` demo site; if both exist, browse via the exact `domain` value set in Task 13, or temporarily set it to `localhost:3000` and remove/rename the other site's domain). Confirm, in order: Hero renders title/subtitle/button; 3 feature cards; numbered Portfolio list (not a grid) with descriptions; Education grid of 6 cards; centered Statement text; 3-column Experience list with matching row counts; Brands section renders (empty gallery, no broken image icons); Testimonials grid of 4 cards; Contact block shows email + 6 social links. Confirm the `porto` theme's light-gray background, `Poppins` font, and `.375rem` rounded corners are visibly applied (open browser DevTools → check `<html>`/`<body>` computed `font-family` is `Poppins`).

- [ ] **Step 6: Push**

```bash
git push origin main
```
