# AGENTS.md

Decisions and verification for this app. `INSTRUCTIONS.md` lives one level up,
in the Rust PWA template: this crate was cut from it, and every rule there
applies here. What follows is what is specific to this game.

## Shape

- `src/card.rs` — the deck: `Rank`, `Suit`, `Card`, the two-character storage
  code, and the printed short form. No `web-sys`, so `cargo test` runs it.
- `src/game.rs` — Klondike: the deal, the move grammar, the score, the undo
  history, the hint heuristic, the save format, and **every word the page
  shows**. Also `web-sys`-free.
- `src/ui.rs` — the browser layer, compiled for `wasm32` only: layout geometry,
  the elements, pointer and keyboard handling, timers, storage and the service
  worker. It decides nothing about the game.
- `src/ui.html`, `src/style.css`, `src/service-worker.js`, `build.rs`,
  `assets/icon.svg` — the shell, the styles, the worker, the site writer and the
  icon, as the template describes them.

## Decisions

**Difficulty is the deal's rules, not a hidden dice roll.** Klondike has no
difficulty knob in its rules, so the three settings are the two that change how
the game plays: how many cards the stock turns over, and how many times the
waste may be turned back. Easy is draw-one unlimited; Medium is draw-three
unlimited; Hard is draw-three with three passes. The menu names the rule beside
the label (`Turn three cards at a time, with three passes.`), because "Hard" on
its own tells the reader nothing they can act on.

**Scoring is a choice of three, and "No score" is a real one.** Standard is the
classic table (waste→column 5, →foundation 10, foundation→column −15, turning a
card 5, recycling the waste −100 at draw-one and −20 at draw-three, floored at
zero). Vegas is a 52-coin stake, five coins a card, one pass. Off is the game and
the clock with no arithmetic. A player who does not care about points should not
have a number in their header pretending otherwise.

**Options apply to the next deal.** The menu's selects are a request, not a
command: changing difficulty mid-game would either discard the game (a
data-loss bug wearing a settings menu) or silently keep the old rules under a new
label. They are stored separately from the game, so they outlive "New game" and
survive the page being evicted from memory.

**One transaction per mutation.** Every move, draw and recycle takes a snapshot
first; undo is then correct by construction rather than by remembering, per
feature, what it changed. The history is capped at 500 entries — a snapshot is
seven small vectors and five numbers, so the cap is a bound, not a policy.

**Every deal is a seed.** "Restart deal" is honest because the same seed deals
the same game, and a save is a seed plus the piles rather than 52 card objects.

**A saved game is checked, not trusted.** `Game::from_stored` refuses a save
unless every card appears exactly once, every code is a real card, every
foundation is an ascending single-suit run, and the won flag agrees with the
foundations. A corrupt `localStorage` entry must never be able to stop the app
from starting, and a half-valid one must never put one card in two piles.

**Cards are placed with `left`/`top`, not `transform`.** A transform transition
is compositor-driven: a renderer that does not advance animations leaves every
card at the position it was first drawn at, which is a board that looks empty and
works. `left`/`top` animate on the main thread, so the card is in the right place
even at frame zero and the movement between frames is the decoration rather than
the fact. The board also carries `.instant` for its first paint, so the deal
arrives instantly instead of flying in from the corner.

**The board is its own stacking context (`isolation: isolate`).** A card's
`z-index` is what keeps a fanned column in order, and those numbers run into the
hundreds. Without the isolation a card paints over the menu sheet and over the
victory dialog, both of which are deliberately above the board.

**One listener per board, not one per card.** A press is read off the element it
landed on; that element carries its pile and its position in the pile. Cards are
never rebuilt — a render moves them and rewrites their description — so a drag in
progress survives a repaint, and the transition has an element to travel with.

**The menu is a checkbox.** Opening it depends on `change` on a checkbox whose
label is the button, and the scrim and the Done control are labels for the same
box. A panel opened only by a `click` listener does not open on a phone; this
family has learned that once already.

**A tap is a pointer gesture, not a `click`.** `pointerdown` records a press,
`pointermove` past a slop threshold turns it into a drag, `pointerup` decides
between the two, and `pointercancel` abandons it. `click` is handled only for
keyboard activation (`event.detail === 0`), because handling both paths would
move every card twice.

## Verification

```bash
cargo test --locked                 # 45 tests: the rules, the save format, the shell
cargo clippy --locked --all-targets -- -D warnings
cargo clippy --locked --lib --target wasm32-unknown-unknown -- -D warnings
```

Then both build steps, in order, and check `dist/` — see `README.md`. The built
site was also driven in a headless browser against the real page: draw, undo,
hint, tap-to-move, drag-to-move, double-tap-to-foundation, the menu's difficulty
and scoring change, the automatic finish through to the victory dialog, the
keyboard, and the service worker registering and taking control. That is not a
committed test — the browser is not part of the toolchain here — and it is
written down so the next reader knows what "verified" covered.

What is *not* covered by a test: the layout's geometry. Card sizes and fan
offsets depend on the box the board gets, and the only honest check is to look at
it at a few sizes.

## Sharing the card format with Spider Solitaire

`src/card.rs` and the `.card` block of `src/style.css` are also carried by
[spider_solitaire](https://github.com/bot-git-ai/spider_solitaire). The two are
separate repositories, so nothing enforces that they agree; the rule is that a
change to either belongs in both. The deck, the codes and the printed faces are
the shared surface; the piles, the scoring and the layout are not.
