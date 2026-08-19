# eng — an accountability layer for agent-driven engineering

Coding agents are good at producing changes. They are not good at being **accountable** for them.

`eng` is a small control plane that sits *above* whatever agent runtime you use, and makes four
things answerable at any moment: what was approved, what is actually running, what authority it
holds, and whether it really did what it promised.

Terminal-first. TypeScript on Node ≥ 22.5. Zero dependencies, no build step.

---

## The pain

Hand real work to an agent and the same four failures show up — not because the model is weak,
but because nothing outside the model is keeping score.

**1. Intent drifts silently.**
You approve a plan. The plan gets edited. The agent keeps executing. Nothing records *which
version of the intent* the running work is bound to, so "did it build what I approved?" has no
answer — only a diff and a memory.

**2. "The agent said it's done" is the only completion signal.**
Exit code 0 is treated as success. Whether the step produced the artifacts it promised, and
whether the verification it was supposed to pass actually passed, is checked by nobody but the
agent narrating its own work.

**3. Authority is granted per-process, not per-task.**
An agent that needs to touch one branch gets a session that can touch everything, and the
containment strategy is *asking it nicely in the prompt*. Capability ("can do"), trust
("we vetted it"), and permission ("may do, here, now") are collapsed into one blob.

**4. Runs are not durable.**
Close the terminal, lose the session. Restart, and in-flight work is either blindly retried
(dangerous — it may have already committed) or silently dropped.

These get **worse** as models get better, because capability raises the amount of work you are
willing to hand off before you can verify any of it. The bottleneck stops being "can it write the
code" and becomes "can I trust what came back without re-reading all of it".

---

## How `eng` solves it

Four mechanisms, one per pain. All of them live in the product, outside the agent's reasoning.

### 1. Approval binds an exact revision — the *Intent Baseline*

Plan revisions are **immutable snapshots**. Approval targets one exact revision, and that becomes
the plan's `approved_revision`. Editing a plan creates revision *n+1*; it never mutates the
approved one, and it never silently promotes itself.

* The workflow compiler **refuses to compile a draft** — only the approved revision compiles.
* A run **pins** `(plan revision, workflow revision, policy revision)` at start and executes
  against those pins for its whole life.
* Compilation is deterministic and validated before it is stored: every task covered, every
  required verification represented, every gate inserted, no dangling dependencies, no cycles,
  no `BLOCKING` open questions left.

So "what did I approve" and "what is running" are two queries, not two guesses.

### 2. Step success is a contract check, not the agent's opinion

Every step carries a `StepContract`: the outputs it must produce, the verifications it answers to,
and its retry policy. After the agent returns, the runtime checks the contract itself:

* **process success ≠ step success** — an agent can exit `SUCCESS` and still fail the step if the
  declared outputs were never registered (`VERIFICATION_FAILURE`).
* Verification steps are **compiled into the workflow** from the plan's verification requirements
  and checked against registered evidence — not against a claim in a log line.
* A run reaches `COMPLETED` only when every required step succeeded **and** no required
  verification is still `PENDING`, `FAILED`, or `UNKNOWN`.
* Failure is classified (`AGENT_FAILURE`, `TOOL_FAILURE`, `POLICY_FAILURE`, `VERIFICATION_FAILURE`,
  `UNKNOWN_OUTCOME`, …), and only the classes the contract opted into are retried.

### 3. Authority is computed by the product, then enforced outside the agent

`eng` keeps **capability**, **trust**, and **permission** as three separate things, and derives the
third from the first two:

* A provider is selected by **hard filters first** — capability match, trust threshold, blocklist,
  provider-type constraints, availability, and the plan's permission ceiling. Only legal candidates
  are ranked, and ranking prefers **least privilege**.
* Effective permission = *what the provider offers* ∩ *what the step requires* ∩ *the plan's
  permission ceiling*. Nothing wider is ever minted.
* Policy is **deterministic typed rules** evaluated under a pinned policy revision, returning
  `ALLOW` / `REQUIRE_HUMAN` / `DENY`. There is no path where an agent talks its way past a rule.
* The final guard, `authorize()`, is **product-owned** and sits behind the runtime boundary, so a
  future runtime adapter cannot forget to call it. It judges the operation by what it is *about to
  touch* — the branch and the write paths — never by what the process reports about itself.

### 4. State is durable, and unknown stays unknown

