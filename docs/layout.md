# Layout system

A three-role layout mini-DSL built on CSS Grid + containment. The whole vocabulary is **viewport → panel → content**, configured through custom properties. See [`golden-card.css`](../src/golden-card.css).

## The DSL

Three roles, each with one job and one forbidden behaviour:

| Role | Class | Owns | Forbidden from |
|------|-------|------|----------------|
| **Viewport** | `.c-viewport` | The screen box + the track grid panels snap into | — |
| **Panel** | `.c-panel` | Nothing about its own size — it's a *slot* the app fills | Growing/shrinking from its content |
| **Content** | `.c-content` | The inside: padding, max-width, alignment, scrolling | Affecting the panel's outer size |

The guiding rule: **the app decides panel sizes; a panel never leaks its content's size outward; content lives inside that sealed box.**

### Tokens

Values are passed as custom properties, so markup reads like a layout description:

| Token | On | Meaning | Default |
|-------|-----|---------|---------|
| `--cols` | viewport | `grid-template-columns` | `1fr` |
| `--rows` | viewport | `grid-template-rows` | `1fr` |
| `--gap` | viewport | gap between panels | `0` |
| `--col` | panel | `grid-column` (which slot) | `auto` |
| `--row` | panel | `grid-row` (which slot) | `auto` |
| `--align` | content | `place-content` (alignment of the inner box) | `start stretch` |
| `--pad` | content | inner padding | `0` |
| `--max-w` | content | max inline-size of the child column | `none` |
| `--pad-top` | content | top breathing room (used by `.distinct`) | `2rem` |

### Usage

```html
<div class="c-viewport" style="--cols: 240px 1fr; --rows: auto 1fr;">
  <div class="c-panel" style="--col: 1; --row: 1 / -1">
    <div class="c-content distinct">…sidebar…</div>
  </div>
  <div class="c-panel" style="--col: 2; --row: 2">
    <div class="c-content prose"><article>…main…</article></div>
  </div>
</div>
```

> The viewport must be a grid container (`display: grid`). Panels must land in **definite** tracks (`fr`, fixed, `minmax`) — never `auto` — or a size-contained panel collapses to 0 (see [containment.md](./containment.md)).

## Content variants

Both variants pin content to the **top** and **center it horizontally**, differing only in how the child is sized. They assume a **single child element** acting as the column — `--max-w` targets `.c-content > *`.

### `.prose`

```css
.c-content.prose { --align: start safe center; --max-w: 90ch; }
```

A centered reading column capped at `90ch`, anchored to the top, scrolling when it outgrows the panel. The *column* is centered; text inside reads left-aligned (add `text-align: center` to the child if you want centered text too).

### `.distinct`

```css
.c-content.distinct { --align: start safe center; --pad: var(--pad-top, 2rem) 1rem 0; }
```

Content at its natural width, centered horizontally, anchored to the top with `2rem` of breathing room (override via `--pad-top`). No width cap.

## CSS theory it relies on

### Grid as the single coordinate space

The viewport is the only element that defines tracks. Panels are pure grid items placed by `--col`/`--row`; content is a **single-cell grid** whose only job is to position one box via `place-content`. This means one property (`place-content`) drives every alignment, and there are no nested track systems to reason about.

### Containment makes the panel a sealed slot

`.c-panel` uses `contain: strict` + `container-type: size`. Size containment is what lets "the app decides the size" be *enforced* rather than hoped for — the panel computes its size ignoring its content, so content can never push back. The full breakdown (size / layout / paint / style containment and their consequences) lives in [containment.md](./containment.md).

Two consequences shape the rest of the system:

- **Definite-size requirement.** A size-contained panel with no definite size collapses to 0. Hence `.c-panel` stretches into its grid slot (`align-self`/`justify-self: stretch` on a `100% × 100%` box), and tracks must be definite.
- **Paint clipping.** `contain: strict` brings paint containment, which clips overflow at the border edge — like an implicit `overflow: clip`. That's why content needs its **own** `overflow: auto`: the panel won't scroll, so without it long content is silently clipped away.

### `container-type: size` for content-responsive panels

Because the panel is size-contained *and* a named container (`container-name: panel`), content inside can respond to the **panel's** size via `@container panel (...)`, independent of the viewport. Resizing a panel (breakpoint, app logic, future drag handle) reflows its content against the panel, not the screen.

### Safe alignment to avoid the overflow traps

Both axes deliberately avoid centering overflowing content:

- **Vertical is `start`, never `center`.** Vertically centering content taller than the panel clips the top and makes it unreachable by scroll. `start` keeps the top reachable and scrolls down.
- **Horizontal is `safe center`.** If a child is ever wider than the panel, plain `center` clips both edges with no scroll-back; `safe` falls back to `start` exactly in that overflow case, and behaves like `center` otherwise.

### Logical properties

Sizing uses `inline-size`/`block-size`/`max-inline-size` rather than `width`/`height`, so the same rules hold under a different writing mode or direction.