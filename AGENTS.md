# `itui` — AI Agent & Engineering Guide (`AGENTS.md`)

This guide provides AI agents and developers with the language rules, module API surface, architectural patterns, and build/test commands needed to work with or build applications using **`itui`** (a Bubble Tea & Lipgloss-inspired Terminal UI toolkit for the `iyi` programming language).

---

## 1. Core `iyi` Language Rules (R-1 .. R-5)

`iyi` is an LLVM-backed, AI-first language designed around **separate compilation**:

- **R-1 (Module Compilation Unit & DAG):**
  - Every `.iyi` file is a compilation unit whose first line must declare `module <path>` matching its file path relative to the project root (e.g., `src/style/theme.iyi` -> `module src/style/theme`).
  - Path segments must match `[a-z][a-z0-9]*` with single `_` between words so paths map reversibly to module names (`src/components/progress_bar` <-> `Src::Components::ProgressBar`).
  - Compiling a module reads only its dependencies' exported declarations (`.iyimod`), never their bodies.
- **R-2 (Explicit Type Signatures):**
  - Every exported (`pub`) function (`pub def`), `pub struct` method, and `getter` field must declare **explicit parameter and return types**.
- **R-2b (Consumer-Side Imports):**
  - `import X::{A, B}`: Brings only the named exported symbols into unqualified scope.
  - `import X::*`: Brings all `pub` symbols from `X` into unqualified scope.
  - `pub import X`: Re-exports module `X` (`src/tui.iyi` re-exports the entire toolkit this way).
- **R-2c (Definition-Site Typing):** Every fully annotated `def` is type-checked at its definition site even if uncalled.
- **R-3 (Closed Types & Orphan Rule):** Classes/structs cannot be reopened in other modules. Use `trait` and `impl Trait for Type` inside the module defining either the trait or the type.
- **Key `iyi` Prelude Conventions:**
  - `!` is reserved for error-union propagation (`read(path)!`). Method names cannot end with `!` (use `sorted`, `sort_in_place` instead of `sort!`).
  - `String#size` and `each_char` operate on UTF-8 code points (characters); `String#bytesize`, `[]`, `[start, count]`, and `byte_index` operate on bytes. `String#split(Char)` takes a `Char` separator (`'\n'`, `' '`).
  - Integer division `//` truncates toward zero.

---

## 2. `itui` Module Map & API Surface

All modules are reachable via a single `import itui/src/tui::*` (after `iyi get github.com/canermastan/itui --as itui`) or selectively with `import itui/src/...::{...}`:

| Module Path | Exported Types (`pub`) | Responsibility (SRP) |
|---|---|---|
| `src/tui` | `TUI` (+ `pub import` of all submodules) | One-liner interactive prompts (`TUI.select`, `TUI.multi_select`, `TUI.input`, `TUI.confirm`, `TUI.theme`, `TUI.window_size`) |
| `src/style/theme` | `Theme` | Official `iyi` mascot palette (`Theme.iyi`: `#E11D48` crimson, `#FACC15` gold, `#FDA4AF` soft rose, `#2A0A12` dark crimson surface, `#F8FAFC` white) |
| `src/style/color` | `Color`, `ColorKind` | `Color.hex("#E11D48")`, `Color.rgb(r, g, b)`, `Color.ansi256(n)`, and standard 16 ANSI colors |
| `src/style/border` | `Border` | `Border.single`, `Border.rounded`, `Border.double`, `Border.thick`, `Border.none` box drawing |
| `src/style/style` | `Style`, `Edges` | Lipgloss-style immutable fluent builder (`bold`, `italic`, `underline`, `dim`, `foreground`, `background`, `border`, `border_foreground`, `padding`, `margin`, `width`, `align`, `render`, `Style.join_horizontal`, `Style.join_vertical`) |
| `src/style/ansi` | `Ansi` | SGR escape wrapping (`Ansi.wrap`) with nested reset preservation, ANSI stripping (`Ansi.strip`), and UTF-8 visible width (`Ansi.visible_width`, `Ansi.pad_visible`) |
| `src/core/terminal` | `Terminal` | `Terminal.with_raw_mode(mouse: Bool)`, `Terminal.enable_mouse`, `Terminal.disable_mouse`, `Terminal.enter_alt_screen`, `Terminal.exit_alt_screen`, `Terminal.size`, `Terminal.read_raw_chunk` |
| `src/core/mouse` | `MouseEvent`, `MouseButton`, `MouseAction` | SGR 1006 mouse parser (`MouseEvent.parse_sgr`), click/wheel predicates (`is_click?`, `is_right_click?`, `is_wheel_up?`, `is_wheel_down?`), and hit-testing (`in_rect?`) |
| `src/core/key` | `KeyEvent`, `KeyKind`, `KeyBinding` | Buffered `KeyEvent.read`, `KeyEvent.parse`, `KeyEvent.parse_all` (paste-safe), keyboard & mouse predicates (`is_up?`, `is_click?`, `is_wheel_up?`, `is_quit?`, `matches?`), and `KeyBinding` |
| `src/core/window` | `WindowSize` | `WindowSize.detect`, `split_horizontal(left_percent, reserved, min_left, min_right)`, `body_height`, `scale_width`, `scale_height` |
| `src/core/focus` | `FocusManager` | Multi-panel `Tab` / `Shift+Tab` focus cycling (`handle_key`, `focused?`, `apply_panel_style`, `format_title`) |
| `src/components/progress_bar` | `ProgressBar` | Clamped `0..100%` progress bar with label and custom glyphs (`with_percent`, `with_width`, `with_label`, `with_colors`, `with_glyphs`, `render`) |
| `src/components/viewport` | `Viewport` | Scrollable window for long text/logs with proportional scrollbar (`handle_key`, `append_line`, `scroll_up`, `scroll_down`, `page_up`, `page_down`, `goto_top`, `goto_bottom`, `scroll_percent`, `render`) |
| `src/components/filter_list` | `FilterList` | Live searchable list highlighting matched substrings in gold `#FACC15` (`handle_key`, `filtered_items`, `selected_item`, `selected_original_index`, `render`) |
| `src/components/text_area` | `TextArea` | Multi-line UTF-8 text editor with line numbers and 2D cursor navigation (`TextArea.from_text`, `handle_key`, `insert_text`, `insert_newline`, `backspace`, `value`, `render`) |
| `src/components/table` | `Table`, `TableColumn` | Column-aligned data table with keyboard row selection and windowing (`handle_key`, `selected_row`, `with_rows`, `with_cursor`, `render`) |
| `src/components/tabs` | `Tabs` | Horizontal tab switcher supporting `Left`/`Right` and `1..9` (`handle_key`, `next_tab`, `prev_tab`, `active_title`, `render`) |
| `src/components/modal` | `Modal` | Auto word-wrapped confirmation/alert dialog with centered screen overlay (`handle_key`, `show`, `hide`, `render`, `render_over(background_view)`) |
| `src/components/spinner` | `Spinner` | `Spinner.dots`, `Spinner.line`, `Spinner.pulse` animated indicators (`tick`, `render`) |
| `src/components/help_bar` | `HelpBar` | Renders active `KeyBinding` arrays into a clean bottom status bar (`render`) |

