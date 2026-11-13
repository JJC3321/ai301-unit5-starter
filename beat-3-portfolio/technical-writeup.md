# Fixing Keyboard Step Scaling and False Dirty-State Signals in Avogadro 2 — and Rebuilding the Issue-Scout Config That Found It

## Overview

Avogadro 2 (`OpenChemistry/avogadrolibs`) is an open-source C++17 and Qt 6 molecular editor used by chemists and researchers, and `issue-scout` is a GitHub issue-hunting skill whose six-check `rubric.md`, real-world fit profile in `scope.md`, and `gh search issues` queries and ranking filters in `search-strategy.md` I authored myself. In Avogadro's Auto Optimize tool (`AutoOpt::keyPressEvent`), keyboard navigation had two accessibility and state-hygiene bugs ([#3188](https://github.com/OpenChemistry/avogadrolibs/issues/3188)): holding `Alt` with a movement key ignored the `5×` larger translation step supported by the Manipulate tool, and pressing any unhandled key still emitted a molecule-modified signal that dirtied saved files. I used `issue-scout` to surface and grade `#3188`, rewrote `scope.md` and `search-strategy.md` after my first classroom-era configuration failed to target C++ without manual steering, stood up Avogadro's Windows CMake superbuild to reproduce and fix both bugs by hand, and submitted [PR #3235](https://github.com/OpenChemistry/avogadrolibs/pull/3235), which maintainer `@ghutchis` approved and merged into `master` on October 26, 2026.

## What happened

### 1. Screening #3188 with `issue-scout` — and fixing the tool after Version 1 fell short

Earlier in the term (`2026-09-27`), I wrote `issue-scout`'s core six-check grading rubric (`rubric.md`) and validated it on the 20-bundle Unit 1 evaluation harness at `19/20` agreement (`sha256:ceb3f8f300269c9d` in `beat-1-sandbox/unit-1/eval-run.txt`). When I moved from our classroom sandbox repo to public GitHub on October 22, 2026, I wanted my first wild contribution to be in C++. Pointing the search at C++ `good first issue` candidates surfaced `OpenChemistry/avogadrolibs#3188`, and my rubric accepted it across every check:

| Rubric check | Grade | Evidence gathered on `OpenChemistry/avogadrolibs#3188` |
|---|---|---|
| `repo-alive` (required) | `pass` | Not archived; active default-branch merges within days of the scan |
| `issue-open` (required) | `pass` | `state: open` (opened `2026-10-17`) |
| `unclaimed` (required) | `pass` | `assignees: none`, no open linked PR, zero prior claim comments |
| `newcomer-scope` (required) | `pass` | Bounded to `AutoOpt::keyPressEvent` (`autoopt.cpp:818-853`), cites `manipulator.cpp:160-170`, includes 4-step test plan |
| `ai-policy-ok` (required) | `pass` | `CONTRIBUTING.md` allows Level 0/1 AI (learning, planning, debugging help) when contributed code is human-written |
| `has-good-first-label` (preferred) | `pass` | Labeled `good first issue`, `help wanted`, and `accessibility` |

I posted a claim comment on `#3188` (`2026-10-22T01:56:41Z`), and maintainer `@ghutchis` confirmed it the same afternoon:

> `@JJC3321 - sure - no one else has asked.`

While the rubric worked well at filtering out bad candidates, that first wild run exposed a miss in my own tool configuration: my carryover fit profile still described our classroom Python/TypeScript web app, and my starter repo's `skill/scope.md` and `skill/search-strategy.md` were still unpopulated templates, which meant I had steered `gh search issues` toward C++ by hand instead of making the scan repeatable. Rather than keep routing around stale config, I rewrote `skill/scope.md` and authored explicit C++ queries and deterministic ranking filters in `skill/search-strategy.md`:

```diff
--- a/skill/scope.md (Version 1: classroom Path Review fit profile)
+++ b/skill/scope.md (Version 2: real-world C++-first fit profile)
-I am comfortable in Python and TypeScript/React ... so I prefer issues in
-`api/`, `ingestion/`, `rag/`, `agent/`, or `safety/` over pure CSS or copy polish.
+For my real-world contributions, I prioritize C++ desktop applications, libraries,
+and systems/tooling projects (such as Qt/C++ GUI plugins, scientific tools, and
+data-structure-heavy libraries)...

--- a/skill/search-strategy.md (Version 1: 0 queries, 0 ranking filters)
+++ b/skill/search-strategy.md (Version 2: repeatable C++ queries + 8-step ranking)
+gh search issues --label "good first issue" --language "c++" --state open --limit 100 \
+  --json title,repository,labels,assignees,commentsCount,updatedAt,url
+gh search issues --label "good first issue" --label "accessibility" --language "c++" \
+  --state open --limit 50 --json title,repository,labels,assignees,commentsCount,updatedAt,url
+1. Drop assigned issues: len(assignees) > 0
+2. Drop stale issues: updatedAt older than 60 days
+3. Drop crowded threads: commentsCount > 8
+4. Drop docs/typo-only labels; rank C++ ahead of Python fallback, then label count,
+   then commentsCount ascending, tie-breaking on updatedAt descending.
```

### 2. Reconfiguring a failing Windows superbuild and reproducing the bug

On October 24, 2026, I forked `OpenChemistry/avogadrolibs` at pre-fix commit `14b4a6f4` and attempted to compile Avogadro 2 on Windows 11 through the `OpenChemistry/openchemistry` CMake superbuild. My first build attempts failed before producing an executable: CMake picked a mismatched compiler toolchain against my Qt 6.10.2 `mingw_64` installation, and optional third-party subprojects (`HDF5`, `MoleQueue`, and Python bindings) failed mid-build. Following Avogadro's Level 0/1 AI policy, I used Claude only to interpret CMake/MinGW errors, then reconfigured the superbuild to pin MinGW-w64 `g++.exe` 13.1.0, Ninja 1.12.1, and `Qt6_DIR="C:/Qt/6.10.2/mingw_64/lib/cmake/Qt6"` while disabling `-DBUILD_MOLEQUEUE=OFF -DUSE_HDF5=OFF -DUSE_PYTHON=OFF`.

Once `avogadro2.exe` built and launched, I ran the maintainer's four-step manual test plan on `14b4a6f4` and confirmed both bugs in the GUI and in `avogadro/qtplugins/autoopt/autoopt.cpp`: every directional branch passed a literal `0.1` Å to `translate(...)` without checking `e->modifiers()`, and `m_molecule->emitChanged(Molecule::Atoms | Molecule::Modified)` sat unconditionally *after* the `switch`, firing even after `default: e->ignore()`.

### 3. Shipping the C++/Qt fix and landing PR #3235

Still on October 24, I created branch `fix/auto-optimize` and wrote the fix by hand in `AutoOpt::keyPressEvent`, mirroring `Manipulator::keyPressEvent` (`manipulator.cpp:160-170`) while keeping the optional tooltip out of scope per my `plan.md`. Commit `43adcacb` (signed off under the Developer Certificate of Origin) touched one file (`+18 / -10`):

```diff
--- a/avogadro/qtplugins/autoopt/autoopt.cpp
+++ b/avogadro/qtplugins/autoopt/autoopt.cpp
@@ -820,35 +820,43 @@ QUndoCommand* AutoOpt::keyPressEvent(QKeyEvent* e)
   if (m_molecule == nullptr)
     return nullptr;
 
+  // Translation step in Angstroms
+  double transStep = 0.1;
+  if (e->modifiers() & Qt::AltModifier)
+    transStep = 0.5;
+
+  bool moved = true;
   switch (e->key()) {
     case Qt::Key_Left:
     case Qt::Key_H:
     case Qt::Key_A:
-      translate(Vector3(-0.1, 0.0, 0.0));
-      e->accept();
+      translate(Vector3(-transStep, 0.0, 0.0));
       break;
...
     default:
-      e->ignore();
+      moved = false;
+  }
+
+  if (moved) {
+    e->accept();
+    m_molecule->emitChanged(Molecule::Atoms | Molecule::Modified);
+  } else {
+    e->ignore();
   }
-  m_molecule->emitChanged(Molecule::Atoms | Molecule::Modified);
   return nullptr;
 }
```

Rebuilding `build/avogadrolibs` and re-running the four-step test plan in `avogadro2.exe` confirmed both behaviors changed as intended:

| Manual check in `avogadro2.exe` | Before (`14b4a6f4`) | After (`43adcacb`) |
|---|---|---|
| Select atom in Auto Optimize, press `→` | Moves `+0.1 Å` on X | Moves `+0.1 Å` on X |
| Press `Alt+→` (and `Alt` + `↑`/`←`/`↓`) | Moves `0.1 Å` (`Alt` ignored) | Moves `0.5 Å` (`5×` step) |
| Save clean molecule (`Ctrl+S`), press unused key `Q` | Window title gains modified marker (`*`) | Title stays clean; `e->ignore()` called with no `emitChanged` |

After running my Unit 7 ship-gate check against `plan.md` to confirm zero scope drift and clean `clang-format` output, I opened [PR #3235](https://github.com/OpenChemistry/avogadrolibs/pull/3235) on October 25, 2026. GitHub Actions passed `clang-format-check.yml` and built test artifacts across all six target platforms (`Linux x86_64`, `Linux ARM64`, `macOS arm64`, `Windows x64`, `Windows ARM`, and `Flatpak`). On October 26, 2026 (`15:35:16Z`), `@ghutchis` approved the PR:

> `LGTM, thanks`

He merged PR #3235 into `master` (`81c108bb`) two hours later, closing `#3188`. Having the Windows superbuild and C++ `issue-scout` workflow in place also paid off immediately afterward, letting me land a second Avogadro accessibility PR ([PR #3338](https://github.com/OpenChemistry/avogadrolibs/pull/3338), merged November 12, 2026).

## Limitations and what's next

First, PR #3235 adds no automated C++ unit test and leaves the optional `Alt` tooltip mention (`manipulator.cpp:61`) unimplemented. Because `tests/qtplugins/` has no existing `AutoOpt` test target and `keyPressEvent` requires an active `QtGui::Molecule` state, I relied on interactive verification in `avogadro2.exe` and six-platform CI builds. Next, I plan to add a headless GoogleTest regression test that constructs a `QtGui::Molecule`, attaches a `QSignalSpy` to `Molecule::changed`, sends synthetic `QKeyEvent`s (`Qt::Key_Right`, `Qt::Key_Right | Qt::AltModifier`, and `Qt::Key_Q`), and asserts `0.1 Å`, `0.5 Å`, and zero emitted signals on ignored keys, paired with a small follow-up PR adding the `Alt` hint to Auto Optimize's tooltip.

Second, my submitted PR #3235 description summarized local testing in a single prose sentence and omitted an explicit AI-use note because `avogadrolibs`'s `.github/PULL_REQUEST_TEMPLATE` contains only the DCO text, while `issue-scout`'s free ranking stage still cannot spot informal `"can I work on this?"` claims inside a low-comment thread (`commentsCount <= 8`) until the paid rubric stage fetches the issue. To close both gaps, I updated my pre-PR checklist to require concrete test output in every PR body (already applied on PR #3338) and plan to add a lightweight `gh api` comment-body pre-check in `issue-scout` before spending model tokens on top-five rubric grading.
