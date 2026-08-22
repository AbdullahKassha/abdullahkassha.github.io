---
title: 'The Hidden Second Job of `.git` in Codex: Repository Metadata vs. Project Boundary'
description: 'Why an empty `.git` can be invalid to Git yet still change Codex project-root and AGENTS.md discovery.'
date: '2026-08-22'
draft: false
sticky: false
showHeroImage: false
tags:
  - Codex
  - Git
  - AGENTS.md
  - Windows
categories:
  - Technical Investigation
series: []
comments: false
sidebar:
  enable: true
  toc: true
  relatedPosts: true
---

An empty directory named `.git` produced an odd split-screen result in a local Codex
workspace:

```text
Git CLI:          not a Git repository
Codex behavior:   instruction discovery changed
```

How can a directory that Git rejects still matter to Codex?

The answer is that `.git` can play two conceptually different roles in this workflow. Git
uses it as repository metadata. Codex can use its presence as a filesystem marker that
defines a project root. Those roles usually point to the same directory, so the distinction
stays hidden. An empty `.git` exposes it.

This article develops that mental model from four evidence layers: current documentation,
current source code, a controlled boundary experiment, and issue reports. The model is a
synthesis, not an official Codex term.

> **Verification note (August 22, 2026):** The documentation, implementation files, and
> issue statuses linked below were re-checked before drafting. The controlled experiment
> itself was performed on August 20–21, 2026; its exact Codex desktop build was not captured.

## The mental model: one name, two questions

Git and Codex do not need the same answer from a path named `.git`.

