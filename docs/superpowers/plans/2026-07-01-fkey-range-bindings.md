# F-key Range Bindings for Indexed Actions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the digit range binding syntax (`1..9`) to also support F-key ranges (`f1..f12`) for indexed actions (switch_tab, switch_workspace, focus_agent).

**Architecture:** Four changes in `src/config/keybinds.rs`: a new `parse_fkey_range` parser, a branch in `parse_binding_string` to expand F-key ranges, relaxed validation in `push_indexed_binding`, and an F-key arm in `IndexedKeybind::matched_index`. No changes to dispatch, wire protocol, or terminal input — those already handle `KeyCode::F(n)`.

**Tech Stack:** Rust, crossterm `KeyCode`, existing `Keybinds`/`IndexedKeybind` infrastructure

---

### File Structure

- Modify: `src/config/keybinds.rs` — all implementation and test changes

---

### Task 1: Add `parse_fkey_range_token` and `parse_fkey_range` helper functions

**Files:**
- Modify: `src/config/keybinds.rs`

- [ ] **Step 1: Add `parse_fkey_range_token` helper after `parse_range_modifiers`**

Place after `parse_range_modifiers` (currently at line 1146) and before `parse_modifier_combo` (line 1148).

```rust
fn parse_fkey_range_token(s: &str) -> Option<(u8, u8)> {
    let (start_str, end_str) = s.split_once("..")?;
    if !start_str.starts_with('f') || !end_str.starts_with('f') {
        return None;
    }
    let start: u8 = start_str[1..].parse().ok()?;
    let end: u8 = end_str[1..].parse().ok()?;
    if start < 1 || end > 12 || start > end {
        return None;
    }
    Some((start, end))
}
```

- [ ] **Step 2: Add `parse_fkey_range` after `parse_fkey_range_token`**

```rust
fn parse_fkey_range(s: &str) -> Option<(KeyModifiers, u8, u8)> {
    let mut modifiers = KeyModifiers::empty();
    let mut range: Option<(u8, u8)> = None;
    for part in s.split('+') {
        let trimmed = part.trim();
        if let Some((start, end)) = parse_fkey_range_token(trimmed) {
            if range.is_some() {
                return None;
            }
            range = Some((start, end));
        } else {
            modifiers |= parse_modifier_token(trimmed)?;
        }
    }
    range.map(|(start, end)| (modifiers, start, end))
}
```

- [ ] **Step 3: Run tests to verify no existing tests break**

```bash
cargo test -p herdr -- config::keybinds 2>&1 | tail -5
```
Expected: all existing tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/config/keybinds.rs
git commit -m "feat: add parse_fkey_range helpers for F-key range bindings"
```

---

### Task 2: Update `parse_binding_string` to expand F-key ranges

**Files:**
- Modify: `src/config/keybinds.rs`

- [ ] **Step 1: Add F-key range branch in `parse_binding_string` before the existing digit range call**

In `parse_binding_string` (line 1013), after extracting `(trigger_prefix, body)` and before the `if let Some(range_modifiers) = parse_range_modifiers(body)` line (currently line 1021), insert:

```rust
    if let Some((range_modifiers, f_start, f_end)) = parse_fkey_range(body) {
        let bindings = (f_start..=f_end)
            .map(|n| {
                let combo = (KeyCode::F(n), range_modifiers);
                let key_label = format_key_combo(combo);
                ResolvedBinding {
                    trigger: if trigger_prefix {
                        BindingTrigger::Prefix(combo)
                    } else {
                        BindingTrigger::Direct(combo)
                    },
                    label: if trigger_prefix {
                        format!("prefix+{key_label}")
                    } else {
                        key_label
                    },
                }
            })
            .collect();
        return Some(ParsedBinding::Range(bindings));
    }
```

- [ ] **Step 2: Run tests to verify no existing tests break**

```bash
cargo test -p herdr -- config::keybinds 2>&1 | tail -5
```
Expected: all existing tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/config/keybinds.rs
git commit -m "feat: expand f-key ranges in parse_binding_string"
```

