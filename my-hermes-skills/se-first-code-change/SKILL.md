---
name: se-first-code-change
description: SE self-review pass before any multi-file code change plan.
version: 1
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [code-craft, refactor, planning, software-engineering, review]
    related_skills: [simplify-code, requesting-code-review, plan]
---

# SE-First Code Change Planning

Before presenting a multi-file code-change plan to the user, run a self-review pass on the plan itself. Catch the craft problems *before* writing the code, not after a commit breaks review.

**Core principle:** The cheapest place to fix a bad design is in the plan you haven't executed yet. Five minutes of SE self-review saves a revert cycle.

## When to Use

Trigger this skill when you are about to propose ANY of:

- A code change touching **2+ files**
- A refactor (extracting helpers, splitting god classes, renaming across modules)
- A feature that needs new code in an existing class
- Adding "two near-identical things" (e.g., 3 nearly-identical drawables, 3 nearly-identical configs)
- Designing a new public API (function signature, class surface, module boundary)

**Skip for:** single-file edits, docs/comments only, typo fixes, or pure mechanical formatting.

## The Three-Question Self-Review

Before you describe the plan to the user, answer these three questions in your head (or on paper for big changes):

### Q1 — Single source of truth?

For each *new* file, constant, type, or block of code you're adding, ask:
- Could this be **one** definition parameterized by something, instead of N near-copies?
- Is there already a thing in the codebase (or the platform/stdlib) that does this?

**Red flag:** "I'll add file A with X, file B with Y, file C with Z" where X/Y/Z differ only by a label, color, or string. → Merge to one parameterized thing.

**Concrete check:** Search the codebase for the proposed new helper's name (and obvious synonyms) before committing to add it. If a similar thing already exists, route to it instead of creating a parallel one.

### Q2 — Named units vs scattered if-blocks?

For each new behavior, ask: does it deserve a **named home**?

- New branchy logic that fans across multiple call sites → extract to a strategy/policy/object.
- New persistent state with read/write APIs → extract to a tiny data-access class.
- New cross-cutting check (host/permission/feature-flag) that one site reads → put it in one place, not duplicated.

**Red flag:** "I'll add an `if (mode == X)` check here, another `if (mode == Y)` check there, plus a third one over there." → Put the mode dispatch in one named function/class.

**The user's bar is "would this pass code review?"** Not "does it work?" Both matter, but review is what catches the duplication/structure problems.

### Q3 — Surface area proportional to value?

For each proposed change, ask:
- Can I delete anything? (Not refactor — actually remove.)
- Am I introducing abstractions the user didn't ask for? (YAGNI)
- Am I exposing internal state that callers don't need?

**Red flag:** "I'll also create an interface for this, and a factory, and..." → Stop. The user asked for a feature, not a framework.

## Java/Android-Specific Pitfalls

When the change is Java/Android code, these are the most-common pre-commit mistakes — verify before claiming "done":

### Import every type you declare

A new `Drawable d = ...`, `Intent intent = ...`, `Bundle bundle = ...` etc. requires an `import` for that type. The compiler catches this but you should grep the file's import list before commit:

```bash
# Find every simple-type name on lines you added, then verify each is imported
git diff path/to/File.java | grep -oE '\b(Drawable|Intent|Bundle|Uri|Context)\b' | sort -u
```

Or just visually scan your new lines — every bare type name not in `java.lang` needs an import.

### Don't mutate shared drawables

For Android drawables loaded via `AppCompatResources.getDrawable(...)`, call `.mutate()` before `setTint` or `setColorFilter`, otherwise the tint state leaks across every consumer of that resource.

```java
// Good: per-instance tint state
Drawable d = AppCompatResources.getDrawable(ctx, R.drawable.icon);
d = DrawableCompat.wrap(d).mutate();   // ← .mutate() is mandatory
DrawableCompat.setTint(d, color);
item.setIcon(d);
```

### SharedPreferences Set<String> is read-only-by-API-contract

