# Judging candidates without held-out tests

Research for issue #3. Question: what does prior art offer for judging
candidate changes when there are no held-out tests, in a loop where several
candidates implement the same intent?

Scope of reading: papers and official documentation, fetched in November 2026.
Most papers were read at abstract level plus the specific sections cited;
where only the abstract was read, or a source could not be fetched, this note
says so. "Inference" marks conclusions drawn here for this loop that no source
states directly. Cost figures for this loop are inferences; the only measured
costs are the ones quoted from the papers.

## Short answer

1. No surveyed mechanism replaces held-out tests. Every published selector
   that works without ground-truth tests leaves a measurable gap to oracle
   selection, and the gap grows with the number of candidates.
2. The mechanisms that help most are execution-grounded: run things and
   compare behaviour. A reviewer that only reads code is weaker than the same
   reviewer given execution results.
3. The cross-test matrix and the attack agent are forms of differential
   testing with generated tests. Prior art supports them as a way to find
   *differences* between candidates. It does not support treating agreement
   between candidates as evidence of correctness, because candidates from the
   same model family share errors, and because generated tests are often wrong
   themselves.
4. Nothing surveyed detects an error in the intent. When every candidate
   implements the same wrong reading, all mechanisms below stay silent. The
   one mitigation with evidence is to show the developer a small number of
   behaviour-distinguishing tests and ask which behaviour is intended.

## What the oracle problem literature says