---

### Task 3: Update `push_indexed_binding` validation and diagnostic

**Files:**
- Modify: `src/config/keybinds.rs`

- [ ] **Step 1: Relax the validation gate in `push_indexed_binding`**

In `push_indexed_binding` (line 860), change the gate at line 868:

Replace:
```rust
    if !matches!(binding.trigger.combo().0, KeyCode::Char('1'..='9')) {
        let diag = format!(
            "indexed keybinding must use 1..9: {field} = {:?}; disabling binding",
            binding.label
        );
```

With:
```rust
    if !matches!(
        binding.trigger.combo().0,
        KeyCode::Char('1'..='9') | KeyCode::F(1..=12)
    ) {
        let diag = format!(
            "indexed keybinding must use 1..9 or f1..f12: {field} = {:?}; disabling binding",
            binding.label
        );
```

- [ ] **Step 2: Update `invalid_indexed_binding_does_not_displace_default_binding` test assertion**

The test at line 1963 checks for the old diagnostic string. Change the assertion in that test:

Replace:
```rust
        assert!(diagnostics.iter().any(|diag| {
            diag.contains("indexed keybinding must use 1..9") && diag.contains("keys.switch_tab")
        }));
```

With:
```rust
        assert!(diagnostics.iter().any(|diag| {
            diag.contains("indexed keybinding must use 1..9 or f1..f12")
                && diag.contains("keys.switch_tab")
        }));
```

- [ ] **Step 3: Run tests to verify**

```bash
cargo test -p herdr -- config::keybinds 2>&1 | tail -5
```
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/config/keybinds.rs
git commit -m "feat: relax push_indexed_binding validation to accept F-keys"
```

---

### Task 4: Update `IndexedKeybind::matched_index` to extract index from F-keys

**Files:**
- Modify: `src/config/keybinds.rs`

- [ ] **Step 1: Add `KeyCode::F` arm to `matched_index`**

In `IndexedKeybind::matched_index` (line 257), replace the method body:

Replace:
```rust
    pub fn matched_index(&self, key: TerminalKey) -> Option<usize> {
        let KeyCode::Char(c @ '1'..='9') = key.code else {
            return None;
        };
        if terminal_key_matches_combo(key, self.trigger.combo()) {
            Some((c as usize) - ('1' as usize))
        } else {
            None
        }
    }
```

With:
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

- [ ] **Step 2: Run existing tests**

```bash
cargo test -p herdr -- config::keybinds 2>&1 | tail -5
```
Expected: all existing tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/config/keybinds.rs
git commit -m "feat: extract index from F-keys in IndexedKeybind::matched_index"
```

---

### Task 5: Write tests for F-key range bindings

**Files:**
- Modify: `src/config/keybinds.rs` (test module)

- [ ] **Step 1: Add unit tests for `parse_fkey_range`**

Add near the existing `parse_range_modifiers` tests or after `parse_shift_tab_as_backtab` (line 1458):

```rust
    #[test]
    fn parse_fkey_range_basic() {
        assert_eq!(parse_fkey_range("f1..f12"), Some((KeyModifiers::empty(), 1, 12)));
    }

    #[test]
    fn parse_fkey_range_with_modifier() {
        assert_eq!(
            parse_fkey_range("alt+f1..f12"),
            Some((KeyModifiers::ALT, 1, 12))
        );
        assert_eq!(
            parse_fkey_range("ctrl+f1..f8"),
            Some((KeyModifiers::CONTROL, 1, 8))
        );
    }

    #[test]
    fn parse_fkey_range_partial() {
        assert_eq!(
            parse_fkey_range("f10..f12"),
            Some((KeyModifiers::empty(), 10, 12))
        );
    }

    #[test]
    fn parse_fkey_range_rejects_invalid() {
        assert_eq!(parse_fkey_range("f13..f20"), None);
        assert_eq!(parse_fkey_range("f0..f5"), None);
        assert_eq!(parse_fkey_range("f5..f3"), None);
        assert_eq!(parse_fkey_range("f1..f13"), None);
        assert_eq!(parse_fkey_range("not-a-range"), None);
    }

    #[test]
    fn parse_fkey_range_token_basic() {
        assert_eq!(parse_fkey_range_token("f1..f12"), Some((1, 12)));
        assert_eq!(parse_fkey_range_token("f3..f7"), Some((3, 7)));
    }

    #[test]
    fn parse_fkey_range_token_rejects_non_fkey() {
        assert_eq!(parse_fkey_range_token("1..9"), None);
        assert_eq!(parse_fkey_range_token("a..z"), None);
    }
```

