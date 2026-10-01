# Literal-Interpretation Lookup for Verb-Heavy User Requests

When the user uses a verb that *could* mean something richer, default to the LITERAL meaning. This file is the concrete lookup table; the SKILL.md has the principle.

## Why this exists

Every few sessions, the user asks for "just X" and I add Y, Z, and a config layer because the verb *could* mean them. The user reverts. Time and trust burned. This file is the lookup to consult before every "while I'm here" impulse.

## The lookup

| User says this | Implement THIS | Do NOT also add |
|---|---|---|
| 加个按钮 | One button, wired up. That's it. | Dialog logic, settings entry, analytics, persistent preference for that button's visibility |
| 加个方法 | One new method, possibly one call site change | Rewriting the method it delegates to, extracting a wrapper around it |
| 拒绝 / 禁止 | One-shot reject on the current action | Persistent blocklist, host/domain dimension, cross-session memory |
| 切换全局 | Same instance, all callers see it | SharedPreferences persistence for next launch |
| 拦截所有 | This WebView / this session — all of it | System-wide intent filter, accessibility service, device policy |
| 以后都 X | Future calls in THIS context (this session, this page) | Forever, even after app reinstall / data clear |
| 全局生效 | Same instance + this app's other instances | Cross-app, cross-device |
| 简单粗暴 | Minimum code, no abstractions | Even "small" helpers, even "for readability" extractions |
| 不要复杂 | YAGNI enforced at the verb level | ANY state, persistence, dimension, abstraction not strictly required |
| 维持现状 | Nothing changes — confirm before touching | Adjacent "improvements" |
| 跳转 / 打开 / 关闭 | The literal action once | Retry logic, fallback chain, error handling for rare cases |

## Real-session failure pattern (2026-10-01, Android toolbar mode-switch)

User asked: 「拒绝并禁止弹窗」按钮 (reject and stop this kind of dialog).

What I built:
- ✗ `ExternalLinkBlocklist` class (full SharedPreferences Set<String> wrapper)
- ✗ `keyOf()` host extraction with scheme-fallback
- ✗ Blocklist check in `handleUrlOverride`
- ✗ Neutral button on the AlertDialog
- ✗ Toast text with formatted blocked-key
- ✗ No clearing UI (created a one-way door)

What the user wanted:
- ✓ ONE neutral button that calls the existing toolbar toggle, OR a small new method that sets mode=2 directly.

Cost: 2 extra commits to undo (revert host blocklist + delete the wrapper class).

## Real-session failure pattern (2026-10-01, "setExternalLinkMode" minimum-change)

User asked: 「多加一个方法就行了」.

What I built:
- ✗ New `setExternalLinkMode(int)` — OK
- ✗ Rewrote `toggleExternalBlockMode` to call `setExternalLinkMode(newMode)` — VIOLATION
- ✗ Refactored message text to a switch — VIOLATION
- ✗ Then added `blockAllExternalLinks` wrapper on top — DOUBLE VIOLATION

What the user wanted:
- ✓ Add one method.
- ✓ Don't touch `toggleExternalBlockMode`. It works.
- ✓ The new method's call site (the neutral button) calls it directly. No wrapper.

After the user re-prompted "就多加一个方法就行了", I:
- ✓ Restored `toggleExternalBlockMode` byte-for-byte
- ✓ Removed the `blockAllExternalLinks` wrapper
- ✓ Net change: +1 method, -0 in other code.

## Pre-commit checklist for "just X" requests

- [ ] Did the user use a "just X" / "就 X" / "不要" signal? If yes, lock scope to X literally.
- [ ] Am I changing any file/method/line NOT directly required by X? Revert.
- [ ] Did I introduce a wrapper/extra layer not strictly required by X? Revert.
- [ ] Did I "improve" surrounding code (renames, refactors, message texts)? Revert.
- [ ] Did I add persistence for a single-session action? Revert unless explicitly asked.
- [ ] Am I adding any kind of "to undo" / "to clear" UI for something the user didn't ask for? Revert.

If any checkbox is checked, do the revert before commit, not after.
