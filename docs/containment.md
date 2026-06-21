# `.c-panel` containment

`contain: strict` on [`.c-panel`](../src/golden-card.css) is shorthand for `contain: size layout paint style;`.

`container-type: size` already forces size + layout + style containment on its own, regardless of what `contain` says. The only thing `strict` adds on top of that here is **paint containment**.

## size containment

The box's own size is computed ignoring its content, as if it were empty.

- If no definite width/height reaches the box, it collapses to 0×0 and everything inside becomes invisible (zero area), even though it's still in the DOM. This is why `.c-panel` sets `width: 100%; height: 100%;` — without it, the box would collapse since it has no other source of definite size.

## layout containment

- `.c-panel` becomes the containing block for all `position: absolute`/`fixed` descendants — a `fixed` child inside anchors to the panel's box, not the viewport.
- Establishes a block formatting context: floats can't escape it, margins don't collapse through it.

## paint containment

- Behaves like an implicit `overflow: clip` at the border edge — nothing inside (box-shadows, outlines, negative-margin children, deliberately-overflowing tooltips/dropdowns) can paint outside the box.
- Payoff: the browser can skip painting the subtree entirely when the panel is off-screen, since it knows nothing escapes it.

## style containment

- Scopes CSS counters (`counter-increment`/`counter-reset`) and `quotes` so they don't leak in/out of the panel. Rarely relevant unless numbering things across page boundaries.

## Net effect

Combined with `container-type: size`, `.c-panel` gets full layout/paint isolation and real rendering-engine payoff (skippable layout/paint work), at the cost of:

- every consumer of `.c-panel` must supply a definite box or its content disappears
- anything meant to overflow the panel gets silently clipped
- `fixed`/`absolute` descendants reanchor to the panel instead of the viewport