- [ ] **Step 2: Run the new tests to verify they pass**

```bash
cargo test -p herdr -- config::keybinds::tests::parse_fkey_range 2>&1 | tail -10
```
Expected: all `parse_fkey_range*` tests pass.

- [ ] **Step 3: Add integration test for F-key range binding in config**

Add after the `prefixed_indexed_bindings_support_modifiers` test (line 1919):

```rust
    #[test]
    fn fkey_range_switch_tab_generates_12_direct_bindings() {
        let config: Config = toml::from_str(
            r#"
[keys]
switch_tab = "f1..f12"
"#,
        )
        .unwrap();
        let kb = config.keybinds();
        assert_eq!(kb.switch_tab.len(), 12);
        assert_eq!(
            kb.switch_tab[0].trigger,
            BindingTrigger::Direct((KeyCode::F(1), KeyModifiers::empty()))
        );
        assert_eq!(kb.switch_tab[0].label, "f1");
        assert_eq!(
            kb.switch_tab[11].trigger,
            BindingTrigger::Direct((KeyCode::F(12), KeyModifiers::empty()))
        );
        assert_eq!(kb.switch_tab[11].label, "f12");
    }

    #[test]
    fn fkey_range_switch_tab_with_prefix() {
        let config: Config = toml::from_str(
            r#"
[keys]
switch_tab = "prefix+f1..f12"
"#,
        )
        .unwrap();
        let kb = config.keybinds();
        assert_eq!(kb.switch_tab.len(), 12);
        assert_eq!(
            kb.switch_tab[0].trigger,
            BindingTrigger::Prefix((KeyCode::F(1), KeyModifiers::empty()))
        );
        assert_eq!(kb.switch_tab[0].label, "prefix+f1");
        assert!(kb.switch_tab.iter().all(|b| b.trigger.is_prefix()));
    }

    #[test]
    fn fkey_range_with_modifier() {
        let config: Config = toml::from_str(
            r#"
[keys]
switch_workspace = "alt+f1..f12"
"#,
        )
        .unwrap();
        let kb = config.keybinds();
        assert_eq!(kb.switch_workspace.len(), 12);
        assert_eq!(
            kb.switch_workspace[0].trigger,
            BindingTrigger::Direct((KeyCode::F(1), KeyModifiers::ALT))
        );
        assert_eq!(kb.switch_workspace[0].label, "alt+f1");
    }

    #[test]
    fn fkey_range_partial_supports_subset() {
        let config: Config = toml::from_str(
            r#"
[keys]
switch_tab = "prefix+f10..f12"
"#,
        )
        .unwrap();
        let kb = config.keybinds();
        assert_eq!(kb.switch_tab.len(), 3);
        assert_eq!(
            kb.switch_tab[0].trigger,
            BindingTrigger::Prefix((KeyCode::F(10), KeyModifiers::empty()))
        );
        assert_eq!(kb.switch_tab[0].label, "prefix+f10");
    }

    #[test]
    fn fkey_range_mixed_with_digit_range() {
        let config: Config = toml::from_str(
            r#"
[keys]
switch_tab = ["prefix+1..9", "prefix+f10..f12"]
"#,
        )
        .unwrap();
        let kb = config.keybinds();
        assert_eq!(kb.switch_tab.len(), 12);
        assert_eq!(
            kb.switch_tab[0].trigger,
            BindingTrigger::Prefix((KeyCode::Char('1'), KeyModifiers::empty()))
        );
        assert_eq!(
            kb.switch_tab[9].trigger,
            BindingTrigger::Prefix((KeyCode::F(10), KeyModifiers::empty()))
        );
    }
```