<figure>
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 620" role="img" aria-labelledby="two-meanings-title two-meanings-desc" style="width: 100%; height: auto">
    <title id="two-meanings-title">Two meanings of .git</title>
    <desc id="two-meanings-desc">The name .git branches into Git repository metadata and a Codex project-root marker. The two roles normally align but are not semantically identical.</desc>
    <style>
      .gm-bg { fill: #f7f7f2; }
      .gm-card { fill: #fffefa; stroke: #cbd5cd; stroke-width: 2; }
      .gm-marker { fill: #dce9df; stroke: #6f927d; stroke-width: 2; }
      .gm-line { fill: none; stroke: #6f927d; stroke-width: 4; stroke-linecap: round; stroke-linejoin: round; }
      .gm-title { fill: #26332c; font: 700 32px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .gm-label { fill: #26332c; font: 700 27px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .gm-copy { fill: #526159; font: 22px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .gm-note { fill: #785f3d; font: 700 23px ui-monospace, SFMono-Regular, Consolas, monospace; }
    </style>
    <rect class="gm-bg" width="1200" height="620" rx="28" />
    <text class="gm-title" x="600" y="62" text-anchor="middle">One name, two questions</text>
    <rect class="gm-marker" x="498" y="94" width="204" height="82" rx="20" />
    <text class="gm-label" x="600" y="145" text-anchor="middle">.git</text>
    <path class="gm-line" d="M600 176 V218 M600 218 H300 V252 M600 218 H900 V252" />
    <rect class="gm-card" x="94" y="252" width="412" height="230" rx="22" />
    <text class="gm-label" x="300" y="310" text-anchor="middle">Git interpretation</text>
    <text class="gm-copy" x="300" y="362" text-anchor="middle">Repository metadata</text>
    <text class="gm-copy" x="300" y="405" text-anchor="middle">HEAD · objects · refs</text>
    <text class="gm-copy" x="300" y="448" text-anchor="middle">config · index · hooks</text>
    <rect class="gm-card" x="694" y="252" width="412" height="230" rx="22" />
    <text class="gm-label" x="900" y="310" text-anchor="middle">Codex interpretation</text>
    <text class="gm-copy" x="900" y="362" text-anchor="middle">Project-root marker</text>
    <text class="gm-copy" x="900" y="405" text-anchor="middle">Discovery boundary for</text>
    <text class="gm-copy" x="900" y="448" text-anchor="middle">project instructions</text>
    <rect x="292" y="520" width="616" height="66" rx="18" fill="#f4eadb" stroke="#cfb991" stroke-width="2" />
    <text class="gm-note" x="600" y="562" text-anchor="middle">Normally aligned ≠ semantically identical</text>
  </svg>
</figure>

Git asks whether the repository metadata is usable. The [`git init`
documentation](https://git-scm.com/docs/git-init) describes an initialized repository as a
`.git` directory with structures such as `objects` and `refs`, plus template files. A merely
empty directory with the same name does not supply that repository structure.

Codex project-root discovery asks a different question: has it reached a directory containing
a configured root marker? The [current advanced configuration
documentation](https://learn.chatgpt.com/docs/config-file/config-advanced) says that Codex
walks upward from the working directory and, by default, treats a directory containing `.git`
as the project root. The marker list is configurable through `project_root_markers`; `.git` is
important here because it is the default, not because it is the only possible marker.

That gives us the compact version of the model:

> **Git cares whether `.git` resolves to usable repository metadata. Codex root discovery can
> care that a configured `.git` marker exists.**

This is deliberately an explanatory simplification. It describes the default marker behavior
verified for this investigation, not every possible configuration or future release.

## Four terms that make the rest predictable

Before following the boundary, it helps to separate four terms:

- **Working directory:** the directory in which the Codex task starts.
- **Project root:** the filesystem boundary Codex selects for project-scoped discovery.
- **Root marker:** a name such as `.git` whose presence can identify that boundary.
- **`AGENTS.md`:** a file that supplies project guidance to Codex for the applicable part of a
  directory tree.

Here, “project root” means the filesystem discovery boundary. It does not mean an
application-level Project used to organize chats and folders in the Codex interface.

## How the project root bounds `AGENTS.md`

The current [`AGENTS.md`
documentation](https://learn.chatgpt.com/docs/agent-configuration/agents-md) describes a path
algorithm, not a global search:

1. Codex starts with the discovered project root.
2. It walks down the ancestry path toward the current working directory.
3. In each directory on that path, it selects at most one applicable instruction file.
4. It combines those files root-first, so nearer guidance appears later and can specialize or
   override broader guidance.
5. If no project root is found, it checks only the current directory.

Consider this tree:

```text
shop\
├── .git                  ← default project-root marker
├── AGENTS.md             ← project-wide guidance
│
├── frontend\
│   ├── AGENTS.md         ← frontend guidance
│   └── components\       ← working directory
│       └── Button.tsx
│
└── backend\
    └── AGENTS.md         ← sibling branch
```

For a task started in `frontend\components`, the relevant directory chain is:

```text
shop
↓
frontend
↓
components
```

`shop\AGENTS.md` and `shop\frontend\AGENTS.md` are on that path. The backend file is not.
Saying that “`AGENTS.md` inheritance follows ancestors, not arbitrary siblings” is a useful
deduction from the documented algorithm; it is not quoted product terminology.

The root matters because it defines where that ancestry begins. Move the root closer to the
working directory and guidance above it falls outside the discovery path.

## The edge case: `.git` exists, but there is no repository

Now place an empty `.git` at the working-directory level:

```text
my-app\
├── AGENTS.md
└── src\
    └── feature\          ← working directory
        └── .git\         ← empty
```

Git can reject `feature` as a repository because the expected metadata is absent. Under the
default Codex root-marker behavior, the same filesystem entry can identify `feature` as the
nearest project root. `my-app\AGENTS.md` then sits above the boundary.

This is not just an interpretation of the prose documentation. The Codex implementation
re-checked for this draft describes discovery as walking upward until a configured marker
entry is found, using `.git` as the default when no custom list is set. The corresponding
tests explicitly note that `.git` may be a file or a directory. See the pinned
[`agents_md.rs`](https://github.com/openai/codex/blob/bd19459358f534ed1cae464ec13d56600aeb45f2/codex-rs/core/src/agents_md.rs)
snapshot and the current
[`agents_md_tests.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/agents_md_tests.rs).

Issue
[`#25651`](https://github.com/openai/codex/issues/25651) provides an independent real-world
example. Its reporter found that an empty nested `.git` prevented repository-root guidance
from loading. A Codex maintainer explained that root discovery uses filesystem markers and
does not validate the entry as a functional Git repository. The issue is now closed as
“not planned,” treating that marker behavior as expected rather than as a loader defect.

There is one current implementation caveat worth keeping separate: project trust can also
gate whether project-scoped instructions load at all. That is an independent decision from
where the marker-based root is found, and it was not the variable changed in the experiment
below.

## A controlled project-boundary experiment

The strongest original evidence came from a four-phase experiment on Windows using
PowerShell and a fresh Codex thread for every phase. Fresh threads mattered because an
existing thread could retain prior instructions and blur the result.

The sanitized test tree was:

```text
workspace\
└── main-threads-lab\
    └── discovery-probe\
        ├── AGENTS.md
        └── child\          ← working directory
```

The parent `AGENTS.md` contained one exact probe rule:

```markdown
When the user sends exactly:

ROOT_MARKER_PROBE

respond exactly:

PARENT-AGENTS-SEEN
```

An exact `PARENT-AGENTS-SEEN` response therefore indicated that the parent instruction had
reached that fresh thread.

| Phase | Root-marker state | Fresh-thread result |
| --- | --- | --- |
| A | Empty higher marker existed above `discovery-probe`; no marker in `child` | `PARENT-AGENTS-SEEN` |
| B | An empty `child\.git` was added | `ROOT_MARKER_PROBE acknowledged.` |
| C | The empty `child\.git` was removed | `PARENT-AGENTS-SEEN` |
| D | The higher marker was removed too, leaving no relevant marker | `ROOT_MARKER_PROBE` |

The boundary changes can be read as a small state machine:

<figure>
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1400 650" role="img" aria-labelledby="boundary-title boundary-desc" style="width: 100%; height: auto">
    <title id="boundary-title">The empty .git boundary experiment</title>
    <desc id="boundary-desc">Four phases show a parent instruction being discovered, cut off by an empty child .git, restored when that marker is removed, and absent again when no root marker remains.</desc>
    <defs>
      <marker id="boundary-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M0 0 L10 5 L0 10 Z" fill="#6f927d" />
      </marker>
    </defs>
    <style>
      .be-bg { fill: #f7f7f2; }
      .be-panel { fill: #fffefa; stroke: #cbd5cd; stroke-width: 2; }
      .be-phase { fill: #dce9df; }
      .be-phase-text { fill: #26332c; font: 700 24px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .be-tree { fill: #435249; font: 20px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .be-good { fill: #3f7358; font: 700 19px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .be-cut { fill: #94633e; font: 700 19px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .be-arrow { fill: none; stroke: #6f927d; stroke-width: 3; marker-end: url(#boundary-arrow); }
      .be-action { fill: #526159; font: 16px ui-monospace, SFMono-Regular, Consolas, monospace; }
      .be-title { fill: #26332c; font: 700 30px ui-monospace, SFMono-Regular, Consolas, monospace; }
    </style>
    <rect class="be-bg" width="1400" height="650" rx="28" />
    <text class="be-title" x="700" y="54" text-anchor="middle">The project-boundary experiment</text>
    <g transform="translate(34 92)">
      <rect class="be-panel" width="300" height="490" rx="22" />
      <rect class="be-phase" width="300" height="66" rx="22" />
      <path class="be-phase" d="M0 44 H300 V66 H0 Z" />
      <text class="be-phase-text" x="150" y="42" text-anchor="middle">A · Higher root</text>
      <text class="be-tree" x="32" y="120">workspace/</text>
      <text class="be-tree" x="32" y="158">├── .git [empty]</text>
      <text class="be-tree" x="32" y="196">└── probe/</text>
      <text class="be-tree" x="32" y="234">    ├── AGENTS.md</text>
      <text class="be-tree" x="32" y="272">    └── child/</text>
      <rect x="24" y="362" width="252" height="86" rx="16" fill="#e7f0e9" />
      <text class="be-good" x="150" y="398" text-anchor="middle">Parent instruction</text>
      <text class="be-good" x="150" y="426" text-anchor="middle">seen</text>
    </g>
    <g transform="translate(378 92)">
      <rect class="be-panel" width="300" height="490" rx="22" />
      <rect class="be-phase" width="300" height="66" rx="22" />
      <path class="be-phase" d="M0 44 H300 V66 H0 Z" />
      <text class="be-phase-text" x="150" y="42" text-anchor="middle">B · Nearer root</text>
      <text class="be-tree" x="32" y="120">probe/</text>
      <text class="be-tree" x="32" y="158">├── AGENTS.md</text>
      <text class="be-tree" x="32" y="196">└── child/</text>
      <text class="be-tree" x="32" y="234">    └── .git [empty]</text>
      <text class="be-tree" x="32" y="286">boundary = child</text>
      <rect x="24" y="362" width="252" height="86" rx="16" fill="#f4eadb" />
      <text class="be-cut" x="150" y="398" text-anchor="middle">Parent instruction</text>
      <text class="be-cut" x="150" y="426" text-anchor="middle">cut off</text>
    </g>
    <g transform="translate(722 92)">
      <rect class="be-panel" width="300" height="490" rx="22" />
      <rect class="be-phase" width="300" height="66" rx="22" />
      <path class="be-phase" d="M0 44 H300 V66 H0 Z" />
      <text class="be-phase-text" x="150" y="42" text-anchor="middle">C · Remove child</text>
      <text class="be-tree" x="32" y="120">workspace/</text>
      <text class="be-tree" x="32" y="158">├── .git [empty]</text>
      <text class="be-tree" x="32" y="196">└── probe/</text>
      <text class="be-tree" x="32" y="234">    ├── AGENTS.md</text>
      <text class="be-tree" x="32" y="272">    └── child/</text>
      <rect x="24" y="362" width="252" height="86" rx="16" fill="#e7f0e9" />
      <text class="be-good" x="150" y="398" text-anchor="middle">Parent instruction</text>
      <text class="be-good" x="150" y="426" text-anchor="middle">returns</text>
    </g>
    <g transform="translate(1066 92)">
      <rect class="be-panel" width="300" height="490" rx="22" />
      <rect class="be-phase" width="300" height="66" rx="22" />
      <path class="be-phase" d="M0 44 H300 V66 H0 Z" />
      <text class="be-phase-text" x="150" y="42" text-anchor="middle">D · No root</text>
      <text class="be-tree" x="32" y="120">workspace/</text>
      <text class="be-tree" x="32" y="158">└── probe/</text>
      <text class="be-tree" x="32" y="196">    ├── AGENTS.md</text>
      <text class="be-tree" x="32" y="234">    └── child/</text>
      <text class="be-tree" x="32" y="286">scope = cwd only</text>
      <rect x="24" y="362" width="252" height="86" rx="16" fill="#f4eadb" />
      <text class="be-cut" x="150" y="398" text-anchor="middle">Current-directory</text>
      <text class="be-cut" x="150" y="426" text-anchor="middle">fallback</text>
    </g>
    <path class="be-arrow" d="M334 606 H378" />
    <text class="be-action" x="356" y="628" text-anchor="middle">add</text>
    <path class="be-arrow" d="M678 606 H722" />
    <text class="be-action" x="700" y="628" text-anchor="middle">remove</text>
    <path class="be-arrow" d="M1022 606 H1066" />
    <text class="be-action" x="1044" y="628" text-anchor="middle">remove higher</text>
  </svg>
</figure>

### Phase A: an ancestor root allowed parent discovery

At the start, an old empty `.git` still existed higher in the sanitized `workspace` path.
Although Git did not recognize it as a repository, it provided a root marker above
`discovery-probe\AGENTS.md`. The probe returned `PARENT-AGENTS-SEEN`.

This supports the conclusion that the parent file was on the root-to-working-directory path.
It does not turn the higher empty marker into valid Git metadata.

### Phase B: an empty nearer marker cut the parent off

An empty directory was created at `child\.git`, and a forced listing confirmed that it had no
contents. In a new thread, the same prompt returned `ROOT_MARKER_PROBE acknowledged.` rather
than the required probe response.

Only the nearer marker changed. The result supports the conclusion that the empty `.git`
changed the effective project-root and instruction-discovery boundary in the environment
tested. The parent `AGENTS.md` now sat above that boundary.

### Phase C: removing the marker restored the instruction

The empty `child\.git` was removed. A third fresh thread again returned
`PARENT-AGENTS-SEEN`. The A/B/C reversal is important: it makes random variation a less
plausible explanation than the controlled marker change.

### Phase D: no marker meant current-directory-only discovery

Finally, the higher empty marker was removed as well. With no relevant marker in the ancestry,
a fourth fresh thread returned the probe text itself rather than the parent-defined response.
That result is consistent with the documented fallback: when Codex cannot determine a project
root, it checks only the current directory.

Together, the four phases reveal three states:

```text
Higher root exists  → parent AGENTS.md can be on the discovery path
Closer root exists  → guidance above that boundary is cut off
No root is found    → discovery falls back to the working directory
```

### What the experiment does—and does not—establish

The experiment directly observed a boundary effect in one Codex environment on August 20–21,
2026. It did not reveal internal state directly; the conclusion combines the controlled A/B/C/D
changes with the documented discovery algorithm and implementation evidence.

It does not establish that every Codex version, custom marker configuration, trust state, or
future implementation behaves identically. The exact desktop build was not recorded, and the
model selection used in part of the investigation is not an application-version identifier.

## Why the distinction became practical on Windows

The investigation began with older Windows workspaces containing this pattern:

```text
workspace\
├── .git\       [empty]
├── .codex\     [empty]
├── .agents\    [empty]
└── project-files\
```

Some groups shared the same creation timestamp. In those directories, a direct audit produced
the compact contradiction:

```text
Own .git: True
Git root: (none)
```

Matching timestamps and repeated directory triplets strongly suggest a common automated
origin, but they do not prove which code path created each historical directory. The right
wording is “consistent with reported behavior,” not “proven to be created by one bug.”

The issue trail makes that connection plausible:

| Issue | Opened | Status checked August 22, 2026 | Relevance |
| --- | --- | --- | --- |
| [`#29492`](https://github.com/openai/codex/issues/29492) | June 22, 2026 | Open | Reports empty `.git`, `.codex`, and `.agents` paths appearing during Windows sandbox handling |
| [`#32872`](https://github.com/openai/codex/issues/32872) | July 13, 2026 | Open | Reproduces an empty `.git` in a non-Git Windows workspace and correlates it with sandbox ACL setup |
| [`#34799`](https://github.com/openai/codex/issues/34799) | July 22, 2026 | Open | Reports an empty `.git` created during an ordinary edit in a TFVC workspace |
| [`#25651`](https://github.com/openai/codex/issues/25651) | June 1, 2026 | Closed as not planned | Confirms the separate discovery consequence of a spurious empty marker |

There is also counter-evidence against declaring the old creation behavior universal. A clean
local project tested on August 20, 2026 created and read ordinary files without producing
`.git`, `.codex`, or `.agents`. Current Windows sandbox source, re-checked at the
[`win.rs`](https://github.com/openai/codex/blob/a26f219f6788c951dcb3bf435fab4c6d0f4d2f40/codex-rs/windows-sandbox-rs/src/bin/setup_main/win.rs)
snapshot, contains filtering intended to prevent missing legacy protected children from being
materialized as sentinel directories.

Those facts support a careful conclusion: the historical behavior did not reproduce in the
clean test, and current source contains preventative handling. They do not prove that every
related path is fixed in every distributed build—especially while the cited Windows reports
remain open.

The second job of `.git` explains why a stale artifact can still matter after its original
creation mechanism stops reproducing. Preventing a new marker from appearing does not
neutralize one already present on disk.

## A practical debugging checklist

If Codex appears to ignore an expected `AGENTS.md`, inspect the boundary before rewriting the
instructions:

1. Confirm the actual working directory for the task.
2. Map the ancestor path from that directory upward.
3. Look for the nearest configured root marker, including an unexpected `.git` file or
   directory.
4. Compare that boundary with the location of the expected `AGENTS.md`.
5. Check whether `project_root_markers` has been customized or disabled.
6. Use Git itself to test repository recognition, for example:

   ```powershell
   git rev-parse --show-toplevel
   ```

7. Compare the reported Git root with the filesystem marker you found. Do not infer repository
   validity from the marker's name, Hidden attribute, or existence alone.
8. Start a fresh Codex run or session after changing a marker so the retest uses a newly built
   instruction chain rather than retained context.

Do not delete a suspicious `.git` merely because it looks empty. First establish whether it is
real repository metadata, a worktree pointer, a submodule marker, or an unintended empty
artifact. The distinction in this article is a diagnostic model, not a blanket cleanup rule.

## What not to conclude

This investigation does **not** support any of these shortcuts:

- Every path named `.git` is a usable Git repository.
- Codex validates a root marker by running Git.
- Every `AGENTS.md` anywhere under a broad workspace is active.
- A Codex Project in the interface is the same concept as a filesystem project root.
- The old Windows creation behavior is present in every current workflow.
- The old Windows creation behavior is completely fixed everywhere.
- One controlled experiment is a guarantee for all future Codex versions.

It also disproved one tempting Windows heuristic: the Hidden attribute is not a reliable way to
separate real from fake `.git` directories. A valid repository can have a non-Hidden `.git`.
Ask Git, and inspect the metadata when necessary.

## The takeaway

In a normal repository, Git's root and Codex's discovery root align. That convenience hides the
fact that they answer different questions.

When behavior becomes surprising, identify three things:

```text
1. Working directory
2. Project root
3. Root markers
```

Then trace the `AGENTS.md` ancestry between the first two.

The durable mental model is simple: **Git repository validity is not the same question as Codex
project-root detection.** A `.git` entry can carry discovery meaning for Codex even when Git
does not recognize usable repository metadata there.

## Sources and provenance

- **Documented behavior:** [Codex advanced
  configuration](https://learn.chatgpt.com/docs/config-file/config-advanced) and [custom
  instructions with `AGENTS.md`](https://learn.chatgpt.com/docs/agent-configuration/agents-md),
  re-checked August 22, 2026.
- **Source-code-backed behavior:** pinned Codex source snapshots for
  [`agents_md.rs`](https://github.com/openai/codex/blob/bd19459358f534ed1cae464ec13d56600aeb45f2/codex-rs/core/src/agents_md.rs)
  and Windows sandbox
  [`win.rs`](https://github.com/openai/codex/blob/a26f219f6788c951dcb3bf435fab4c6d0f4d2f40/codex-rs/windows-sandbox-rs/src/bin/setup_main/win.rs),
  plus the current
  [`agents_md_tests.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/agents_md_tests.rs).
- **Direct observation:** the sanitized A/B/C/D boundary experiment and historical local
  filesystem audit described above.
- **Issue-report evidence:** OpenAI Codex issues
  [`#29492`](https://github.com/openai/codex/issues/29492),
  [`#32872`](https://github.com/openai/codex/issues/32872),
  [`#34799`](https://github.com/openai/codex/issues/34799), and
  [`#25651`](https://github.com/openai/codex/issues/25651).
- **Git behavior:** official [`git init`](https://git-scm.com/docs/git-init) documentation.
- **Synthesis:** “the hidden second job of `.git`” and “inheritance follows ancestors, not
  arbitrary siblings” are explanatory formulations derived from the evidence above, not
  official product terms.