`getStringSet` returns a Set you must NOT mutate in place. Read it, copy into a `new HashSet<>(...)`, modify, then `putStringSet`. Android explicitly documents this.

### Lint runs `assembleDebug`, not just `compileDebugJavaWithJavac`

A common false-confidence trap: code "compiles" but lint catches unused resources, hardcoded strings, or accessibility issues only at the `:app:lint` task. Run `./gradlew assembleDebug` (not just `:app:compileDebugJavaWithJavac`) for a real signal.

## What to Tell the User

After running the self-review, present the plan **and** tell the user the self-review happened. Use a brief one-liner like:

> "Before showing the plan, I ran an SE self-review and revised: dropped 3 duplicate drawables in favor of runtime tint, extracted the blocklist reads into a utility class."

If the self-review produced no changes, you don't need to announce it. The point is to make revisions visible when they happen.

## Common Anti-Patterns (and Their Fixes)

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Copy-paste-with-variation | 3 near-identical files differing only in a label/color | Parameterize: 1 file + runtime variation |
| Scattered if-trees | `if (mode == X)` checks in 4 different methods | Extract: 1 strategy/dispatch class with named branches |
| Helper sprawl on a God class | Activity has 15 private utility methods | Extract cohesive helpers into a sibling class, keep the Activity for coordination |
| Speculative abstractions | Interface + factory + impl for one caller | Inline until you have ≥2 callers or an explicit contract change |
| Premature optimization | "I'll cache this in case it's slow" | Profile first. If not measured, skip. |
| "Just add one more param" | Function signature now has 7 args | Group into a small struct/record. |
| Persistent-state invention | "I'll save this to SharedPreferences so it sticks" | Default: don't persist. Add persistence only when user explicitly asks for cross-session/cross-launch behavior |
| Direction-broadening | "切换全局" → I added host-level cross-launch blocklist | "全局" inside a single user action means the same instance/action, not persistence + dimension expansion |

## Minimum-Change Discipline (Listen to the User's Scope)

When the user gives a narrow scope ("就加一个方法就行了" / "别动其他文件" / "不要复杂化" / "能维持现状就别动"), treat that as a HARD constraint, not a preference. The cost of over-implementing is high (user trust erosion + revert cycle + commits to undo), and the cost of under-implementing is low (one follow-up commit).

**Hard rules when user says "just X" / "就 X":**

1. **Do ONLY X.** Do not "while I'm here" refactor adjacent code. Do not "tidy up" the function X lives in. Do not extract helpers "for cleanliness" if X can be inline.
2. **Do not propose "improvements" the user didn't ask for.** If the user said "加个按钮", they did not say "and also restructure the dialog flow". Don't propose the second part even in passing.
3. **If a tempting "while-I'm-here" change appears, STOP and ask.** One extra confirm turn beats one extra revert commit.
4. **Reuse > Reimplement.** When X naturally calls an existing method, call it. Do not write a "cleaner" parallel version. (Reverse failure mode I see often: copying-and-modifying instead of extracting a parameter.)

**Concrete example from a real failure (2026-10-01):**

> User asked: "弹窗加一个「拒绝并禁止弹窗」按钮,等效于顶部 toolbar 切换按钮。"
>
> ❌ What I did: Extracted `setExternalLinkMode(int)` + rewrote `toggleExternalBlockMode` to call it + added a wrapper `blockAllExternalLinks` + changed Toast text parameters.
>
> ✅ What the user wanted: One new method, nothing else touched. `toggleExternalBlockMode` stays exactly as it was. The new button calls `toggleExternalBlockMode()` directly OR a single new method — pick one, no parallel implementations.

The corrective checklist before commit:

- [ ] Did the user say "just X" or "就 X 就行了" or similar? → Lock scope to X.
- [ ] Am I touching files/methods NOT in X? → Stop. Either justify explicitly or revert.
- [ ] Am I introducing a new method that wraps another new method? → Invert: collapse to one.
- [ ] Am I "improving" Toast text, parameter signatures, or layout while adding the feature? → Revert. Defer.