- [ ] **Step 4: Run the integration tests**

```bash
cargo test -p herdr -- config::keybinds::tests::fkey_range 2>&1 | tail -10
```
Expected: all `fkey_range*` tests pass.

- [ ] **Step 5: Add `matched_index` tests for F-keys**

Add after the new integration tests:

```rust
    #[test]
    fn matched_index_fkey_returns_correct_index() {
        let binding = IndexedKeybind {
            trigger: BindingTrigger::Direct((KeyCode::F(3), KeyModifiers::empty())),
            label: "f3".into(),
        };
        assert_eq!(binding.matched_index(TerminalKey::from(KeyEvent::new(KeyCode::F(3), KeyModifiers::empty()))), Some(2));
    }

    #[test]
    fn matched_index_fkey_rejects_wrong_modifier() {
        let binding = IndexedKeybind {
            trigger: BindingTrigger::Direct((KeyCode::F(3), KeyModifiers::ALT)),
            label: "alt+f3".into(),
        };
        assert_eq!(binding.matched_index(TerminalKey::from(KeyEvent::new(KeyCode::F(3), KeyModifiers::empty()))), None);
    }

    #[test]
    fn matched_index_fkey_rejects_out_of_range() {
        let binding = IndexedKeybind {
            trigger: BindingTrigger::Direct((KeyCode::F(1), KeyModifiers::empty())),
            label: "f1".into(),
        };
        assert_eq!(binding.matched_index(TerminalKey::from(KeyEvent::new(KeyCode::F(13), KeyModifiers::empty()))), None);
    }

    #[test]
    fn matched_index_fkey_rejects_non_fkey() {
        let binding = IndexedKeybind {
            trigger: BindingTrigger::Direct((KeyCode::F(1), KeyModifiers::empty())),
            label: "f1".into(),
        };
        assert_eq!(binding.matched_index(TerminalKey::from(KeyEvent::new(KeyCode::Char('a'), KeyModifiers::empty()))), None);
    }
```

- [ ] **Step 6: Run the `matched_index` tests**

```bash
cargo test -p herdr -- config::keybinds::tests::matched_index 2>&1 | tail -10
```
Expected: all `matched_index*` tests pass.

- [ ] **Step 7: Add diagnostic test for F-key range on non-indexed action**

Add after the `invalid_indexed_binding_does_not_displace_default_binding` test (line 1989):

```rust
    #[test]
    fn fkey_range_rejected_on_non_indexed_action() {
        let config: Config = toml::from_str(
            r#"
[keys]
help = "f1..f12"
"#,
        )
        .unwrap();

        let diagnostics = config.collect_diagnostics();
        let kb = config.keybinds();

        assert!(kb.help.bindings.is_empty());
        assert!(diagnostics.iter().any(|diag| {
            diag.contains("range keybinding is only valid for indexed actions")
                && diag.contains("keys.help")
        }));
    }
```

- [ ] **Step 8: Run the diagnostic test**

```bash
cargo test -p herdr -- config::keybinds::tests::fkey_range_rejected 2>&1 | tail -5
```
Expected: test passes.

- [ ] **Step 9: Run all keybinds tests to confirm**

```bash
cargo test -p herdr -- config::keybinds 2>&1 | tail -5
```
Expected: all tests pass.

- [ ] **Step 10: Commit**

```bash
git add src/config/keybinds.rs
git commit -m "test: add tests for F-key range bindings and matched_index"
```

---

### Task 6: Full validation

**Files:**
- None (read-only)

- [ ] **Step 1: Run `just check`**

```bash
just check
```
Expected: all formatting, clippy, and test checks pass.

- [ ] **Step 2: Commit any final fixups if needed**

```bash
git add -A && git commit -m "chore: fix clippy/format issues from just check"
```
Skip this step if `just check` passes cleanly.