---

## 3. Quick Recipes for Agents

### A. Layer 1: One-Liner Interactive Prompts (No Model / Loop Required)

```crystal
module app/cli

import itui/src/tui::{TUI}

name      = TUI.input("Project name:", "iyi-app")
target    = TUI.select("Target arch:", ["x86_64-linux", "aarch64-darwin", "wasm32-wasi"])
features  = TUI.multi_select("Components:", ["ProgressBar", "Viewport", "FilterList", "TextArea"])
confirmed = TUI.confirm("Start build?")
```

### B. Layer 2: Elm Architecture (`Model` -> `update` -> `view`) Full-Screen Apps

All `itui` components are **immutable `struct`s**. Every `handle_key(key)` or `with_*` call returns a new updated value:

```crystal
module app/studio

import itui/src/tui::*

pub struct AppModel
  getter viewport : Viewport
  getter progress : ProgressBar
  getter focus : FocusManager
  getter quit : Bool

  def initialize(@viewport : Viewport, @progress : ProgressBar, @focus : FocusManager, @quit : Bool = false)
  end

  def update(key : KeyEvent) : AppModel
    return AppModel.new(@viewport, @progress, @focus, true) if key.is_quit?
    next_focus, _ = @focus.handle_key(key)
    next_vp = @viewport.handle_key(key)
    AppModel.new(next_vp, @progress, next_focus, false)
  end
end
```

---

## 4. Build, Context Inspection & Verification Commands

```powershell
# Inspect the exact exported declaration surface of all imports (no bodies)
iyi mod context examples/demo.iyi

# Fast front-end typecheck and R-1..R-5 rule verification
iyi check examples/demo.iyi
iyi check -f json examples/demo.iyi

# Apply compiler-suggested fixes automatically
iyi fix examples/demo.iyi

# Compile and run the demo
iyi run examples/demo.iyi

# Run the full unit and edge-case test suite (4 test files, 39 tests)
iyi test tests
```

---

## 5. Engineering & Code Quality Standards

1. **Comment Discipline:** Do not write comments on self-explanatory functions, getters, or obvious lines. Add concise comments only for non-obvious algorithms or invariants (e.g., UTF-8 codepoint vs. byte indexing, nested ANSI reset restoration, unbuffered raw IO chunk reads).
2. **Strict Immutability:** Components must never mutate internal state in place; every state transition returns a new `struct`.
3. **Box Geometry Invariant:** Whenever a multi-line component (`Style`, `Table`, `Viewport`, `TextArea`, `Modal`) renders, every emitted line must have an identical `Ansi.visible_width(line)`.
