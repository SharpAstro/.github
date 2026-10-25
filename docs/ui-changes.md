# UI changes: what to check before calling one done

This applies to any change a person will see: a pixel GUI (DIR.Lib / SdlVulkan.Renderer: pilot, tianwen, the FITS
viewer), a terminal UI (Console.Lib), a web page (WebGl.Renderer). The rules are not about taste. Each one is a mistake
that has shipped, and the example in brackets is where it happened.

## 1. Look at it

A test that a property is set is not a test that it looks right. Before a UI change is done:

- **Render a capture and look at it.** Use the repo's capture path (pilot: `PILOT_TEST_CAPTURE`, `SessionsCapture`;
  tianwen: its renderer captures). Capture the real layout at a narrow width (1280) and a wide one (2000), with
  realistic content: long titles, long paths, many rows. *(pilot's tooltips: tested as "painted something", shipped as a
  strip as wide as the window with its end cut off.)*
- **Look at every palette** the view has (day, night, a lit or selected row) and every state it can be in (below).
- **Put the capture in the PR**, before and after, for any change to layout or colour. The reviewer looks at it.
  "Builds and tests pass" says nothing about what it looks like.

## 2. Contrast

- Body text is at least **4.5:1** against what it is drawn on, and large text (18 units or more, or bold 14) at least
  **3:1** (WCAG AA). Icons, control borders and focus rings are also at least **3:1**.
- Check it against the background it is actually on: hover, selected, focused, a tinted card, the night palette. A
  colour that passes on the panel can fail on the row-focus colour.
- Dim or faint text is for secondary information only: never for something the person has to act on.
- Colour never carries meaning alone. A state also has a word or a mark (✓ ✗ ●): red and green are not enough.
- Measure, don't guess. pilot has a WCAG luminance/contrast helper in `TranscriptPaneTests`, so use it in a test for
  any new colour pair.

## 3. Readability

- Sizes: body 13 to 14 units, secondary 11 to 12, nothing under 10.
- Text that can be longer than its box either **wraps** (where the box can grow) or is **cut with an ellipsis and
  carries its whole text on hover**. It is never cut silently. *(pilot's card titles and details: cut with no "…", and
  no hover text.)*
- A long word or path cannot wrap at a space: break it after its separators, or it overflows the box.
- Paragraphs stay under about 80 characters per line. Sentences say what happened and what to do, in the person's words
  ("Mark done anyway", not "Accept the read").
- Times say their zone, or are local with the full date and zone on hover. Numbers carry units.

## 4. Alignment and size

- Controls on one row **share one height and one middle** (pilot's header: every button is 28 units, its text centred).
  *(Sync, All repos and Autopilot each stood their own height beside a 28-unit `?`.)*
- Text on one row shares a baseline.
- Gaps, gutters and sizes are **named constants**, shared across views. A literal pixel count inside one view drifts
  from the next view's.
- Icons are centred in their box and drawn with a size. A star-sized child under a content-sized row gets no height.
  *(The Flights view switch drew its words at the top and its icons not at all.)*
- Check the arranged bounds in a test (heights equal, Y equal, nothing zero-sized), not only the tree.

## 5. Single source of truth

- **One model per value.** A count, a state or a label shown in two places reads one property, never two copies that
  can disagree.
- **The authority sets the state.** A switch shows what the node (or device, or server) says, not what the click
  hoped for. Set it from the answer.
- **One component, used everywhere.** A pill, a header row or a card is built once and called by every page. Seven
  hand-built headers drift into seven headers (pilot#231).
- **Colours and fonts come from the palette or style**, never a literal in a view, so a night palette or a theme
  changes all of it.
- **Shared library first.** A control two apps need belongs in DIR.Lib, AppShell or SdlVulkan.Renderer, with an issue
  there, not a copy in each app.

## 6. The right control

| The person wants to | Use | Not |
| --- | --- | --- |
| turn a setting on or off | a switch (track and knob) | a button that stays lit (DIR.Lib#156) |
| pick one of a few | a segmented control / button group | separate toggles |
| pick one of many | a dropdown or a searchable list | a long row of pills |
| filter by several | pills: a click picks one, **Shift**-click adds or removes | pills that only add |
| do something | a button, worded as a verb | a link |
| go somewhere | a link; one that leaves the app ends in ↗ | a button |
| do something that cannot be undone | a button with a confirm step in place | a bare button |
| see more of a long thing | fold/unfold with a caret, the whole row clickable | a caret as the only target |

## 7. Clutter

- Every element earns its place: if nothing can be done with it now, it is hidden, not greyed out forever.
- Nothing is said twice on one screen.
- A long list shows a count and the first few, with a way to the rest.
- Secondary detail goes behind a hover, a fold or a page, not into the main line.
- Ask what the person came to the screen to do, and make that the most visible thing on it. (A flight ready to merge
  says so at the top of its page, with Merge.)

## 8. Stability: nothing moves under the pointer

- **Reserve the space** for content that comes and goes (a link that may be absent, a status line), so the rows below
  never jump.
- **Never insert above what the pointer is over.** A list that updates live holds still while hovered and shows what
  arrived as a marker (pilot's cards panel). *(A new card pushed the cards down between aim and click.)*
- A button's label changing must not change its width enough to move its neighbours.

## 9. Interaction

- Hit targets are at least 24 units, and the whole row is clickable where that has an obvious meaning (expand, select,
  open).
- Every clickable thing has a hover state and the hand cursor. Every icon-only control has a tooltip.
- Keyboard: arrows move, Enter opens, Escape closes or goes back, and the focus is visible.
- A refused action says why where the action was (on the card, under the button), never only in a log.
- Selection follows conventions: a click selects one, Shift or Ctrl extends.
- Nothing blocks the render thread: no I/O, no locks, no waiting in a frame.

## 10. States to try

Empty · one item · many (100+) · very long text and paths · loading · error / refused · stale (a feed down, another
node unreachable) · narrow window · wide window · every palette · a remote or watch-only item (another node's, no
grant) · the item just finished or just removed while it is shown.

## For reviewers

On a PR that changes UI:
- Ask for the captures if they are missing.
- Go through sections 2 to 8 against them.
- Say which states from section 10 were not shown.

A UI change without a capture has not been reviewed for how it looks, only for how it is written.