A persistent daemon owns all state (append-only event log + revisioned object store with optimistic
concurrency). The CLI owns **none** — it attaches and streams.

* Ctrl-C detaches; the run keeps going. `eng .` reattaches and replays.
* On daemon restart, steps that were mid-flight are marked `UNKNOWN_OUTCOME` and are
  **not retried automatically** — an interrupted step may already have had an effect, and guessing
  is how you get double writes.
* Human gates have **no timeout path**. Nothing is approved by waiting, and rejection is a valid
  terminal state, not an error to be worked around.

### Where the human sits

Gates are first-class, durable objects — `PLAN_APPROVAL`, `PERMISSION_ESCALATION`, `MERGE`,
`DEPLOYMENT`, `RISK_ACCEPTANCE` and friends. The default policy already requires a human for merge
and deployment, and denies `admin` outright. A step that needs more authority than it holds does
not fail and does not proceed — it opens a gate, waits, and re-dispatches only if a human approves.

`eng attention` answers the only question that matters when you come back to the terminal:
**what needs me?**

---

## What a run looks like

The walkthrough below is the fixture-driven demo, verified end to end. The implementation lives in
a private repository for now; this repo is the public overview.

```text
$ eng demo fixtures/async-payment-retry.json
plan plan_ed9fb93fe53e V1 created (session ss_019ea6c7b941)
plan V1 approved via gate gate_156a98ae6b64 (Intent Baseline)
workflow wfd_4a1874330c3f W1 compiled: step:T-1 → verify:VR-1 → gate:MERGE
run run_b73fcb85262d started (pins W1 / V1)

$ eng status
runs:  completed=0 running=0 waiting=1
  run_b73fcb85262d WAITING  W1/V1  step:T-1=SUCCEEDED verify:VR-1=SUCCEEDED gate:MERGE=WAITING  adaptations=1
needs attention:
  [GATE] MERGE needs your decision  → gate gate_b3049fb197c6

$ eng gate approve gate_b3049fb197c6 "ok"
$ eng status
runs:  completed=1 running=0 waiting=0
  run_b73fcb85262d COMPLETED  W1/V1  step:T-1=SUCCEEDED verify:VR-1=SUCCEEDED gate:MERGE=SUCCEEDED  adaptations=1
```

Three things in that trace are the whole point: the run is **pinned** to `W1/V1` (not to "the plan"),
`adaptations=1` is a provider that failed and was **rebound at runtime** without a human, and the run
sat in `WAITING` on a MERGE gate rather than finishing itself.

---

## Critical path

Everything above is either implemented or explicitly scoped out. What stands between this and being
real is a short, **ordered** chain — each link is blocked by the one before it.

**1. One real agent through the facade.** ← *we are here*
The default runtime is a fake. Until a real coding agent runs a real step through `dispatch()` and
has its outputs contract-checked, every guarantee on this page is a guarantee about a simulation.
The hard part is not spawning the agent — it is the **output boundary**. The product has to observe
the artifacts and change sets itself (files on disk, commits on a branch) instead of accepting a
list the agent hands back. *An agent that can name its own outputs can name outputs it never
produced* — and the contract check collapses straight back into pain #2.

**2. Verification that executes, not verification that checks presence.**
Today a verification step passes when the declared evidence artifact was **registered**. That proves
the evidence *exists* — not that it *says pass*. Real verification has to run the repository's own
check and record its result as the evidence. Until then, "no required verification is pending" is a
weaker sentence than it reads, and pain #2 survives one level further down.

**3. The scope decision on execution surface.**
`authorize()` is product-owned and enforced outside the agent, but the only surface it enforces
today is git branch and write paths. A real agent wants general command execution, and at that
moment code-level containment stops being sufficient — an OS-level boundary is required. Two honest
options: keep the surface deliberately narrow, or take on a sandbox. "Neither, but claim both" is
where security theatre starts. This decision bounds how far 1 is allowed to go.

**4. Gates that show what they are gating.**
A human approving a MERGE gate today sees a gate id, not a diff. Approval without evidence in front
of it is a rubber stamp — which re-creates the original pain one layer up, at the human. Gates have
to render the change they are holding.

**Not on the path, but next:** concurrency. One step executes at a time. Parallel agents are the
point of the word *workspace*, and nothing in the model prevents it — but that is throughput work,
not validity work, and it can wait.

