# F-key Range Bindings for Indexed Actions

**Date:** 2026-07-01
**Status:** approved

## Motivation

Herder currently supports indexed keybindings using digit ranges (`1..9`) for actions like `switch_tab`, `switch_workspace`, and `focus_agent`. Users configure these as e.g. `switch_tab = "prefix+1..9"`, which generates `prefix+1` through `prefix+9`. Function keys (F1-F12) already parse as single-key bindings (e.g., `help = "f1"`), but there is no range syntax for F-keys to use them as indexed bindings.

Users want to bind F1-F12 to indexed actions so that pressing F1 switches to the first tab, F2 to the second, etc., optionally behind a prefix or combined with modifiers (ctrl, alt, shift).

## Design

Extend the existing range binding syntax (`1..9`) to support F-key ranges (`f1..f12`). The change is entirely within `src/config/keybinds.rs`. No changes to dispatch logic (`src/app/input/navigate.rs`), wire protocol, input parsing, or terminal encoding are needed — those layers already handle `KeyCode::F(n)`.

### Config syntax

```toml
[keys]
# Direct: F1 = tab 1, F2 = tab 2, …, F12 = tab 12
switch_tab = "f1..f12"

# Behind prefix: prefix+F1 = tab 1, etc.
switch_tab = "prefix+f1..f12"

# With modifiers: alt+F1 = tab 1, ctrl+F1 = workspace 1
switch_tab = "alt+f1..f12"
switch_workspace = "ctrl+f1..f12"
focus_agent = "alt+f1..f12"

# Mixed: digits cover 1-9, F-keys cover 10-12
switch_tab = ["prefix+1..9", "prefix+f10..f12"]
```

The single-key form `f1` already works for non-indexed actions via `parse_key_combo`. This change only adds the **range** form `f1..f12`.

### Index mapping

F-key number maps directly to zero-based index: F1 → index 0, F2 → index 1, …, F12 → index 11.

## Implementation

All changes in `src/config/keybinds.rs`. Four touchpoints:

### 1. `parse_range_modifiers` — add F-key range variant

Keep the existing function for `1..9`. Add a new `parse_fkey_range()` function that recognizes `f{start}..f{end}` tokens alongside modifier tokens:

```rust
fn parse_fkey_range(s: &str) -> Option<(KeyModifiers, u8, u8)> {
    // Split on '+', find token matching fN..fM, return (mods, start_n, end_n).
    // Same structure as parse_range_modifiers but matches f-start..f-end instead of 1..9.
}
```

### 2. `parse_binding_string` — expand F-key range

Add a branch before the existing `parse_range_modifiers` call. When an F-key range is detected, expand `F(start)..F(end)` with the same modifier/prefix combinators:

```rust
if let Some((range_modifiers, f_start, f_end)) = parse_fkey_range(body) {
    let bindings = (f_start..=f_end)
        .map(|n| {
            let combo = (KeyCode::F(n), range_modifiers);
            // ... same prefix/direct label logic as digit range
        })
        .collect();
    return Some(ParsedBinding::Range(bindings));
}
```

### 3. `push_indexed_binding` — relax validation gate

Currently rejects anything that isn't `KeyCode::Char('1'..='9')`. Extend to accept `KeyCode::F(1..=12)`:

```rust
// Before:
if !matches!(binding.trigger.combo().0, KeyCode::Char('1'..='9')) { /* diag */ }

// After:
let valid = matches!(
    binding.trigger.combo().0,
    KeyCode::Char('1'..='9') | KeyCode::F(1..=12)
);
```

Update the diagnostic string from `"indexed keybinding must use 1..9"` to mention F1-F12 as well.

### 4. `IndexedKeybind::matched_index` — extract index from F-keys

Currently only handles `KeyCode::Char('1'..='9')`. Add `KeyCode::F(1..=12)`:

```rust
pub fn matched_index(&self, key: TerminalKey) -> Option<usize> {
    if !terminal_key_matches_combo(key, self.trigger.combo()) {
        return None;
    }
    match key.code {
        KeyCode::Char(c @ '1'..='9') => Some((c as usize) - ('1' as usize)),
        KeyCode::F(n @ 1..=12) => Some((n as usize) - 1),
        _ => None,
    }
}
```

### Not changing

- `append_legacy_indexed_bindings` — the legacy `[keys.indexed]` configuration is not extended to F-keys. Users of the legacy format should migrate to the `switch_tab`/`switch_workspace`/`focus_agent` format.
- Dispatch logic (`navigate.rs`) — `indexed_navigation_action()` is index-agnostic; it just calls `matched_index()`.
- Terminal/input parsing — `KeyCode::F(n)` already flows through from escape sequences to `TerminalKey`.
- Wire protocol — `KeyCode::F(n)` already serializes.

## Testing

Unit tests in `src/config/keybinds.rs`:

1. `parse_fkey_range("f1..f12")` → `Some((empty_mods, 1, 12))`
2. `parse_fkey_range("alt+f1..f12")` → `Some((ALT, 1, 12))`
3. `parse_binding_string("f1..f12")` → `ParsedBinding::Range` of 12 `KeyCode::F(1..12)` `Direct` bindings
4. `parse_binding_string("prefix+f1..f12")` → same with `Prefix` trigger
5. `parse_binding_string("ctrl+f1..f12")` → `KeyCode::F(1..12)` with `CONTROL`
6. `IndexedKeybind::matched_index` with F3 returns `Some(2)`
7. `IndexedKeybind::matched_index` with F1 returns `Some(0)`
8. `IndexedKeybind::matched_index` with F13 returns `None`
9. `push_indexed_binding` accepts `KeyCode::F(1)`
10. Range validation: non-indexed action with `f1..f12` range produces diagnostic (existing behavior, unchanged)
11. `parse_modifier_combo` does NOT match `f` or `fkey` as a modifier (regression guard)

Integration test: load a config with `switch_tab = "f1..f12"` via `Config::validated_keybinds()` and verify 12 `IndexedKeybind` entries with correct triggers.

## Backward compatibility

No breaking changes. Existing `1..9` range bindings continue to work identically. The single-key `f1` binding (for non-indexed actions) is unchanged. The diagnostic string changes are cosmetic.
