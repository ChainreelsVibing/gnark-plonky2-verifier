---
name: "chainreels-factory"
description: "Use for any code change in the Chainreels repos (HyperL2, ZKArch, RGS, HyperL2Verifier, docs): runs it through the spec, isolate, build, prove, refute, ship pipeline with money-path gates."
---

# Chainreels software factory

Chainreels is an on-chain gaming platform that will hold real deposits, and nearly all its code
is AI-written. The 2026-11 audit showed the result of shipping that unchecked: withdrawals that
debit and cannot release, tables paying 101%, amounts in the wrong unit, features never run end
to end. Code is the cheap part. The factory's product is a small pull request whose claims were
run, attacked by a second agent, and tied to a written spec.

Every change goes through six stages. Never skip one. If a stage cannot finish, stop and say
what is missing. Never write "tested" for something that was only read.

| Stage | Output | Stop when |
|---|---|---|
| 1 Spec | Testable criteria, tier, units | A criterion cannot be tested |
| 2 Isolate | Own worktree and branch | An open PR changes the same behaviour |
| 3 Build | Slices of at most 200 changed lines | The change cannot be split |
| 4 Prove | Executed evidence per criterion | Anything was not actually run |
| 5 Refute | A fresh agent's verdict | Any criterion is refuted |
| 6 Ship | PR, CI green, reviewers asked | CI is red or a gate is unmet |

Reviews, explanations and investigations need none of this until they turn into an edit.

## Tiers

Pick in the spec; when unsure, pick the higher.

- **T0** docs, comments, tests only: checks pass, one human approval.
- **T1** backend code that cannot move value, change who may act, or affect an outcome: T0 plus
  evidence and a refute pass.
- **T2** money paths: T1 plus a failing test written first, an end-to-end run on a local chain,
  the invariant checklist, and the architect's approval. Never merged by an agent.

**Money paths (always T2):** balances, ledger, treasury, settlement, solvency guards; deposits,
bridge indexing, withdrawals, burns, checkpoints, force exit; faucet, campaigns, rewards, VIP,
referrals, bonus tokens (gHYPE), lottery, claims roots; bet placement, round state machine,
payout, rounding, RTP tables, game config; VRF, commit-reveal, anything that serializes a draw;
login, tokens, authorization, admin/sequencer/internal routes, rate limits; every contract,
deploy script, circuit and verifier; CI workflows, Dockerfiles, compose, env and secrets; public
fairness or custody claims in `docs`.

## 1 Spec

Written before any code, kept at the top of the PR. T2 needs a person's approval before Build.

```
Task:      board item or issue link
Problem:   one or two sentences
Tier:      T0 | T1 | T2, and why
Criteria:  numbered, each "given X, when Y, then Z"
Units:     each amount touched, its unit, its maximum
Flags:     switches read or changed, and their production values
Not doing: what is deliberately left alone
Rollback:  flag, revert, or migration down
```

## 2 Isolate

`git fetch origin`; check `gh pr list` and `gh pr diff <n> --name-only` for overlap (same
behaviour: stop and ask; same file, independent edit: proceed); then
`git worktree add ../wt/<task>-<id> -b factory/<task>-<id> origin/main`. One worktree and branch
per task and per agent. Never commit to `main`, never touch another task's worktree, only
`--force-with-lease` on your own branch, regenerate lockfiles instead of merging them.

## 3 Build

- **200 changed lines per PR at most; aim under 100** (the elite benchmark), excluding generated
  files and lockfiles. Bigger work is a stack of PRs, each shippable alone.
- **Use the existing layout.** HyperL2: `src/api` → `src/services` → `src/domain` (pure rules) and
  `src/infrastructure`/`src/repositories`; jobs in `src/jobs`. New layers go in the spec first.
- **T2: test first.** Commit the test that fails today, then make it pass; keep the failing output.
- **No dead features.** Any new route, job or contract function is called end to end in the same PR.
- **Plain text.** Commits, PR text and comments: short active sentences, no filler, no emoji.

## 4 Prove

Run it and paste real output. A claim without executed evidence is deleted, not softened.

| Repo | Checks |
|---|---|
| HyperL2 | `pnpm install --frozen-lockfile`, `pnpm lint`, `pnpm build`, `pnpm test` |
| ZKArch Rust | `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace` |
| ZKArch contracts | `forge build`, `forge test -vv` in `contracts/` |
| RGS, HyperL2Verifier | the repo's `lint`, `build`, `test` scripts |
| docs | every example run; every claim checked against the code |

- **T0:** check output. **T1:** plus a before/after pair for the changed behaviour.
- **T2:** plus this scenario on `anvil` with local MongoDB and Redis (`pnpm infra:up`) and the
  relevant switches on:
  1. Deposit on the local L1; the credit appears in the stated unit.
  2. Bet; settle a win and a loss; balances and ledger rows are right after each.
  3. Withdraw; it is burned, released and finalized.
  4. Repeat the changed step with the same request key; nothing changes the second time.
  5. Run the changed step from two processes at once; totals still balance.

  If an earlier defect blocks the scenario, name it and stop. The task is not done.

## 5 Refute

