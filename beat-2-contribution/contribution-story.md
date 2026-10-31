# Contribution Story

- **Project:** [OpenChemistry/avogadrolibs](https://github.com/OpenChemistry/avogadrolibs), the core C++/Qt library behind Avogadro 2, an open-source molecular editor and visualizer
- **Issue:** https://github.com/OpenChemistry/avogadrolibs/issues/3188
- **Pull Request:** https://github.com/OpenChemistry/avogadrolibs/pull/3235

## Toolkit

### Unit 5: Used issue-scout to find a C++ first issue outside the classroom repo

I wanted my first real contribution to be in C++, so I pointed issue-scout's search at C++ issues instead of the Python and TypeScript work I usually do. It surfaced Avogadro's #3188, and graded against my rubric it passed every required check (see the table in Why I Chose This Issue). One shortcoming: my fit profile was written for the Path Review classroom repo and favors Python/TypeScript folders like `api/` and `rag/`, so it didn't describe what I was actually looking for this time. Because of that, I steered the search by language myself rather than relying on the profile's ranking.

### Unit 6: Did not run repro-check; reproduced the bugs by hand

I didn't run repro-check on this issue. My installed copy was still the classroom version, whose `scope.md` limits it to the Path Review repo and tells it to refuse issues from any other repository, so it couldn't grade an Avogadro issue. Instead I reproduced both bugs by hand, following the maintainer's four-step test plan in #3188. My claim comment was a short request, and the maintainer approved it the same day. The shortcoming is that I posted without the rubric checking my work. Next time I'd install the Unit 6 version, which is scoped to outside repos, before claiming.

### Unit 7: Used the ship-gate to check that my branch matched plan.md

I ran the ship-gate on my `fix/auto-optimize` branch to compare the changes I actually made against what I wrote in `plan.md`. It confirmed the branch did what the plan said and nothing more: the `transStep` Alt check and the `moved` flag around `emitChanged`, both in `AutoOpt::keyPressEvent`. The tooltip, which I had marked as out of scope, was untouched. What held me up most was compiling Avogadro: building it from source through the `openchemistry` superbuild kept failing, and I couldn't test my change until it ran. I used AI to help troubleshoot the build errors, which Avogadro's AI policy allows for debugging help. Once I had a working build I could actually run the app and confirm my changes worked, instead of relying on the code alone.

### Unit 8: Did not run the PR-monitor; the PR merged within a day

I didn't use the PR-monitor or playbook on PR #3235, and I don't have it installed. After opening the PR on Oct 25 I asked @ghutchis for a review myself ("can you review this"). He approved and merged it the next day, before there was any waiting state worth monitoring. So the monitor had nothing to read and no nudge to draft. The honest lesson is that I skipped the tool because the PR moved fast, not because I tested it. On a slower PR I'd install it first and let its threshold decide when a follow-up nudge is appropriate, instead of pinging the maintainer right away.

## Why I Chose This Issue

The issue was labeled `good first issue`, `help wanted`, `hacktoberfest`, and `accessibility`, and it is part of the project's accessibility roadmap (#3186). The maintainer had already written it out clearly: the exact function (`AutoOpt::keyPressEvent`, `autoopt.cpp:818-853`), a reference implementation in the Manipulate tool (`manipulator.cpp:160-170`), and a four-step manual test plan. The scope was small, "done" was well defined, and I could check my fix against existing code in the same repo.

It also fit my profile. I'm most comfortable in Python and TypeScript, but I write a fair amount of C++ for systems and data-structures work. My fit profile asks for "bounded OSS fixes with … clear acceptance criteria" and ranks "UI-only styling with no behavior change" lower. This was a real behavior fix in C++, in a scientific desktop app I had never worked in.

Graded against my rubric:

| Check | Result | Evidence |
|---|---|---|
| `repo-alive` | pass | Not archived; frequent default-branch merges |
| `issue-open` | pass | Opened Oct 17, 2026 |
| `unclaimed` | pass | No assignee, no linked PR, no claim comments before mine |
| `newcomer-scope` | pass | One function, exact line numbers, a reference implementation, and a test plan written by the maintainer |
| `ai-policy-ok` | pass (with conditions) | `CONTRIBUTING.md` allows AI only at "levels 0 and 1" (autocomplete, learning, planning); contributed code must be written by a human |
| `has-good-first-label` | pass | `good first issue` |

**Toolkit:** I used issue-scout to search for C++ issues, because I wanted to contribute in that language, and it found #3188. The rubric grading above is what convinced me it was safe to claim: it was unclaimed, small, labeled for newcomers, and in a repo whose AI policy I could follow. In hindsight, the scout was strongest at filtering out bad candidates (claimed, vague, or inactive issues). It was weaker at matching my goals, because my fit profile still described my Path Review preferences rather than "C++ in a real project."

## Reproduction Process

**Environment:** Windows, Avogadro 2 built from source through the `openchemistry` superbuild, from my fork `JJC3321/avogadrolibs` on `master` before the fix (parent commit `14b4a6f4`).

**Steps (from the issue's test plan):**

1. Open a molecule, switch to the Auto Optimize tool, and select an atom.
2. Press → and watch the atom move.
3. Press Alt+→.
4. Save the file so the window title has no "modified" marker, then press a letter Auto Optimize does not use (for example `Q`).

**Observed before the fix:**

- Steps 2 and 3 move the atom by the same 0.1 Å. Alt has no effect.
- Step 4: the window title gains the "modified" marker even though no atom moved.

**Confirmed in code:** before the fix, every case in the `switch` passed a hard-coded `0.1` to `translate(...)`, and `m_molecule->emitChanged(Molecule::Atoms | Molecule::Modified)` sat *after* the `switch`, so it ran for every key, including the ones that hit `default: e->ignore()`.

**Toolkit:** I didn't use repro-check here. My installed copy was scoped to the classroom repo and would have refused an Avogadro issue. I reproduced the bugs manually instead, using the maintainer's test plan, and confirmed the cause by reading `AutoOpt::keyPressEvent`.

## Understanding the Issue

1. **What is the problem?** In the Auto Optimize tool, Alt+arrow does not take a larger step, and any key press (even an unused one) marks the molecule as modified.
2. **What should happen instead?** Auto Optimize should match the Manipulate tool: 0.1 Å by default, a larger step with Alt, and no change signal when a key doesn't move anything.
3. **Who does it affect?** Users who move atoms with the keyboard instead of the mouse, which is why it sits under the accessibility roadmap. The false "modified" state also produces unneeded redraws and a save prompt when nothing changed.
4. **Where is it in the code?** `AutoOpt::keyPressEvent` in `avogadro/qtplugins/autoopt/autoopt.cpp`. The reference behavior is `Manipulator::keyPressEvent` in `avogadro/qtplugins/manipulator/manipulator.cpp`.

**Toolkit:** [Which tool helped you understand the issue (repro-check, reading the code with Claude under the project's AI policy, etc.), what it produced, and anything it got wrong.]

## Solution Approach

### Analysis

The cause is entirely in `AutoOpt::keyPressEvent`:

- The step size was a literal `0.1` in each of the four direction cases, and the function never read `e->modifiers()`, so Alt could not change anything.
- `emitChanged(...)` ran unconditionally after the `switch`. The `default` branch called `e->ignore()` but then fell through to the change signal anyway.

### Proposed Solution

- Compute `transStep` up front: 0.1 Å, or 0.5 Å when `Qt::AltModifier` is held, and use it in all four directions, matching the Manipulate tool.
- Track whether the key moved atoms with a `moved` flag. Call `e->accept()` and `emitChanged(...)` only when it did; otherwise call `e->ignore()`.

**What I won't change:** the key bindings themselves, the Manipulate tool, the optimizer, or the tooltip. The issue lists the tooltip as optional, and I left it for a separate PR to keep this one focused.

### Implementation Plan

1. Fork `OpenChemistry/avogadrolibs`, clone it, and build Avogadro through the `openchemistry` superbuild.
2. Reproduce both bugs on `master`.
3. Create a branch, `fix/auto-optimize`.
4. Add `transStep` with the Alt check and replace the four hard-coded `0.1` values.
5. Add the `moved` flag, move `e->accept()` and `emitChanged(...)` inside `if (moved)`, and call `e->ignore()` otherwise.
6. Rebuild, rerun the reproduction steps, and confirm `clang-format` leaves the file unchanged.
7. Commit with a DCO `Signed-off-by` line (the repo's PR template is the Developer Certificate of Origin) and open the PR.

**Toolkit:** [How you used plan-check on plan.md and plan-comment.md, its verdict, and what you revised. If you skipped it because the maintainer's issue already contained the plan, say so.]

## Implementation Notes

- **2026-10-17:** @ghutchis opens #3188 with labels and a test plan, linked to the accessibility roadmap #3186.
- **2026-10-22:** I comment to claim the issue; @ghutchis replies the same day, "sure - no one else has asked."
- **2026-10-24:** Fork cloned and Avogadro built through the `openchemistry` superbuild. Reproduced both bugs. Wrote the fix in a single commit, `43adcac` "Fix the Alt for larger steps and no change signal for ignored keys" (+18 / −10, one file), signed off with my full name.
- **2026-10-25:** Opened PR #3235 from `JJC3321:fix/auto-optimize` and asked @ghutchis for a review. CI built test artifacts for Linux x86_64 and ARM64, macOS arm64, Windows x64 and ARM, and Flatpak.
- **2026-10-26:** @ghutchis approved ("LGTM, thanks") and merged. #3188 closed as completed.

**Toolkit:** [Which tool, if any, helped during implementation, what it produced, and what you changed because of it.]

## Testing Strategy

**Manual testing (the issue's own test plan),** on my local build of the fix branch:

| Step | Before the fix | After the fix |
|---|---|---|
| → with an atom selected | moves 0.1 Å | moves 0.1 Å |
| Alt+→ | moves 0.1 Å (Alt ignored) | moves 0.5 Å |
| Unused letter after saving | title gains "modified" marker | title stays clean |

**Automated checks:** the project's CI built the PR on Linux (x86_64 and ARM64), macOS, Windows (x64 and ARM), and Flatpak, and published test builds for each. CI also runs `clang-format-check.yml` on pull requests.

**Why no unit test:** `tests/qtplugins/` has no existing AutoOpt tests, and `keyPressEvent` needs a live molecule, an active tool, and a GL widget. The issue asked for the manual test plan above, so I used that rather than build a new test harness for a two-line behavior change. A GoogleTest regression test that sends a synthetic `QKeyEvent` and asserts the molecule's change signal would be a good follow-up.

**Toolkit:** The hardest part of testing was getting the software to compile at all. My first attempts to build Avogadro through the `openchemistry` superbuild failed, and I used AI to help me read the build errors and work out what to fix. The code change itself I wrote by hand, as the project's AI policy requires. Once the build worked, I ran the app with my changes and went through the issue's test plan to confirm both fixes behaved as expected.

## Pull Request

- **PR Link:** https://github.com/OpenChemistry/avogadrolibs/pull/3235
- **Claim comment:** https://github.com/OpenChemistry/avogadrolibs/issues/3188#issuecomment-6330819774
- **Status:** Merged Oct 26, 2026, approved by @ghutchis

**PR Description (as submitted):**

> fix #3188
>
> **Summary**
> - Add a check to the `AutoOpt::keyPressEvent` to detect when an Alt and a movement key are pressed, and change the movement from 0.1 A to 0.5 A.
> - Add a check to see whether atoms have moved before calling `emitChanged` to prevent unused keys from triggering `emitChanged`.
>
> I compiled and built the software successfully on my local machine, and the application runs as expected with the changes I made.

**Evidence:**

- Diff: one file, `avogadro/qtplugins/autoopt/autoopt.cpp`, +18 / −10 ([Files changed](https://github.com/OpenChemistry/avogadrolibs/pull/3235/files)).
- CI: all build workflows produced artifacts for commit `43adcac` (Linux x86_64 106.8 MB, Linux ARM64 107.3 MB, macOS arm64 90.8 MB, Windows x64 114.5 MB, Windows ARM 103.1 MB, Flatpak 58.4 MB).
- Review: approved with "LGTM, thanks"; merged the same day with no changes requested.

**Looking back at the description:** it links the issue and summarizes both changes, but it doesn't include before/after output or say what AI helped with. Next time I'd add both. Avogadro's AI policy makes that disclosure especially worth including.

**Toolkit:** Before opening the PR, I ran the ship-gate on my branch against `plan.md` to confirm that what I changed was what I had planned to change. It found my one-file diff matched the plan's two changes, with no unplanned edits, so I opened PR #3235 with the scope I had promised on the issue. [Anything it flagged in pr_draft.md, or anything you'd change about it.]

## Learnings

- **Building a large C++/Qt project from source.** Avogadro depends on Qt, Eigen, and the OpenChemistry superbuild. Getting a working local build took far more effort than the code change itself. I struggled with compile errors and used AI to help me troubleshoot them, and I learned that a fix I can't run is a fix I can't verify.
- **Finding my way around an unfamiliar codebase.** I didn't try to understand all of Avogadro. The issue's file and line pointers, plus the existing Manipulate implementation, were enough. Copying a pattern that already exists in the repo is the fastest way to write code a maintainer will accept.
- **Qt's event model.** `accept()` vs `ignore()` decides whether a parent widget can still handle a key, and one misplaced `emitChanged` call has visible effects: a false "modified" state, an unneeded redraw, and a save prompt when nothing changed.
- **Reading the contribution rules first.** Avogadro's AI policy only allows AI for learning and planning, and its PR template is a DCO that requires a `Signed-off-by` line with my full name. Knowing both before writing code shaped how I worked.
- **Choosing well matters most.** What made this go smoothly was a bounded issue with a reference implementation and a test plan, in an active repo, with nobody else on it. For my next issue I want the same signals, plus one that needs an automated regression test. The optional Alt tooltip (`manipulator.cpp:61`) is a natural small follow-up.

## Resources

- [Issue #3188](https://github.com/OpenChemistry/avogadrolibs/issues/3188) and the [accessibility roadmap #3186](https://github.com/OpenChemistry/avogadrolibs/issues/3186)
- [avogadrolibs CONTRIBUTING.md](https://github.com/OpenChemistry/avogadrolibs/blob/master/CONTRIBUTING.md): build steps, coding conventions, and AI usage policy
- [Avogadro build instructions](https://two.avogadro.cc/develop/build.html): the `openchemistry` superbuild
- `avogadro/qtplugins/manipulator/manipulator.cpp`: the reference Alt-step implementation
- [Qt `QKeyEvent` documentation](https://doc.qt.io/qt-6/qkeyevent.html): `key()`, `modifiers()`, `accept()`, `ignore()`
- [Developer Certificate of Origin](https://developercertificate.org/): the repo's PR template
