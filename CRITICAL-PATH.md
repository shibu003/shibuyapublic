# Critical path

Companion to the [README](README.md), which describes what `eng` is and what it already does.
This file is the other half: **what it does not do yet, and in what order that has to change.**

Reviewed 2026-08-19.

---

Everything in the README is either implemented or explicitly scoped out. What stands between the
current bootstrap slice and something real is a short, **ordered** chain — each link is blocked by
the one before it.

## 1. One real agent through the facade ← *we are here*

The default runtime is a fake. Until a real coding agent runs a real step through `dispatch()` and
has its outputs contract-checked, every guarantee in the README is a guarantee about a simulation.

The hard part is not spawning the agent — it is the **output boundary**. The product has to observe
the artifacts and change sets itself (files on disk, commits on a branch) instead of accepting a
list the agent hands back. *An agent that can name its own outputs can name outputs it never
produced* — and the contract check collapses straight back into the pain it was built to remove.

**Done when:** a real agent completes a step, and a step it *claims* to have completed but did not
is caught by the product without reading the agent's transcript.

## 2. Verification that executes, not verification that checks presence

Today a verification step passes when the declared evidence artifact was **registered**. That proves
the evidence *exists* — not that it *says pass*.

Real verification has to run the repository's own check and record its result as the evidence. Until
then, "no required verification is pending" is a weaker sentence than it reads, and self-reported
completion survives one level further down.

**Done when:** a step whose tests fail cannot reach `SUCCEEDED`, and the failing output is the
evidence on record.

## 3. The scope decision on execution surface

`authorize()` is product-owned and enforced outside the agent, but the only surface it enforces
today is git branch and write paths. A real agent wants general command execution, and at that
moment code-level containment stops being sufficient — an OS-level boundary is required.

Two honest options: keep the surface deliberately narrow, or take on a sandbox. "Neither, but claim
both" is where security theatre starts. This decision bounds how far 1 is allowed to go, so it is
not deferrable past 1.

**Done when:** the answer is written down and the README's security claim matches it exactly.

## 4. Gates that show what they are gating

A human approving a MERGE gate today sees a gate id, not a diff. Approval without evidence in front
of it is a rubber stamp — which re-creates the original problem one layer up, at the human.

This is also the first real slice of the `Rich Visual` plane: the workflow DAG, the engineering
model and the state machines are supposed to be *seen*, not reassembled from an event stream. None
of that is built. Gates are where it has to start, because a gate is the one moment the human is
already stopped and looking.

**Done when:** a gate renders the change set and verification results it is holding, in the terminal,
before the decision is taken.

---

## Not on the path

**Concurrency.** One step executes at a time. Parallel agents are the point of the word *workspace*,
and nothing in the model prevents it — but that is throughput work, not validity work. It can wait,
and doing it early only multiplies whatever 1 and 2 get wrong.

---

## Why this order

**2 is worthless before 1** — executing verification against a fake runtime verifies the fixture.
**4 is cosmetic before 2** — rendering evidence that was never checked just makes a rubber stamp
prettier. **3 bounds 1** rather than following it, because the scope of what a real agent may do
decides what "real" means in step 1.

If exactly one of these ships, it is **1**. It is the only one that can still falsify the design.

---

## 日本語

README が「何であるか・すでに何をするか」で、こちらは「まだ何をしないか・どの順で変わるか」。

1. **実エージェントを facade の向こうに1本通す**（現在地）
   難所は spawn ではなく**出力境界**。成果物はエージェントの申告リストではなく、プロダクト自身が観測しなければならない。
   自分の出力を自分で名乗れる相手は、作っていない出力も名乗れる。そこを許すと、契約検査は元の痛みにそのまま戻る。
   *完了条件*: 実エージェントが step を完了できること、かつ「完了したと主張したが実際はしていない」step を、
   transcript を読まずにプロダクト側が捕まえられること。

2. **検証を「存在確認」から「実行」へ**
   今は宣言した証拠が登録済みなら通る。証拠が「ある」ことは見ているが、中身が pass かは見ていない。
   *完了条件*: テストが落ちる step が `SUCCEEDED` に到達できず、その失敗出力が証拠として記録されること。

3. **実行面のスコープ判断**
   今の強制対象は git のブランチと書き込みパスのみ。汎用コマンド実行を許した瞬間に code-level では足りず、OS 境界が要る。
   狭いまま行くか、サンドボックスを背負うか。「どちらもやらないが両方主張する」がセキュリティ演劇の入り口。
   1 の後ではなく **1 の範囲を決める**ので、1 より後には送れない。
   *完了条件*: 判断が明文化され、README のセキュリティ主張がそれと完全に一致していること。

4. **gate が中身を見せる**
   今は gate id しか見えない。差分の見えない承認は形骸化で、元の問題が人間の層で再発する。
   これは `Rich Visual` 面（workflow DAG・エンジニアリングモデル・状態機械を「見る」層。未実装）の最初の一切れでもある。
   gate から始めるべき理由は、そこが人間が既に立ち止まって見ている唯一の瞬間だから。
   *完了条件*: gate が、保留している change set と検証結果を、決定の前にターミナル上で描画すること。

**経路外**: 並列実行。正しさではなくスループットの話で、1 と 2 の誤りを早期に増幅するだけ。

**順序の理由**: 2 は 1 の前では fixture を検証するだけ。4 は 2 の前では形骸化した承認を綺麗にするだけ。
3 は 1 に続くのではなく 1 を**縛る**（実エージェントに何を許すかが「実」の意味を決めるため）。
1本だけ出すなら **1** — 設計を反証できるのはそれだけ。