The order is load-bearing: **2 is worthless before 1, and 4 is cosmetic before 2.** If exactly one
of these ships, it is 1 — it is the only one that can still falsify the design.

---

## Honest scope

This is a **bootstrap-stage vertical slice** (2026-08), not a product. Specifically:

* The default agent runtime is a **fake, fixture-driven runtime** behind the runtime boundary. It
  exists to prove the semantics — approval binding, contract checking, gates, adaptation, recovery —
  without a model in the loop.
* A **real runtime** exists as exactly one provider: it spawns real `git` and performs
  branch-scoped commits into a bounded local repository. That is the whole of it.
* The **security claim is narrow but non-zero**: enforcement is code-level containment at a product
  boundary, verified by tests that try to escape it. It is *not* an OS/kernel sandbox. General shell
  and arbitrary command execution are out of scope and unimplemented. Credential and auth boundaries
  are out of scope.
* `eng` is **not** a coding agent, an IDE, or a model wrapper. It makes no model calls of its own.

If a claim isn't in this list, assume it isn't implemented yet.

---

## 日本語要約

**痛み** — エージェントは「変更を作る」のは得意になったが、「その変更に責任を持つ」仕組みが外側に無い。
結果として (1) 承認した意図と実行中の意図が静かにズレる、(2) 完了の唯一の根拠がエージェントの自己申告になる、
(3) 権限がタスク単位ではなくプロセス単位で渡され、封じ込めが「プロンプトでお願いする」になる、
(4) 実行が永続化されず、再起動で盲目的な再試行か消失のどちらかになる。
モデルが賢くなるほど**任せる量が増える**ので、この 4 つは悪化する。

**解決** — 4 つとも、エージェントの推論の外側（プロダクト側）で機械的に閉じる。

1. **承認は特定 revision に束縛される**。Plan revision は不変スナップショットで、承認済み revision だけがコンパイルでき、
   run は plan / workflow / policy の revision を pin して走る。「承認したもの」と「動いているもの」が両方クエリで答えられる。
2. **step の成功は契約検査**。プロセスの成功 ≠ step の成功で、宣言した成果物が登録されていなければ失敗扱い。
   検証要件は workflow の step としてコンパイルされ、必須検証が通るまで run は COMPLETED にならない。
3. **権限はプロダクトが計算し、プロダクトが強制する**。capability / trust / permission を分離し、
   ハードフィルタ → 最小権限ランキングの順で provider を選び、実効権限は ceiling との積集合。
   policy は pin された revision 上の決定的ルール（DENY / REQUIRE_HUMAN）で、エージェントの主張では動かない。
   最終ガードは「これから触るブランチと書き込みパス」から判定し、自己申告を信用しない。
4. **状態は永続、不明は不明のまま**。daemon が状態を持ち CLI は持たない。Ctrl-C しても run は生きる。
   再起動時の実行中 step は `UNKNOWN_OUTCOME` として**自動再試行しない**。human gate に時間切れ承認は存在しない。

**クリティカルパス（順序が本質）** — 1. **実エージェントを facade の向こうに 1 本通す**（現在地。難所は出力境界 —
成果物はエージェントの申告ではなくプロダクト自身が観測しなければ、自分の成果物を自分で名乗れてしまう）→
2. **検証を「存在確認」から「実行」へ**（今は宣言した証拠が登録済みなら通る。中身が pass かは見ていない）→
3. **実行面のスコープ判断**（今の強制対象は git のブランチと書き込みパスのみ。汎用コマンド実行を許した瞬間に
code-level では足りず OS 境界が要る。狭いまま行くか、サンドボックスを背負うか。「両方主張する」はセキュリティ演劇）→
4. **gate が中身を見せる**（今は gate id しか見えない。差分の見えない承認は形骸化で、痛み #2 が人間の層で再発する）。
並列実行は経路上ではなく次の話（正しさではなくスループット）。**2 は 1 の前では無意味、4 は 2 の前では化粧**。
1 本だけ出すなら 1 — 設計を反証できるのはそれだけ。

**現状** — bootstrap 段階の vertical slice。既定の agent runtime は fixture 駆動の fake、実 runtime は
「実 `git` を起動し、限定されたローカル repo のブランチにコミットする」1 provider のみ。
セキュリティ主張は狭いが非ゼロ（OS/kernel サンドボックスではなく、プロダクト境界での code-level 封じ込め）。
汎用シェル実行・認証境界は対象外。
