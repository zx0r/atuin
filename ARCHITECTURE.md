# Architectural Specification: Semantic Theme Normalization & TUI Rendering Architecture

**Subsystem:** Shell History Search & Inspection Engine (`Atuin`)  
**Workspace:** `~/x/dev/own/atuin`  
**Branch:** `feat/semantic-borders-and-preview`  
**Author:** Principal Architect & Senior UX/UI Engineer  
**Status:** Approved & Implemented  

---

## 1. Problem Topology & Architectural Defect

### 1.1 The Symptom
In terminal environments utilizing custom chromatic design systems (e.g., Environment X Dark Tier), Atuin's interactive TUI (`atuin search -i`) exhibited jarring visual anomalies in full-screen/standard mode (`Compactness::Full`):
- All window borders (search input frame, history list container, preview panel) were rendered in blinding pure white (`#ffffff`).
- The command preview text in the preview panel rendered in pure white without respect to theme foreground or syntax tokens.
- CLI companion utilities such as Antigravity CLI (`agy`) shared the identical uncalibrated foreground color, destroying visual hierarchy and eye comfort.

### 1.2 Upstream Root Cause Analysis (Code-Level)
Inspection of `crates/atuin/src/command/client/search/interactive.rs` revealed that while Atuin provided a semantic theme engine (`Theme`, `Meaning`, and TOML definitions in `~/.config/atuin/themes/`), the interactive search renderer failed to bind theme styles to structural Ratatui widgets:

```rust
// Upstream implementation in interactive.rs:
// 1. Input box border:
input.block(
    Block::default()
        .borders(...)
        .border_type(BorderType::Rounded)
        // MISSING: .border_style(...)
)

// 2. Preview box border & content:
Paragraph::new(command).block(
    Block::default()
        .borders(...)
        .border_type(BorderType::Rounded)
        // MISSING: .border_style(...)
)
// MISSING: Paragraph text style or syntax classification
```

Because `.border_style(...)` and `.style(...)` were omitted on these blocks:
1. Ratatui defaulted to `Style::default()`, emitting ANSI escape sequence `\x1b[39m` (`Color::Reset`).
2. Terminal emulators resolve `Color::Reset` to their global **Default Foreground Color (`foreground`)**.
3. In `~/.config/ghostty/config.ghostty`, a legacy override `foreground = #ffffff` hijacked all `Color::Reset` outputs, forcing pure white onto borders, preview text, and unstyled CLI tools (`agy`).

---

## 2. Engineering Solution: Dual-Tier Architecture

To achieve absolute systemic harmony, a two-tier remediation strategy was engineered:

```
┌────────────────────────────────────────────────────────┐
│               LAYER 1: Terminal SSOT                    │
│    ~/.config/ghostty/config.ghostty & themes/x.ghosty  │
│    ~/.config/kitty/kitty.conf & themes/x.conf          │
│    (Defines single source of truth for base canvas)    │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│            LAYER 2: Semantic TUI Binding                │
│    ~/x/dev/own/atuin (crates/atuin interactive.rs)      │
│    (Explicit binding of Ratatui blocks to tokens)      │
│    • Borders & Dividers  ──► Meaning::Muted (#4A5168)  │
│    • Preview Content    ──► Syntax* & Meaning::Base    │
│    • Input Prompt       ──► Meaning::Title / Base      │
└────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Details

### 3.1 Terminal SSOT Calibration
- **Ghostty (`~/.config/ghostty/config.ghostty`):**
  Removed all redundant color keys (`background`, `foreground`, `split-divider-color`, `selection-*`, `cursor-*`). All color channels now strictly inherit from `themes/x.ghosty` (`foreground = #bcc4dd`).
- **Kitty (`~/.config/kitty/kitty.conf` & `themes/x.conf`):**
  Relocated standalone color keys (`cursor_trail_color`, `visual_bell_color`) into `themes/x.conf`, eliminating configuration fragmentation.

### 3.2 Atuin Interactive Engine (`interactive.rs`)

#### 3.2.1 Block Borders & Title Dividers
All structural blocks (`results_list`, `input`, `preview`, `info`) are explicitly styled with `Meaning::Muted`:
```rust
let border_style = Style::from_crossterm(theme.as_style(Meaning::Muted));
let title_line = Line::from(Span::styled(
    format!("{:─>width$}", "", width = style.inner_width.saturating_sub(2)),
    border_style,
));
```

#### 3.2.2 Multi-Line Preview Syntax Highlighting
`build_preview` now incorporates syntax classification (`syntax::classify`), generating styled `Line` spans for shell tokens, and falls back to `Meaning::Base`:
```rust
fn highlight_line(
    line: String,
    shell: Option<&str>,
    theme: &Theme,
    syntax_highlight: bool,
) -> Line<'static> {
    if !syntax_highlight {
        return Line::from(Span::styled(
            line,
            Style::from_crossterm(theme.as_style(Meaning::Base)),
        ));
    }
    let syntax = syntax::classify(&line, shell);
    let mut spans = Vec::new();
    let bytes = line.as_bytes();
    let mut i = 0;
    while i < bytes.len() {
        let meaning = syntax.get(i).copied().unwrap_or(Meaning::Base);
        let start = i;
        while i < bytes.len() && syntax.get(i).copied().unwrap_or(Meaning::Base) == meaning {
            i += 1;
        }
        spans.push(Span::styled(
            line[start..i].to_string(),
            Style::from_crossterm(theme.as_style(meaning)),
        ));
    }
    if spans.is_empty() {
        Line::from(Span::styled(line, Style::from_crossterm(theme.as_style(Meaning::Base))))
    } else {
        Line::from(spans)
    }
}
```

#### 3.2.3 Input Box Styling
The interactive query prompt renders mode headers with `Meaning::Title` and the active buffer with `Meaning::Base`.

---

## 4. Semantic Token Matrix

| UI Component | Semantic Meaning | `x.toml` Hex | Purpose |
|:---|:---|:---|:---|
| Container Borders | `Meaning::Muted` | `#4A5168` | Low-luminance structural framing (recedes into void) |
| Horizontal Dividers (`─`) | `Meaning::Muted` | `#4A5168` | Section separation without glare |
| Preview Text (Base) | `Meaning::Base` | `#7B8496` | Calm, readable body text |
| Preview Commands | `Meaning::SyntaxCommand` | `#00FF00` | Executable verbs |
| Preview Flags | `Meaning::SyntaxFlag` | `#646B84` | Dimmed option parameters |
| Preview Strings | `Meaning::SyntaxString` | `#00FF00` | Quoted string literals |
| Preview Variables | `Meaning::SyntaxVariable`| `#A855F7` | Environment variables |
| Preview Operators | `Meaning::SyntaxOperator`| `#F40091` | Pipes (`|`), redirections (`>`), logic (`&&`) |
| Input Prompt Tag | `Meaning::Title` | `#97A0BC` | Mode indicator (`[ SRCH: FULLTXT ]`) |
| Input Query Text | `Meaning::Base` | `#7B8496` | Active typing buffer |

---

## 5. Deployment & Verification

1. **Compilation:** Built via `cargo build --release -p atuin`.
2. **Quality Gate:** Passed all 607 unit and integration tests (`cargo test -p atuin --bin atuin`).
3. **Binary Link:** Installed to `~/.local/bin/atuin` (preceding `/opt/homebrew/bin` in system `$PATH`).
