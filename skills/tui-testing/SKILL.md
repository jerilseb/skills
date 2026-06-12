---
name: tui-testing
description: >
  Verify that terminal UI (TUI) apps work correctly — drive them in a real pty
  with tmux, assert rendering with framework test backends (ratatui TestBackend,
  Bubble Tea teatest, Textual Pilot, Ink testing-library), and avoid the classic
  pitfalls (timing races, invisible-color bugs that plain capture misses, leaked
  sessions). Use when asked to test, verify, debug, or smoke-check a TUI/curses/
  full-screen terminal app, or when a CLI needs interactive keyboard-driven testing.
---

# TUI Testing

A TUI needs a real terminal (a pty) — you cannot pipe stdin/stdout like a normal
CLI. Test in layers; each layer catches what the others are blind to:

1. **Unit/render tests with the framework's test backend** — fast, deterministic,
   CI-friendly. Tests logic and rendered frames, NOT the event loop or terminal I/O.
2. **tmux smoke test** — the real binary in a real pty: launches, takes input,
   renders, exits cleanly.
3. **Human eyeball / screenshot** — the only layer that catches "technically
   rendered but unreadable" (e.g. dark-gray-on-black). Both automated layers are
   blind here; ask the user to look when readability matters.

## Layer 1: framework test backends

Prefer these over screen-scraping when the codebase uses a known framework:

| Framework | Harness | Note |
|---|---|---|
| ratatui (Rust) | `TestBackend` | In-memory cell grid; `terminal.draw(...)` then assert on `backend().buffer()` (styled cells — can assert colors). `assert_buffer` / pair with `insta` snapshots. `TestBackend::resize()` for reflow tests. Drive app state directly; no key simulation. |
| Bubble Tea (Go) | `teatest` | Full program test: send messages/keys, golden-file the output. |
| Textual (Python) | `Pilot` (`app.run_test()`) | Async harness; `pilot.press(...)`, query widgets, snapshot via `pytest-textual-snapshot`. |
| Ink (React/JS) | `ink-testing-library` | `render()` → `lastFrame()` string asserts; `stdin.write()` for keys. |
| curses/urwid etc. | none standard | Fall back to tmux or pexpect. |

Keep game/app logic separable from I/O so plain unit tests cover state transitions
(movement, selection, validation) without any terminal at all.

## Layer 2: driving the real binary with tmux

```bash
# Launch detached at a FIXED size (default is 80x24 otherwise)
tmux kill-session -t t 2>/dev/null   # guard against leftovers first
tmux new-session -d -s t -x 120 -y 40 'the-app --flags'

# Send input: literals, named keys, ctrl combos
tmux send-keys -t t 'j' 'j' Enter
tmux send-keys -t t Up Down Left Right C-c Escape

# Read the screen
tmux capture-pane -t t -p        # plain text
tmux capture-pane -t t -p -e     # WITH ANSI escapes — needed for any style assert

# Resize mid-run to test reflow
tmux resize-window -t t -x 60 -y 20

# Exit checks
tmux send-keys -t t q
tmux has-session -t t 2>/dev/null || echo "exited"

# Always clean up
tmux kill-session -t t 2>/dev/null
```

Synchronization: there is no "rendered" signal. Poll for expected text instead of
fixed sleeps where possible:

```bash
for i in $(seq 1 20); do
  tmux capture-pane -t t -p | grep -q 'EXPECTED' && break
  sleep 0.25
done
```

## Pitfalls (each of these has burned a real session)

- **Time keeps passing between your commands.** A real-time app (tick loop, game,
  auto-refresh) advances during the seconds you spend thinking between shell
  invocations. Chain setup + input + capture into ONE shell command with tight
  sleeps. Turn-based TUIs don't have this problem.
- **Plain capture is color-blind.** `capture-pane -p` returns characters only — a
  test can pass while the text is invisible to a human (e.g. a ratatui Block title
  span with no explicit fg inherits the border's DarkGray, plus `.dim()` ⇒
  unreadable on dark themes; borders render first and title spans only PATCH those
  cells). Use `-p -e` and assert on the SGR codes (`38;5;N`) when style matters —
  and remember even that only proves which codes were emitted, not readability.
- **tmux is itself a terminal emulator in the middle.** Its own `$TERM`
  (`screen`/`tmux-256color`) may drop or translate truecolor, key encodings, mouse
  events. You test "app under tmux", not "app under the user's terminal" — a
  user-reported rendering bug may not reproduce byte-for-byte.
- **Key coalescing.** Rapid `send-keys` can land within one app tick; two queued
  direction keys may race the event loop. Space the sends or verify the app queues
  input.
- **Exit assertions are weak.** When the app exits the pane dies with the final
  screen, so "what did it print on exit" and "did it restore the terminal (raw
  mode, alternate screen)" are hard to verify. `has-session` proves it ended, not
  that it ended cleanly. To capture exit output, launch via
  `'the-app; echo EXIT=$?; sleep 30'` so the pane survives.
- **Session leaks.** Failed runs leave sessions behind that answer later captures.
  `kill-session` before AND after; use unique session names per test.
- **Size assumptions.** Always pass `-x/-y`; assert the app's too-small fallback by
  launching tiny.

## Alternatives to tmux

- **expect / pexpect / node-pty** — "wait for prompt, send keys" scripting; good
  for wizard-style flows, weaker for full-screen layouts.
- **`script -qec 'app' /dev/null`** — quick pty wrapper when you only need the app
  to believe it has a tty, no interaction.

## Minimal verification checklist for a new/changed TUI

1. Unit tests for state logic (no terminal).
2. Test-backend render test: expected text present in the buffer.
3. tmux: launches at a known size, initial screen correct.
4. tmux: one input per supported interaction class actually changes the screen.
5. tmux: pause/quit/error states show, `q`/Ctrl-C exits, session ends.
6. `capture-pane -e`: critical text has an explicit, readable fg color.
7. Resize smaller than the layout minimum: graceful fallback, no panic.