## Literal-Interpretation Override (Don't Enrich the Verb)

When the user uses a verb that *could* mean a richer thing, pick the LITERAL meaning unless they specify richer. Default expansion path: implement the literal; if they want richer they'll say so.

| User says | Literal interpretation | Common over-interpretation to AVOID |
|---|---|---|
| "禁止 X" / "拒绝 X" | One-shot action: cancel this attempt | Persistent blocklist that survives app restart |
| "切换全局" / "全局生效" | Same instance, all callers see it | Persist to disk for next launch |
| "拦截所有" | All of THIS scope (this WebView, this session) | All apps on device, system-wide |
| "以后都 X" | Future calls in this context | Forever, even after uninstall/reinstall |
| "加个按钮" | Add one button, that's it | Add button + dialog + settings entry + onboarding hint |

**Rule of thumb:** If you find yourself reaching for `SharedPreferences` to implement a single UI action, ask "did the user ask for this to survive an app restart?" If no, do NOT persist.

## YAGNI Enforcement (when the user has already said "就 X")

User signs that they've already rejected scope creep:

- "你就...就行了" / "别的不用管"
- "不要复杂" / "简单粗暴"
- "我不需要...只需要..."
- "不要...也不要..."

When you see these phrasings, **trust them**. Do not:
- Add a "while-I'm-here" improvement
- Build a "for the future" abstraction
- Add a "to be safe" persistence layer
- Add a "for testing" entry point
- Add a "to undo" clearing UI for something you invented

If you do any of the above, you are guaranteed at least one revert commit later.

## 存量代码改动 = 默认禁止,先报告 (Default-No-Edit Rule)

This is the user's load-bearing rule, distinct from "minimum change" above. Minimum-change says "if you must touch adjacent code, do it small". This rule says **don't touch adjacent code at all without explicit permission**.

User phrasings that trigger this rule (any one):
- "过去的能不动就不要动了,保持稳定,尽量少改"
- "能不动就不动" / "能维持现状就别动"
- "如果非要迫不得已修改过去的,咱再打个报告"
- "如果原来的破烂不堪,我建议你向我说明一下原因,我们考虑重构"

**The asymmetry:**
- ✅ **New code you write** → must follow SE 三问 (single source / named units / surface area minimal). Author as if it shipped tomorrow.
- ⚠️ **Modifying existing stable code** → STOP. Produce a 4-column "存量改动报告" and ask the user to sign off before you write a single character.

**4-column report table format (use this exact shape):**

| # | 文件 | 改了什么 | 为什么必须改 (avoidable if …) | 你点头? |

Example:
```
1. LinkDbHelper.java  | 升 DATABASE_VERSION 16→17 + 加 3 个索引(idx_links_title / _timestamp / _tags_name) | 不加索引,3000+ 行搜索/分页会全表扫 → 卡顿。可避:用 in-memory filter,但 3000 条每次按键都重扫,慢。
2. dialog_select_link.xml | 整体重写(原来是空 RecyclerView 占位) | 原文件只能放 RecyclerView,加 EditText 必须扩字段。可避:新建 dialog_select_link_v2.xml 保留原文件。
3. AddSubjectItemDialog.showSelectLinkDialog() | 改为 SelectLinkDialog.show(...) | 丑弹窗入口就在这里,绕不开。可避:无,这是丑弹窗的唯一入口。
```

If the user says yes → do the change. If they say "新建 v2 保留原文件" → do that instead. If they say no → drop the change entirely; if the feature cannot land without that change, **say so and ask before extending scope**.

**Diagnostic — did I already edit before reporting?** If you catch yourself mid-flow realizing "I already changed LinkDbHelper before the user agreed" → that is exactly the failure pattern this rule prevents. Patch the report, list what was already done, and ask for retroactive approval. Do NOT pretend it didn't happen.

