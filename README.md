# eng — an accountability layer for agent-driven engineering

Coding agents are good at producing changes. They are not good at being **accountable** for them.

`eng` is a small control plane that sits *above* whatever agent runtime you use, and makes four
things answerable at any moment: what was approved, what is actually running, what authority it
holds, and whether it really did what it promised.

Terminal-first. **Nothing here is shipped.** The semantics below were proved end to end by a
throwaway TypeScript slice; the implementation is being rebuilt in Rust, and that work has not
started. Read this as a design with a working proof behind it, not as software you can install.

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

### Seeing the system, not reconstructing it from logs

A wall of state transitions is not understanding. The engineering model, the workflow DAG and the
state machines are meant to be **looked at** — that is the `Rich Visual` plane in the architecture,
and it is designed, not built (see `CRITICAL-PATH.md`).

What *is* built is the property that would make such a view worth trusting: **the diagrams are
checked against the code.** `docs/diagrams.md` carries the WorkflowRun / StepRun / HumanGate state
machines, and a test asserts that every state in the implementation appears in the diagram that
claims to describe it. Add a state to the code and not to the picture, and the test fails. A diagram
that can silently fall behind the system is worse than no diagram, because people believe it.

### Unknowns are scheduled, not handled "before the real work"

Research is not a phase that happens before planning. It is a task type *inside* the plan
(`RESEARCH`, `SPIKE`), and an unresolved question is a first-class object carrying a blocking class.

* A plan holding a `BLOCKING` open question **refuses to compile**. You cannot execute your way past
  a decision you have not made. Surveying prior art and competing approaches for a design choice is
  work the plan schedules and the compiler waits on — mechanically, not as a reminder someone reads.
* Findings come back as **registered outputs on a run that is pinned to the exact plan revision**, so
  the decision and the evidence behind it stay attached to each other instead of drifting into a
  chat log.

The agent that actually performs that investigation is step 1 of `CRITICAL-PATH.md`: today the task
type, the blocking question and the compile-time refusal exist; the capability that fills them does
not.

---

## What a run looks like

The walkthrough below was run end to end against the TypeScript slice, before the Rust decision.
It is a record of something that happened, not a command you can run today.

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

## Honest scope

**There is no implementation right now.** A TypeScript vertical slice (2026-08) proved the
semantics and produced the trace above; the decision to build the real thing in Rust means the
codebase starts from zero. Everything below describes what the slice demonstrated, not what you
can run today:

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

If a claim isn't in this list, assume it isn't implemented yet — and assume everything in it
has to be built a second time, in Rust, before any of it is real again.

What is missing, and in what order it has to change:
**[CRITICAL-PATH.md](CRITICAL-PATH.md)**.

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

**見えること・調べること** — 状態機械の図（`docs/diagrams.md`）は**テストでコードと照合**される。
実装にある状態が図に無ければテストが落ちるので、図が黙って実装から遅れることがない
（黙って古くなる図は、無い図より悪い。人は信じてしまうため）。図を視覚として見る `Rich Visual` 面は設計済み・未実装。
調査は「本番前の下調べ」ではなく plan の中の task type（`RESEARCH`/`SPIKE`）で、未解決の問いは blocking class を持つ
第一級オブジェクト。**`BLOCKING` な open question を抱えた plan はコンパイルを拒否される** —
決めていない判断を実行で追い越せない。競合・先行事例の調査は plan が積むタスクで、コンパイラが機械的に待つ。
成果は「厳密な plan revision に pin された run の登録済み出力」として戻るので、判断とその根拠が離れない。

**現状 — 実装は現在ゼロ。** TypeScript の vertical slice（2026-08）が上記の意味論を end-to-end で証明したが、
本実装を Rust で作る判断をしたため、コードベースはゼロから始まる。以下はスライスが示したことであって、
今動かせるものではない。スライスでの既定 agent runtime は fixture 駆動の fake、実 runtime は
「実 `git` を起動し、限定されたローカル repo のブランチにコミットする」1 provider のみだった。
セキュリティ主張は狭いが非ゼロ（OS/kernel サンドボックスではなく、プロダクト境界での code-level 封じ込め）。
汎用シェル実行・認証境界は対象外。

**この先の順序**: [CRITICAL-PATH.md](CRITICAL-PATH.md)。
