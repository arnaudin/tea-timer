# Spec: collapse the prompt and the options into one shared tray

Ported from the `tea-timer` project (sibling app, same design language). Applies to
`breathe/index.html`. Small, self-contained, CSS-led.

## Problem

Right now the bottom of the page stacks three things: the grounding prompt, the
controls, and the settings. The prompt only has something to say during a session,
and the settings are only actionable before one. So in every state, one of the two
is dead weight — and because the prompt's height depends on how long its line is,
the orb above it resizes as lines rotate.

## Change

Put the prompt and the settings in the **same grid cell**, below the controls. Only
one is visible at a time, cross-faded. The cell is always as tall as the taller of
the two, so nothing above it ever moves.

- **Idle / done** — settings visible, prompt hidden.
- **Running / paused** — prompt visible, settings hidden.

## Why it also tightens behaviour

`changed()` (~line 806) already resets a running session when a setting changes.
Leaving the settings reachable mid-session therefore only invites an accidental
reset. Hiding them makes the existing rule legible: to change the practice, press
Reset first.

## Mapping to the current markup

The three siblings inside `.bottom` (~lines 379-428) become two. Wrap the prompt and
the settings together, after the controls:

```html
<div class="bottom chrome">
  <div class="controls">…</div>

  <div class="tray">
    <p class="prompt" id="prompt" aria-live="polite">Choose a practice, then begin.</p>
    <section class="settings" id="settings" aria-label="Settings">…</section>
  </div>
</div>
```

`.promptwrap` is no longer needed; the tray does its centring.

## CSS

```css
.tray{ display:grid; align-items:center; width:100%; }
.tray > *{ grid-area:1/1; }

.settings{ transition:opacity .5s ease, visibility .5s; }
.app.session .settings{ opacity:0; visibility:hidden; }
.app:not(.session) .prompt{ opacity:0; visibility:hidden; }
```

Notes:

- `.app.session` already exists (~line 772, `running || paused`). Reuse it; do not add
  a new state class.
- Use `visibility`, not `display` or `hidden`. It preserves the reserved height, and it
  removes the hidden half from the tab order and the accessibility tree, so you cannot
  tab into a stepper mid-session.
- **Delete** the old dim-on-run rules (~lines 258-259):
  `.app.running .settings,.app.paused .settings{opacity:.5}` and the `:hover` companion.
  They fight the new rule.
- `.prompt` keeps its own longer `transition` and its `.fade` cross-fade. Add
  `visibility .9s` to it so it fades rather than snaps.
- Add `margin:0 auto` to `.prompt` now that `.promptwrap` is gone.

## Also do this

Pausing should stop the prompt rotation, otherwise the next rotating line overwrites
whatever the pause state wrote within one interval. In `pause()` (~line 654), stop the
prompt timer alongside the phase timer.

## Short windows

Below ~400px tall, drop the tray outright during a session rather than reserving its
height — the pacer should take the space back:

```css
@media (max-height:400px){ .app.session .tray{ display:none } }
```

## Two things to decide

1. **The Wim Hof safety note** (~line 421) currently lives inside its settings panel,
   so this change hides it once a session starts. It is pre-flight guidance and is read
   before Begin, so hiding it is defensible — but it ends with "Stop if you feel unwell,"
   which is mid-session advice. Consider lifting the note out of the panel so it stays
   put, or folding that one sentence into the rotating prompt set for that mode.

2. **Focus mode.** `.app.focus .chrome` takes the whole bottom out of flow and hides it
   (~lines 101-105). The tray sits inside `.chrome`, so it inherits that correctly and
   needs no special case. Confirm the reveal-on-input path still shows the right half of
   the tray for the current state.

## Acceptance

- Starting a session does not move the orb or the countdown by a single pixel.
  Screenshot before and after Begin and compare.
- Settings panels of different heights (Coherence vs Wim Hof, which is much taller)
  each reserve their own height correctly while idle.
- Tab from the controls during a session never lands on a settings control.
- Pause holds its prompt line indefinitely instead of being overwritten.
