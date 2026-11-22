# Design language: one look for every app on the stack

Every app built on DIR.Lib and SdlVulkan.Renderer (the list is in `framework-consumers.md`) draws its controls from
the same set, so a reader who learns one app can read the others. The model is Windows 11 (WinUI / Fluent), but we
do not copy it one to one. `ui-changes.md` is the checklist for a single change. This page says what the controls
are and what each one means. The controls themselves are DIR.Lib's (`Layout.Builder`). An app that hand-rolls a
control this page names has a bug to file. Tracking issue: SharpAstro/.github#14.

## 1. Colour means something; neutral is grey

- **Controls at rest are neutral:** a grey fill from the palette (`ControlFill`), never a tint. A blue "plain" button
  says nothing, because everything is blue.
- **The accent marks a few things, and only these:**
  - the one main action of a view
  - a selected chip or segment
  - an on switch or toggle tile
  - the filled part of a slider
  - the focus ring
- **Which colour the accent is belongs to the app.** An app may take the desktop's (AppShell#5) or keep its own
  (tianwen may keep a night-vision red). The language fixes where the accent goes, not its hue.
- **Taken from the desktop, the accent is the step matched to the chrome:** dark chrome takes the ramp's Light2,
  light chrome takes Dark1, as WinUI does. Never mix the accent into the background yourself; that is what makes a
  fill muddy.
- **Text and icons on an accent fill come from `UiPalette.InkOn(fill)`.** They are never picked by eye, so a pale
  accent gets dark text.
- **State colours stay clear of the accent.** Success is not the accent, Warn keeps away from the accent's hue
  (tianwen#2025), and a severity always carries a mark as well as a colour.

## 2. Light and dark are both first-class

- Every control, page and capture is designed for a **light** and a **dark** palette together, plus any palette an
  app adds (pilot's Night). Dark first with light patched on later is not done.
- An app follows the desktop's app mode by default. A switch to override it is allowed, not required.
- Hover, pressed and ring shades are **relative to the fill**: a step lighter on dark, a step darker on light. They
  are never one fixed colour.
- In light mode, the backdrop behind a document or image is not white, so a white sheet keeps its edge.

## 3. One kind of control per job

The kind tells the reader what a press does before they press it.

| Kind | Looks | Means |
| --- | --- | --- |
| **Accent button** | accent fill, `InkOn` text | the one main action of a view. At most one per view |
| **Standard button** | grey fill, thin ring | any other action |
| **Subtle button** | no fill; lights on hover | chrome and toolbar actions: back, more `…`, pin, close |
| **Hyperlink** | link-coloured text, no fill | navigation; carries `↗` when it leaves the app |
| **Chip** | fully round; accent when on, grey with ring when off | a filter; a click picks one, Shift/Ctrl adds or removes |
| **Segmented** | one bar of segments, the chosen one accent | one of a few (2 to 5) |
| **Switch** | a track and knob; accent track when on | on/off for a setting (DIR.Lib#156) |
| **Toggle tile** | an icon tile, accent when on, label under it | a quick on/off with an icon |
| **Split button** | two parts with a divider: the action, then `›` | an action or toggle plus its details |
| **Setting row** | a whole card row, trailing `⌄`, `›` or `↗` | expand in place, go deeper, or leave the app |
| **Dropdown** | grey box with ring and `⌄` | one of many |
| **Slider** | accent fill up to the thumb, grey after | a value in a range |

## 4. Marks

- **`›` means there is more behind it.** A button or row that opens a menu, a page or a panel carries a trailing
  chevron. A control that acts at once carries none.
- **`⌄`** expands in place (a setting row, a dropdown). **`↗`** leaves the app (a browser, another program).
- An icon-only control always has its word as a tooltip.

## 5. Shape and edges

- **Corner radius:** 4 on controls (`UiMetrics.ControlRadius`), 6 to 8 on raised surfaces (toasts, menus, flyouts,
  cards; `UiMetrics.CornerRadius`), and fully round (half the height) on a chip, a switch track and a slider thumb.
- **A thin ring, not a shadow, on controls:** a 1px edge (`ControlEdge`) a step off the fill. An accent fill needs
  no ring. Shadows are for raised surfaces only.
- **One height per row:** buttons, chips, segments, dropdowns and switches on a row share `UiMetrics.ButtonHeight`
  and one vertical middle.

## 6. States

Every control is drawn in each state it can reach, in light and in dark:

- rest
- hover
- pressed
- focus: an accent ring, 2px, outside the control's edge
- selected or on, where it applies
- disabled: flat and dim with no hover, and the reason as its tooltip (`Node.Disabled(reason)`)

A state set by an authority (a node, a device) shows the authority's answer, not the click: a switch flips when the
change is confirmed.

## 7. Done means captured

A new or changed control is done when CPU-rendered tests draw it in every state on a light and a dark palette, each
with a light accent (a pale teal such as `#00e5d5`, which needs dark text) and a dark one (a deep blue such as
`#0063b1`), and check:

- text on every fill is 4.5:1 or better
- every ring and icon is 3:1 or better against what it is drawn on

The captures go in the PR.