A second agent with a fresh context gets the spec, diff and evidence, and nothing else. It opens
every changed file and what it calls; for each criterion looks for what would make it false (a
guard elsewhere, a type bound, another code path in use, a retry, a second process); recomputes
every number; walks the checklist for T2. It answers per criterion `HOLDS`,
`HOLDS WITH CORRECTION`, `REFUTED` (file and line) or `CANNOT DETERMINE` (which step).

`REFUTED` returns the task to Build. After three rounds, stop and hand both agents' notes to a
person; do not raise the cap. The author never refutes its own work; the refuter never fixes.

## 6 Ship

Rebase on `origin/main`, rerun checks, push, open the PR with: **Spec** (unchanged), **What
changed** (three to six sentences), **Evidence**, **Refute verdict**, **Risks and follow-ups**.
Link the board item. Reviewers: a teammate for T0/T1, the architect for T2. Never merge on a red
or missing CI run, never approve your own work, never merge unless a person says so. Give the
developer their check-out line for the reporting bot: `DONE: <outcome> #<pr>` / `NEXT: <first thing>`.

## Invariant checklist (T2)

Each names a failure the audit found. Show with a test that it still holds.

1. **Units.** Every amount has a named unit: play balances are cents, withdrawable balances are
   raw L1 base units (wei on the native bridge). Convert only at a boundary, range-check before
   storing, `bigint` never `number`. Binding limits: 64-bit storage, checkpoint total below about
   18.45 ETH in wei, 4,096 accounts, bets below 2^30 base units.
2. **Ledger.** Debit and credit in one transaction; request keys from server-side ids; a retry
   changes nothing; burns and mints have replay guards in backend and contract.
3. **Outcome secrecy.** The outcome is the VRF output alone, so nothing derived from it reaches a
   player before their last payout-affecting choice, and no timeout or refund is cheaper than a loss.
4. **Free play.** Bonus-token rounds reserve at placement, record losses, and never count as
   real-money volume for bonuses, VIP, referrals or points.
5. **Return to player.** No table above the cap takes real-money stakes; recompute it in a test.
6. **Authorization.** Each state-changing route and each read of another player's data has a
   test that the wrong caller is refused. No dev token or test hook outside a test build. Runtime
   secrets cannot change `NODE_ENV`.
7. **Chain events.** Every indexer handler checks the emitting contract.
8. **Contracts.** Access control on every external function, replay guards on ids,
   checks-effects-interactions, no key holding several roles.
9. **Switches.** Money-path flags default off; turning one on carries the T2 scenario with it on.
10. **Secrets and CI.** No secret reaches a PR-triggered job; none printed or committed; deploy
    jobs sit behind an environment with required reviewers.
11. **Public claims.** Docs say only what the code and verifier check.

## Running agents in parallel (Sandcastle)

For a batch, use [`@ai-hero/sandcastle`](https://github.com/mattpocock/sandcastle). Check its
current docs for option names; the shape is:

```ts
import * as sc from "@ai-hero/sandcastle";
import { docker } from "@ai-hero/sandcastle/sandboxes/docker";
const agent = sc.claudeCode(process.env.FACTORY_MODEL!);
const plan = await sc.run({ agent, sandbox: docker(), promptFile: ".sandcastle/plan-prompt.md",
  output: sc.Output.object({ tag: "plan", schema: Plan }) });   // non-overlapping, spec-approved tasks
for (const t of plan.output.tasks) {                            // at most 3 at a time
  await using box = await sc.createSandbox({ sandbox: docker(), branch: t.branch });
  for (let round = 1; round <= 3; round++) {
    await box.run({ agent, promptFile: ".sandcastle/implement-prompt.md", promptArgs: t });
    const v = await box.run({ agent, promptFile: ".sandcastle/refute-prompt.md", promptArgs: t,
      output: sc.Output.object({ tag: "verdict", schema: Verdict }) });
    if (v.output.criteria.every((c) => c.verdict.startsWith("HOLDS"))) break;
  }                                                             // then push and open the PR
}
```

- The implement and refute prompts are stages 3 to 5 of this skill with the spec inlined.
- **No merge phase.** The script opens PRs only.
- The sandbox gets a model credential and a GitHub token that can push branches and open PRs.
  No deploy key, RPC secret or production database address.
- T2 tasks are planned only when the spec carries a person's approval.
- Parallelism is set by what reviewers can read in a day, not by what machines can produce.

## People

- **Owner:** priorities on the board; approves what gets built.
- **Architect:** approves every T2 spec and T2 PR within one business day; owns the checklist.
- **Developers:** operate the line: write specs, read the evidence and verdict, review each
  other's T0/T1 work, answer for what ships under their name.

An agent's approval never counts as a person's. Every repo requires one human approval and
passing CI, for everyone.

## Fix first

Build nothing new on these until fixed; each is its own stack of small T2 PRs, in this order:
1. CI secrets reachable from PR jobs (the only High finding live today).
2. Units: one definition of a chip at the bridge boundary, with range checks.
3. Withdrawal release authority and the funding cap.
4. Force-exit request indexing and account locking.
5. Bonus-token accounting and its exclusion from bonuses and rewards.
6. Plinko tables above 100%, and a cap on real-money tables.
7. The integration test and production deploy workflow, which have never passed.

## Stop and report

When a criterion cannot be tested, an open PR conflicts in behaviour, the T2 scenario cannot run,
the refute loop hits three rounds, CI is red for an outside reason, or a step needs a secret the
sandbox lacks: say what was done, what was not, and what a person must decide.