**Concrete failure pattern (2026-10-01, reading-share):**

User asked for an ugly-dialog fix in SubjectDetailActivity. I started editing `LinkDbHelper.java` (升 DB version + 加索引) before producing the report. The user later said: "过去的能不动就不要动了… 如果非要迫不得已修改过去的,咱再打个报告". I had to surface the already-applied change in a retroactive report.

**Rule: 在动任何已有 .java / .kt / 资源文件之前,如果该文件不在本次 feature 的直接调用链上(新增代码的依赖 / 编排对象不算),先打报告。** Specifically:
- DAO methods you ADD → 不算改存量,只要新增方法不修改旧方法体。✅
- 改 `LinkDbHelper` 的 `onCreate` / `onUpgrade` → 算改存量,即使新版是"加索引"。⚠️ 要报告。
- 改 dialog XML / layout 文件 → 算改存量,即使原文只有空占位。⚠️ 要报告。
- 改 Activity 里某个现有方法体(哪怕只换一行调用) → 算改存量。⚠️ 要报告。

**与"minimum-change"的区别:**
- minimum-change (上一节) = 你已经决定要改存量时,改得尽量小
- default-no-edit (本节) = 在你决定要改存量之前,**先问**;大部分情况下答案是"新建文件,不动旧的"

## Reference: Literal-Verb Lookup

When the user phrases a request with a "could-mean-richer" verb, consult `literal-interpretation-verbs.md` (同级文件) before defaulting to the richer interpretation. The reference contains:

- A verb → literal-meaning table (加按钮, 拒绝, 切换全局, 拦截所有, 以后都 X, 简单粗暴, 不要复杂, 维持现状 …)
- Two concrete failure-pattern walkthroughs from this user's sessions (Android toolbar mode-switch + minimum-change refactor)
- A pre-commit checklist specifically for "just X" requests

## User Language Preference (Java not Kotlin)

For this user specifically: **write Java, not Kotlin**. Even when Kotlin would be terser. Signals:
- User has never used Kotlin
- User said "我感觉还是使用Java语言我熟悉一些"
- They want code they can read and debug without learning a new language

Rule: when starting an Android project from scratch for this user, default to Java 8 source compatibility. Do not switch to Kotlin even if it would be one-line shorter. The bar is "user can read and debug it", not "first-line elegance".

## Modular-First Design (Reject the God Class)

When designing a new app / feature from scratch with multiple subsystems, **do not write the god class first and split later**. The order matters:

1. **Sketch the module boundary first** (one screen, ≤10 lines per file's responsibility)
2. **Sketch the dependency graph** — must be a DAG, never a cycle
3. **Then write leaf modules** (zero in-project dependencies)
4. **Then write the orchestrator** that composes leaves

If the user stops you mid-write with "等一下,先停一下,让我把要求说清楚" or similar → you have ALREADY written a god class. Stop, list the modules, ask for sign-off on the boundary before continuing.

**Concrete failure pattern (2026-10-01, Read Aloud project):** I wrote a 326-line `ReadOutTts.java` that did 7 things (WS frame encoding + text segmentation + AudioTrack mgmt + WebSocket listener + state machine + error propagation + config constants). User stopped me, said "高内聚低耦合,便于测试,职责单一". I had to throw it away and redo from scratch.

**Self-check before starting any 3+ module Android app:**

| Module | Single job | Deps (in-project only) |
|---|---|---|
| config/ | constants only | (none) |
| text/ | pure string ops | config only |
| audio/ | byte → speaker | config only |
| transport/ | wire bytes/strings | config only (or stdlib) |
| protocol/ | encode/decode messages | config only |
| orchestrator/ | compose leaves | everything above |
| ui/ | state → controls | orchestrator's interface only |
| MainActivity/ | view binding + lifecycle | ui interface only |

If you find yourself writing a method in the orchestrator that handles "what happens when text frame arrives" AND "how to play PCM bytes" — STOP. The audio sink has its own file.

## Adapter / Orchestrator / ValueObject 三件套 (Android/Java 落地范式)

This is the Java/Android-specific instantiation of the modular rule above. Whenever you build a new "picker / list-with-side-effects" UI (选链接、选文件、选主题、选标签……), split into exactly three new files plus zero-or-one existing orchestrator wiring:

| File | Role | Allowed deps | Forbidden |
|---|---|---|---|
| `util/<Scope>Spec.java` | Immutable value object. Owns prefix parsing, validation, defaults. | java stdlib only | DAO, Adapter, Dialog, any UI type |
| `ui/<area>/<Scope>Adapter.java` | Pure RecyclerView.Adapter. `setItems(List<T>)`, `appendItems(...)`, `setOn<X>ClickListener(...)`. Holds NO DAO, NO business rules. | model classes (e.g. `LinkItem`), the spec | DAO, SubjectDao, any I/O, ExecutorService |
| `ui/<area>/<Scope>Dialog.java` | DialogFragment orchestrator. Owns Adapter + DAO/SubjectDao + lifecycle. IO via `Executors.newSingleThreadExecutor()`. UI thread via `requireActivity().runOnUiThread`. | everything | — (this is the only file that holds DAO + UI together) |
| existing Activity / DialogFragment | Wires the new Dialog via callback listener | new Dialog's listener interface | — |

**Spec-before-Dialog rule:** Define `<Scope>Spec` and `<Scope>Scope` enum FIRST (in util/), then write the Adapter, then write the Dialog. Reason: the DAO method signature is `searchLinks(Spec)` not `searchLinks(String, int, boolean)` — the spec is the contract.

**Single-builder rule for DAO SQL:** When a DAO method has multiple "scopes" / "modes" / "filters", DO NOT write `switch (scope) { case A: ... case B: ... case C: ... }` inline with N copies of `cursor = database.rawQuery(...)`. Split into:
- `private String buildSearchQuery(Spec spec)` — one switch, one SQL string per case
- `private List<String> buildSearchArgs(Spec spec)` — one switch, one args list per case

This lets you ADD a new scope by editing two short methods, not hunting through 100 lines of cursor boilerplate.

**Concrete failure pattern (avoid):** Writing a `DialogFragment` that BOTH (a) renders the picker and (b) inserts into a different table via `SubjectDao`. The picker is supposed to be inert — pick, return. The orchestrator (the existing Activity) is supposed to decide what to do with the picked value. If you wrote `insertSubjectItemForSelection(long linkId)` inside the Dialog, you have a picker doing persistence. **Self-check before commit:** does any method on the Dialog end with `insertXxx(...)` or `deleteXxx(...)`? If yes, move it out.

**Worked example (2026-10-01, reading-share select link):**
- `LinkSearchSpec` (util/) — parses `title:foo` / `tag:bar` / `url:baz`
- `LinkSearchScope` (util/) — enum
- `SelectLinkAdapter` (ui/subject/) — renders `LinkItem` + applies disabled-alpha
- `SelectLinkDialog` (ui/subject/) — owns `LinkDao` + `SubjectDao`, listens to Adapter clicks, decides "未收录 → 回调" vs "已收录 → 取消收录"
- `AddSubjectItemDialog` (existing, minimal change) — wires `SelectLinkDialog.show(...)` + `setOnLinkSelectedListener(...)` callback
- `LinkDao.searchLinks(LinkSearchSpec)` + `buildSearchQuery(spec)` + `buildSearchArgs(spec)` — single builder, no inline duplication

## 项目级规则入 AGENTS.md (Project-Local Rules)

When the user says "把这个规则写进 AGENTS.md" / "存到项目的 AGENTS.md" / "把这个原则沉淀到项目里" — they are explicitly asking for **project-scoped, version-controlled rules** that travel with the repo, not session-only memory.

**Workflow:**
1. Read existing `AGENTS.md` first — don't duplicate or contradict.
2. Append new sections under their own headings (numbered, so ordering is stable: 第四条 / 第五条 / 第六条 …).
3. Encode each rule in imperative declarative form ("改 .java 后必须 import 扫描"), not narrative.
4. Reference the failure pattern once inline ("…教训:本会话 …") so future readers see WHY the rule exists.
5. Commit it in the same commit as the code that triggered the rule (if any). Single commit = single rule change.

**Do NOT:**
- Replace user's existing AGENTS.md wholesale — append.
- Write rules without a "why" — every rule should reference at least one past failure or explicit user statement.
- Mix project rules with session memory — project rules go in AGENTS.md, session rules go in ~/.hermes/memory.

**Signals that the user wants project-local rules:**
- "把这个写进 AGENTS.md" / "写到 AGENTS.md 里"
- "以后维护的时候方便看到"
- "沉淀到项目里"
- "作为通用原则写下来"

**Signals that the user wants session memory (not AGENTS.md):**
- "记住" / "记一下" / "下次别这样"
- "你的记忆里应该有这个"

## Session Start: Confirm What They Actually Mean (especially on input typos)

Real users mistype. A single-character typo in a feature request ("独享" instead of "选链接丑", "增加" instead of "新增") can send the entire subsequent investigation down the wrong file tree.

**Rule: when a request contains a typo / unusual phrasing / dialect word, do NOT hard-interpret.** Instead:

1. State your best guess in 1-2 lines
2. Ask: "对吗?"
3. Proceed only on confirmation

The cost of one round-trip question is ~10 seconds. The cost of reading 5 wrong files, building the wrong abstraction, and redoing it is ~30 minutes. The user explicitly corrected this pattern: "reading share 不是一个项目吗?不知道吗?" — they're OK with quick clarification, they're NOT OK with confident garbage.

**Symptom checklist before guessing:**

- The request word doesn't appear in any file path / class name / method name in the project
- It's a noun/verb combination that doesn't fit the project's domain
- It sounds like a homophone or near-homophone of something more common (独/选, 增/新)

**If 2+ symptoms hit, ask.** Don't ask "what do you mean by X" — ask with your guess attached: "你指的是不是 Y?我对吗?"

## Reading-Share-Specific DB Pain Point

When planning work that touches `LinkDao` / `LinkDbHelper` in `/root/projects/reading-share-android/`:

- The default DB has **3000+ link rows** in production
- `getAllLinks()` (`LinkDao.java:275`) loads everything into memory via single Cursor
- Any new code that wants to *list* links (select dialog, picker, search) MUST paginate or hit SQL LIKE
- Add `getLinksPage(offset, limit)` and `searchLinks(query, limit)` methods first if not present
- DB version already 16; bumping needs `onUpgrade` path with `CREATE INDEX IF NOT EXISTS`

Pattern is "the user has scale, don't pretend the DAO is small."

## "Stop Mid-Write and Re-Design" Anti-Pattern

If you've written 4+ files in a multi-file design and realize there's a structural problem (god class emerging, circular dep risk, callback chain won't connect), DO NOT patch incrementally. Stop, list the issue, propose a redesign, get sign-off, then re-implement.

Cheaper to throw away 4 files and redo than to keep patching a broken foundation. The user told me directly: "你使用什么语言？要用Java语言吧" mid-stream — that's the kind of correction you should have caught during design, not after writing.

## Why This Skill Exists

The "do it then review" loop has high cost: a 5-minute plan rewrite becomes a 30-minute revert-and-redo. The user (this user especially — they say "不要一直堆砌代码" and "为了保持代码的可维护性") values structural correctness as much as functional correctness. This skill encodes that as a pre-write gate, not a post-write fix.

## Related

- `simplify-code` — runs the same principles POST-write on recent changes (4 parallel reviewers)
- `requesting-code-review` — pre-commit verification: tests, lint, security scan
- `plan` — produces the markdown plan file; this skill refines the plan's craft before it lands

This skill is the pre-write gate. The other three are post-write or post-commit.