The test oracle problem is "the challenge of distinguishing the corresponding
desired, correct behaviour from potentially incorrect behavior". The standard
survey lists the automated substitutes (specifications, contracts, metamorphic
relations, pseudo-oracles built from agreement between independent
implementations, implicit oracles such as crashes) and states that when none
is adequate "the final source of test oracle information remains the human".
[The Oracle Problem in Software Testing: A Survey, IEEE TSE 2015](https://doi.org/10.1109/TSE.2014.2372785)
([full text read](https://web.eecs.umich.edu/~weimerw/2018-481/readings/testoracles.pdf)).

Inference: the loop already has that structure. Gates are implicit oracles,
the matrix is a pseudo-oracle, and approval is the human oracle. The research
question is how much the middle layer can be trusted.

## Mechanism by mechanism

### Differential and N-version testing (the cross-test matrix)

What it is: run several implementations on the same inputs and treat
disagreement as a signal. No expected output is needed.

Evidence that it finds real problems:

- PatchDiff generated tests that pass under one patch and fail under another.
  It flagged 29.6% of test-passing SWE-bench patches as behaving differently
  from the reference patch; 82.7% of those were not caught by running all
  developer tests; manual inspection judged 28.6% of the divergent patches
  certainly incorrect.
  [Are "Solved Issues" in SWE-bench Really Solved Correctly?, 2025](https://arxiv.org/abs/2503.15223)
- Clustering candidates by their outputs on generated inputs, then choosing
  between clusters, is the selection step in AlphaCode ("filtering based on
  program behavior", abstract only read)
  ([2022](https://arxiv.org/abs/2203.07814)), MBR-EXEC
  ([2022](https://arxiv.org/abs/2204.11454), abstract only read) and S\*
  ([2025](https://arxiv.org/abs/2502.14382)).
- S\* asks a model to write inputs that distinguish two clusters, runs both,
  and lets the model choose with the execution results in hand. On
  LiveCodeBench with 8 candidates this averaged 57.5 pass@1 against 55.6 for
  an LLM judge without those executions and 53.1 for voting with generated
  tests. Runs were single and margins are 1 to 2 points.

Evidence against treating agreement as correctness:

- The 1986 multiversion experiment built 27 versions from one specification
  and ran one million tests. Versions were individually reliable, "but the
  number of tests in which more than one program failed was substantially
  more than expected" under independence.
  [An Experimental Evaluation of the Assumption of Independence in Multiversion Programming, IEEE TSE 1986](https://doi.org/10.1109/TSE.1986.6312924).
  Only the abstract was verified; the full text was not obtained.
- Across more than 350 language models, models "agree 60% of the time when
  both models err" on one leaderboard dataset; shared provider and shared
  architecture raise error agreement, and more accurate models have more
  correlated errors even across providers.
  [Correlated Errors in Large Language Models, ICML 2025](https://arxiv.org/abs/2506.07962)
- CodeT ranks candidates by agreement between generated solutions and
  generated tests. Its own failure analysis: on HumanEval with one model, 53
  of 164 problems had a correct solution that was not ranked first; most were
  attributed to the model misunderstanding the problem, which produces wrong
  solutions and wrong tests that agree with each other. One documented case
  had 20 incorrect solutions outvoting 9 correct ones.
  [CodeT: Code Generation with Generated Tests, 2022](https://arxiv.org/abs/2207.10397)

Catches: policy and edge-case differences between candidates; regressions one
candidate introduces and another does not.
Misses: anything all candidates get wrong in the same way, including a wrong
intent.
Cost (inference): compute only, quadratic in candidates times tests; no extra
model calls unless a model is asked to interpret the disagreements.

### Generated tests as an oracle (candidate tests, attack agent)

What the evidence says about tests written by a model without ground truth:

- Agentless generated reproduction tests for 300 issues. 213 yielded a test
  that reproduced the issue on the original code; only 94 of those also
  passed with the reference fix applied. Even so, selection with these tests
  resolved 96 issues against 77 for majority voting alone and 81 with
  regression tests added. Validation cost $0.25 of $0.70 per issue.
  [Agentless, 2024](https://arxiv.org/html/2407.01489v2)
- SWT-bench kept a fix only if the agent's own generated tests went from fail
  to pass. Precision more than doubled, to 47.8%, at 20% recall.
  [SWT-Bench, 2024](https://arxiv.org/abs/2406.12952)
- Of patches that passed model-generated tests, 21.8% (Claude 3.7 Sonnet) and
  33.0% (GPT-4o) failed the hidden tests or existing regression tests.
  Letting the agent refine its patch against the generated tests raised that
  to 25.5% and 35.9%; of 22 patches newly passing after refinement, 14 failed
  the hidden tests.
  [Investigating Test Overfitting on SWE-bench, 2025](https://arxiv.org/abs/2511.16858)
- Models asked to judge test assertions follow the code in front of them.
  Classification accuracy was 52% to 69% overall, and accuracy on correct
  assertions fell when the code under test was buggy (for one model from
  55.17% to 39.02%). The authors conclude that LLM test generation "mainly
  capture[s] the actual program behavior making bug detection difficult".
  [Can we find bugs using LLM-generated oracles?, 2024](https://arxiv.org/abs/2410.21136)
- In S\*, voting with model-generated tests scored slightly below using the
  public tests alone; the authors attribute this to errors in the predicted
  outputs. ([2025](https://arxiv.org/abs/2502.14382))
- CodeMonkeys used generated tests to shortlist three of ten candidates and
  then a model to choose. With 69.8% of issues having at least one correct
  candidate and 45.8% expected from random choice, selection reached 57.4%,
  "approximately half" of the gap. Test generation was 19.2% of total cost
  and selection 5.8%.
  [CodeMonkeys, 2025](https://arxiv.org/abs/2501.14723)

Inference for the attack agent: an attack test that fails on some candidates
and passes on others is useful evidence whichever side is right, because it
names a behaviour difference for the reviewer and the developer. An attack
test that fails on every candidate is ambiguous: either all candidates share a
defect or the test encodes a wrong expectation, and the cited results say the
second is common. An attack agent that reads candidate code before writing
tests is exposed to the "asserts actual behaviour" bias; one that writes from
the intent alone is exposed to misreading the intent. No source compares the
two set-ups.

Catches: failures to do what the intent literally asks; crashes; behaviour
differences.
Misses: shared misreadings; can reject correct candidates (20% recall in the
strictest published filter).
Cost: the largest model spend among the mechanisms here (a quarter to a third
of pipeline cost in the two systems that report it).

### Existing tests as a regression gate

- Trae Agent discards candidates that fail model-selected existing tests.
  The filter wrongly discarded correct patches in 3.69% of cases, with
  precision 61.20% and recall 93.40%.
  [Trae Agent, 2025](https://arxiv.org/html/2507.23370)
- Agentless gained 4 resolved issues from this step at $0.01 per issue.
  ([2024](https://arxiv.org/html/2407.01489v2))
- Running the whole existing suite matters: 7.8% of SWE-bench patches that
  passed the targeted test files failed when all developer test files were
  run. ([2025](https://arxiv.org/abs/2503.15223))

Catches: regressions. Misses: everything about the new behaviour.
Cost: compute only. Cheapest signal available.

### Property-based testing

- Hypothesis describes tests that "should pass for all inputs in whatever
  range you describe" and runs 100 generated inputs per test by default.
  [Hypothesis documentation](https://hypothesis.readthedocs.io/en/latest/tutorial/introduction.html)
- Models can write properties, with limits: for 40 Python API methods the
  best model produced a valid, sound property test in 2.4 samples on average,
  but covered only 21% of the properties stated in the documentation.
  [Can Large Language Models Write Good Property-Based Tests?, 2023](https://arxiv.org/abs/2307.04346)
- An agent that proposes properties and runs them across 100 packages
  produced bug reports of which 56% were valid on manual review; among the
  21 top-ranked, 86% were valid. Cost was not found in the abstract.
  [Agentic Property-Based Testing, 2025](https://arxiv.org/abs/2510.09907)

Inference: two uses fit this loop. (a) Differential properties, "candidates A
and B return the same output for generated inputs", need no expected value and
automate the matrix over a much larger input space; they apply only where
candidates expose the same callable surface. (b) Invariant properties written
by the attack agent (round-trips, idempotence, "never raises on valid input")
are less exposed to wrong expected values than example tests, but about 44% of
agent-reported violations were not real bugs, so they inform the reviewer and
should not reject a candidate automatically.

Catches: edge-case and input-space defects; divergences the example tests
missed. Misses: properties nobody stated; a wrong intent.
Cost (inference): one model call to write properties, then compute.

### Mutation testing

- Mutant detection correlates with real-fault detection independently of
  coverage; 73% of 357 real faults were coupled to common mutants, 17% were
  not coupled to any mutant (mostly wrong or missing algorithms) and 10%
  would need new operators.
  [Are Mutants a Valid Substitute for Real Faults in Software Testing?, FSE 2014](https://cs.ubc.ca/~rtholmes/papers/fse_2014_just.pdf)
- "Traditional mutation analysis is computationally prohibitive". The
  industrial answer is to mutate only changed, covered, non-trivial lines and
  show a few surviving mutants at review time.
  [State of Mutation Testing at Google, 2018](https://research.google/pubs/state-of-mutation-testing-at-google/)
- Developers shown surviving mutants write more and better tests (abstract
  only read).
  [Does mutation testing improve testing practices?, ICSE 2021](https://arxiv.org/abs/2103.07189)
- Model-written mutants and tests are practical at scale: 9,095 mutants and
  571 tests across 10,795 classes, 73% of tests accepted by engineers.
  [Mutation-Guided LLM-based Test Generation at Meta, 2025](https://arxiv.org/abs/2501.12862)

Mutation testing judges tests, not candidates. Inference: its use here is to
weight the matrix, so that a candidate whose tests kill no mutants of its own
diff contributes little evidence, and so that "all candidates pass all tests"
is not read as strong when the tests are weak. It says nothing about whether
the behaviour is the intended one, and by the 2014 result it under-represents
algorithmic and omission faults, which are the likely faults of a feature
implemented from a short intent.

Cost (inference): compute-heavy; one test run per mutant per candidate. Only
affordable if restricted to changed lines and a small mutant sample.

### Metamorphic testing

Metamorphic relations are "necessary properties of the target function" over
multiple related inputs and outputs, used for both test generation and result
checking.
[Metamorphic Testing: A Review of Challenges and Opportunities, ACM CSUR 2018](https://i.cs.hku.hk/~tse/Papers/2010s/hlmtCSUR.html)
(abstract only read). The oracle survey treats them alongside pseudo-oracles
as derived oracles.

Evidence is thin for this loop. A search found studies of models proposing
metamorphic relations (for example
[arXiv 2503.22141](https://arxiv.org/abs/2503.22141)), none of which was read,
and no study of metamorphic relations used to select among candidate changes.
Inference: in practice a metamorphic relation is one kind of property test, so
it comes at no separate cost if property-based testing is adopted, and it fits
only intents with an input-output function at their core.

### LLM as judge (the blind reviewer)

- Judges track human scores better than text-similarity metrics on code
  tasks (Pearson 68.51 on code generation against 64.92 for the best
  conventional metric), which is a modest margin (abstract only read).
  [Can LLMs Replace Human Evaluators?, ISSTA 2025](https://arxiv.org/abs/2502.06193)
- A committee that scores each patch from the issue and the diff resolved
  35.7% where 54.3% of issues had a correct candidate among ten and the
  average candidate resolved 26.6%.
  [DEI, 2024](https://arxiv.org/html/2408.07060)
- Prompt-only judges improve and then decline as candidates are added; with
  three candidates a prompt-only judge was 6.9 to 14.5 points below oracle
  selection, against 3.6 to 7.7 points for a selector that also runs code.
  [Trae Agent, 2025](https://arxiv.org/html/2507.23370)
- Order effects: changing the order of the responses shown to a judge changed
  the outcome enough that one system "won" 66 of 80 comparisons. The fix
  studied is to average over orderings.
  [Large Language Models are not Fair Evaluators, 2023](https://arxiv.org/abs/2305.17926)
- Self-preference: evaluators score their own outputs higher, and the effect
  grows with the ability to recognise them.
  [LLM Evaluators Recognize and Favor Their Own Generations, 2024](https://arxiv.org/abs/2404.13076)
- A judge "marks incorrect answers as correct if both models agree on the
  incorrect answer", inflating weaker models from the same provider.
  [Correlated Errors in Large Language Models, 2025](https://arxiv.org/html/2506.07962)

Catches: readability, scope creep, obvious mismatches with the intent,
unsafe changes. Misses: behaviour it predicts wrongly; errors it shares with
the candidates' authors.
Cost: one to a few model calls per candidate; a multiple of that if orderings
are averaged.

Inference: blindness to authorship does not remove the correlated-error
problem when reviewer and candidates run on the same model. The measures with
evidence are execution grounding and order averaging. A reviewer on a
different model family is suggested by the correlation results but its benefit
for code selection was not measured in anything read here, and the same paper
finds strong models correlated across providers.

### Asking the developer about behaviour (intent check)

TiCoder shows the developer generated tests and asks whether each matches
their intent, then prunes candidates. In a 15-person study participants were
"significantly more likely to correctly evaluate AI-generated code" with lower
reported cognitive load; with simulated answers, pass@1 improved by 45.97
points on average within 5 interactions.
[LLM-Based Test-Driven Interactive Code Generation, IEEE TSE 2024](https://arxiv.org/abs/2404.10100)
(abstract only read; the benchmark size and task type were not checked).

Inference: this is the only mechanism surveyed that addresses errors in the
intent, and the matrix already produces its input: each test on which
candidates disagree is a question about intended behaviour. It moves cost to
developer attention.

## What contradicts "matrix plus attack agent can stand in for held-out tests"

1. Shared errors are the normal case, not the exception. Independent human
   teams fail together more than chance predicts, language models agree on
   60% of joint errors, and agreement-based ranking fails mainly when the
   model misunderstands the task. The matrix reports nothing when all
   candidates share a misreading. That is the failure seen in the feature
   replay, with the difference that held-out tests at least made the failure
   visible.
2. Generated tests are weak oracles. Between a fifth and a third of patches
   that pass model-written tests fail hidden tests; fewer than half of
   generated reproduction tests that reproduce an issue are consistent with
   the reference fix; the strictest test-based filter reached 47.8%
   precision. Attack-test failures therefore need adjudication before they
   count against a candidate.
3. Every published selector leaves a gap to oracle selection (about half the
   random-to-oracle gap recovered in CodeMonkeys; 3.6 to 7.7 points at three
   candidates in Trae Agent; 18.6 points for a review committee over ten
   candidates), and more candidates make selection harder.

Two points cut the other way. Held-out tests are themselves a weak standard:
7.8% of test-passing patches fail the wider developer suite, 29.6% diverge
behaviourally from the reference, and augmented tests exposed 345 wrongly
accepted patches
([UTBoost, ACL 2025](https://arxiv.org/abs/2506.09289), abstract only read).
And differential tests between patches found divergences that developer tests
missed, which is direct support for the matrix as a detector of differences.

## Suggested use in this loop (inference)

| Order | Mechanism | Role | Cost | Trust |
|---|---|---|---|---|
| 1 | Full existing test suite | Hard gate | Compute | High for regressions |
| 2 | Cross-test matrix, extended with differential property tests where candidates share an interface | Cluster candidates by behaviour; list the distinguishing tests | Compute, one model call for properties | High for "these differ", none for "these are right" |
| 3 | Attack agent | Add distinguishing tests and invariant properties | Largest model spend | Evidence, not verdict; an all-candidates failure needs adjudication |
| 4 | Reviewer given the execution results, orderings averaged | Rank clusters, explain differences | A few model calls | Better than reading code alone; shares errors with candidates |
| 5 | Developer answers the few distinguishing questions | Resolve intent ambiguity before approval | Developer attention | The only check on the intent |
| Optional | Mutation sample on changed lines | Discount weak tests in the matrix | Compute-heavy | Measures test strength only |

On budget: the two systems that report cost differ by an order of magnitude.
Agentless reports $0.70 per issue, of which $0.25 is validation. CodeMonkeys
reports about $2,300 for its SWE-bench Verified run with ten candidates per
issue. Both are at their own models and prices, so neither is a forecast for
this loop.

## What this research could not settle

- Whether any of this transfers from benchmark issues with a reference patch
  to a live intent. Every number above was measured against held-out tests or
  a reference implementation.
- How often an attack agent's tests are wrong in a multi-candidate setting,
  and whether it should see candidate code. No source measures this.
- Whether a reviewer from a different model family selects better. Suggested,
  not measured for code.
- Whether mutation-weighting the matrix improves selection. No source found.
- Metamorphic testing for candidate selection: no study found; the studies of
  model-proposed relations were not read.
- Detection of intent errors without the developer. Nothing found.
- The size of the feature-versus-bug-fix difference. The repository-level
  benchmarks cited are built from issue resolution, where a reproduction test
  is a natural oracle. Inference: a feature has no failing behaviour to
  reproduce, so the reproduction-test results (the largest gains above) may
  not carry over. No source read here measures this.

## Not verified

- The 1986 multiversion paper: abstract only. The significance level and the
  counts of coincident failures often quoted for it were seen only in a
  secondary summary and are not repeated here.
- The original differential testing paper (Digital Technical Journal, 1998)
  was not located; the technique is cited through the oracle survey's
  treatment of pseudo-oracles.
- The published analysis behind SWE-bench Verified (share of benchmark tasks
  with underspecified issues or unfair tests) could not be fetched and is not
  used.
- Figures marked "abstract only read" were not checked against the full text.
- Most numbers were extracted from the fetched pages by an automated reader
  and not re-checked by hand against the PDFs; treat a figure as a pointer to
  the cited section, not as a transcription.
