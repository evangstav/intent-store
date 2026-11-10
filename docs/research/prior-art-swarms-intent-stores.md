# Prior art: agent swarms and intent stores

Research for issue #4. Date of search: 2026-11-10.

## Question

What existing systems or papers combine a store of intent or specifications with
multiple agents producing competing candidates and a selection step? For each:
how intent is recorded, how diversity among candidates is obtained (assigned or
emergent), how the winner is selected, and what the human approves. What
contradicts the two working assumptions: diversity must be assigned, and
selection needs more than one mechanism?

## Short answer

- **No system found combines all the parts.** Spec-driven tools keep intent in
  versioned files but produce one implementation. Multi-candidate systems
  generate and select among candidates but take intent from a throwaway prompt
  or a benchmark issue and record no decision. The combination (versioned
  intent, competing candidates, layered selection, human approval, recorded
  decision) was not found in this search. That is absence of evidence from a
  limited search, not proof that none exists.
- **"Diversity must be assigned" is too strong as stated.** Plain temperature
  sampling of one agent yields candidates that differ in *which tasks they
  solve*, and every benchmark system below exploits that. The evidence supports
  a narrower claim: assigned or structured diversity yields more than sampling
  does, and sampling alone gives little diversity of *approach*.
- **"Selection needs more than one mechanism" is mostly supported.** The
  strongest systems layer an execution filter with a vote or a judge, and
  ablations show each layer adds. Single-mechanism selection does ship in
  products (one LLM judge plus the human), and no layered pipeline closes the
  gap to an oracle selector.

## Method and limits

Sources are papers on arXiv, official documentation and engineering posts by
the systems' authors. Each was fetched and read on the search date. Limits:

- Most pages were read through an automated fetch-and-extract step. Quotes are
  short and exact as returned; section numbers and table figures for the arXiv
  HTML papers should be spot-checked before they are relied on in a decision.
  AlphaCode quotes were checked against the extracted PDF text directly.
- All quantitative evidence comes from benchmarks (SWE-bench, LiveCodeBench,
  Codeforces) that score correctness against hidden tests. None measures
  architectural or design diversity, which is what "directions" assign.
- Product behaviour (Cursor, Kiro, Spec Kit, OpenSpec) is taken from vendor
  documentation; no independent evaluation of it was found.
- Not verified: OpenAI Codex cloud's "best of N" feature. Only third-party
  descriptions turned up; the first-party changelog pages read did not contain
  it. It is left out of the table.

## Systems

### A. Spec-driven tools: intent is stored, candidates do not compete

| System | Intent record | Diversity | Selection | Human approves |
|---|---|---|---|---|
| GitHub Spec Kit | Markdown per feature under `specs/[branch-name]/`, plus a constitution file; versioned in branches and merged [1] | None built in. The methodology names "Branching for Exploration": "Generate multiple implementation approaches from the same specification" [1], and the launch post says "You can also ask for multiple plan variations" [2]. No mechanism for it is described | None | Spec, plan, tasks, then "focused changes" in code; "your role isn't just to steer. It's to verify" [2] |
| Kiro specs | Three files per spec (`requirements.md` or `bugfix.md`, `design.md`, `tasks.md`) [3], requirements in EARS notation [4], shown under `.kiro/specs/` and "designed to be version-controlled" [5] | None; parallelism only runs independent tasks concurrently [3] | None | Approval gates between phases; "Quick Spec" skips them [4] |
| OpenSpec | A folder per change under `openspec/changes/` holding proposal, specs, design and tasks; archived on completion, which updates the standing specs [6] | None | None | "you review the plan before any code is written" [6] |
| Copilot Workspace | Task, then an editable specification and plan inside the product session [7] | None described | None | Each step is editable; code runs "once you're satisfied with the plan" [7] |

These tools support the store half of the design: intent as reviewed, versioned
files next to the code. OpenSpec's archive of completed changes is the closest
thing found to a decision record. None records rejected alternatives, because
none produces any.

### B. Multi-candidate systems: candidates compete, intent is a prompt

| System | Intent record | Diversity | Selection | Human approves |
|---|---|---|---|---|
| AlphaCode [8] | Problem statement | **Assigned, randomly**: half the samples in Python and half in C++, randomised problem tags and ratings in the prompt, high temperature | Two stages: filter on the statement's example tests (removes about 99% of samples), then cluster survivors by behaviour on model-generated inputs and submit one per cluster, largest first | Nothing (10 automatic submissions) |
| CodeT [9] | Prompt | Emergent (sampling) | Model-generated tests run against all samples; "dual execution agreement" scores agreement with tests and with other samples | Nothing |
| MBR-EXEC [10] | Prompt | Emergent | Execution-based minimum Bayes risk: pick the program whose outputs agree with the most others | Nothing |
| Self-consistency [11], Universal self-consistency [12] | Prompt | Emergent | Majority answer [11]; or one LLM call picks "the most consistent answer among multiple candidates" [12] | Nothing |
| Coder-Reviewer reranking [13] | Prompt | Emergent | A reviewer model scores how well the instruction can be recovered from each program | Nothing |
| PlanSearch [14] | Problem statement | **Assigned, generated**: the model writes observations, combines them into distinct natural-language plans, and each plan becomes one program | Filter on public tests | Nothing |
| S* [15] | Problem statement | Emergent, plus iterative refinement | Generates "distinguishing inputs for pairwise comparison" and uses execution results | Nothing |
| Agentless [16] | Benchmark issue | Emergent: temperature 0.8, four localisation sets, 40 patches | Three layers: regression tests, generated reproduction test, then majority vote over normalised patches | Nothing |
| CodeMonkeys [17] | Benchmark issue | Emergent: 10 independent trajectories at positive temperature, each writing its own test script | Each edit is run against each generated test; the top three by votes go to a multi-turn selector that can write new distinguishing tests | Nothing |
| DEI [18] | Benchmark issue | **Assigned by agent**: a committee of different agent frameworks | One mechanism: an LLM review that explains and scores each patch 1-10, averaged over 10 votes | Nothing |
| Trae Agent [19] | Benchmark issue | Emergent (high temperature); optionally assigned by cycling three models | Prune (deduplicate, regression tests), then a selector agent that reads code and runs tests, repeated N times and majority-voted | Nothing |
| Anthropic "high compute" scaffold [20] | Benchmark issue | Emergent: "multiple parallel attempts" | Discard patches that "break the visible regression tests", then "rank the remaining attempts with a scoring model" | Nothing |
| Augment SWE-bench agent [21] | Benchmark issue | Emergent | One mechanism: o1 given candidate diffs and the problem statement | Nothing |
| Cursor [22][23][24] | Chat prompt | **Assigned by model**: `/best-of-n` "runs the same task across multiple models at once", each in its own worktree | One LLM judge: "automatically evaluate all runs and give a recommendation for the best solution" [23] | The human picks: "Pick the strongest result before applying anything" [24] |
| FunSearch [25] | An evaluation procedure plus a seed program | Evolutionary pool with "a strategy to improve the diversity of ideas" | Automatic execution and scoring; best programs return to the pool | Nothing; the human reads the resulting program |

Among these, Cursor is the only one found with a human approval step, and
FunSearch is the only one whose "intent" is itself executable.

## Against the working assumptions

### Assumption 1: diversity must be assigned, it does not emerge

**Contradicting evidence.** Sampling one agent repeatedly does produce useful
variation in outcomes:

- Ten runs of the same agent raise the share of issues solved by at least one
  run from 14.0 to 35.3 (Moatless), 22.3 to 38.0 (Aider) and 20.3 to 34.7
  (Agentless) [18].
- CodeMonkeys' ten undirected trajectories reach 69.8% coverage against 45.8%
  for a random single pick [17].
- Coverage "scales with the number of samples" over four orders of magnitude,
  often log-linearly [26].

So for correctness on a fixed task, diversity does emerge with nothing assigned.
The undirected "solo" agent in the loop is this baseline, and the literature
says several solo agents would not be redundant.

**Supporting evidence.** Assigned or structured diversity yields more:

- Ten *different* agents raise the same union measure from 26.7 to 54.3, a
  larger gain than any single agent's ten runs: "Different agents resolve more
  distinct issues than different runs of a single agent" [18].
- PlanSearch attributes weak search to a "lack of diverse LLM outputs" and
  reports that repeated sampling "returns highly similar programs". Assigning
  each sample a distinct plan lifts pass@200 on LiveCodeBench from 55.6
  (repeated sampling) to 77.0 with the same model, and "diversity as measured
  in idea space is highly predictive of search performance" [14]. This is the
  closest prior art to forking an intent into directions; its directions are
  generated by the model, not chosen by a person.
- AlphaCode's ablation: the best results come from random tags per sample,
  "showing that the extra diversity of samples is important to increasing the
  solve rate" [8].
- Cursor reports that "having multiple models attempt the same problem and
  picking the best result significantly improves the final output" [22]. No
  numbers are given.
- An older, human result points the same way: 27 program versions written
  independently from one specification failed together far more often than
  independence predicts [27]. Independent work from the same statement is not
  independent in its mistakes.

**Mixed.** Trae Agent's three-model mixture scored 65.67%, below its best
single model at 66.40% [19]. Assigning diversity by model did not help there.

**Reading.** The prototype finding and the literature measure different things.
The literature shows emergent diversity in *whether* a candidate is correct; the
assumption concerns diversity of *approach*. A defensible restatement:
"approach-level diversity must be assigned; outcome-level diversity also comes
from resampling." No source found tests assigned architectural directions on
repository-scale tasks. That evidence is thin.

### Assumption 2: selection needs more than one mechanism

**Supporting evidence.**

- Agentless ablation on SWE-bench Lite: majority voting alone 77 fixes, plus
  regression tests 81, plus reproduction tests 96 [16].
- CodeMonkeys compared four selectors; test voting followed by a multi-turn
  selector "performs best" [17].
- AlphaCode: "Filtering and clustering clearly enable scaling, as otherwise the
  solve rate remains flat" [8].
- Trae Agent reports that LLM-only selectors (the Augment and DEI style) rise
  and then fall as the number of candidates grows, because "subtle behavioral
  differences between them become increasingly difficult to distinguish using
  LLMs alone" [19].
- Without an automatic verifier, majority voting and reward models "plateau
  beyond several hundred samples" [26].
- Voting assumes independent errors, which [27] found not to hold for human
  versions. Applying that to same-model candidates is an inference made here,
  not a claim in the source.

**Contradicting evidence.**

- Single-mechanism selectors ship and beat no selection: DEI's review committee
  lifts the average candidate from 26.6 to 35.7 [18]; Augment used one o1 call
  [21]; Cursor uses one judge and then the human [23][24].
- Universal self-consistency reports matching execution-based voting on code
  with a single LLM call and no execution [12].

**What no pipeline solves.** Every system leaves a large gap to an oracle
selector: CodeMonkeys 57.4% achieved against 69.8% coverage, closing
"approximately half of the gap between random and oracle selection" [17]; DEI
35.7 against 54.3 [18]; AlphaCode "a large gap" to the upper bound [8]. More
mechanisms help; none makes the human check redundant.

### Prior art for the specific gates

- **Cross-test matrix.** Direct precedent in CodeT [9] and CodeMonkeys [17]:
  every candidate runs against tests written for other candidates.
  CodeMonkeys notes that its trajectories "have no awareness of other state
  machines", which matches blind sandboxes.
- **Regression filter before judging.** Agentless [16], Trae Agent [19] and the
  Anthropic scaffold [20] all filter on existing tests first. Trae reports the
  cost: 3.69% of correct patches wrongly discarded, and a 33.04% false-positive
  rate [19].
- **Attack agent.** No dedicated adversarial agent was found. The nearest are
  selectors that write new inputs to tell candidates apart: S* [15] and the
  CodeMonkeys selection state machine [17].
- **Blind reviewer.** LLM review as a selector is common [18][21][23]. No source
  found describes blinding the reviewer to candidate origin.

### Cost

Augment judged ensembling "not realistic for real-world usage at current model
serving costs" [21]. CodeMonkeys reports about $2,300 for SWE-bench Verified
[17]; over that benchmark's 500 issues this is about $4.60 per issue (arithmetic
done here), close to the v0 target of about $5 per intent. Model prices have
changed since both were written.

## Sources

1. GitHub Spec Kit, `spec-driven.md`: https://github.com/github/spec-kit/blob/main/spec-driven.md
2. GitHub Blog, "Spec-driven development with AI": https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
3. Kiro docs, Specs: https://kiro.dev/docs/specs/
4. Kiro docs, Feature specs: https://kiro.dev/docs/specs/feature-specs/
5. Kiro docs, Specs best practices: https://kiro.dev/docs/specs/best-practices/
6. OpenSpec README: https://github.com/Fission-AI/OpenSpec
7. GitHub Blog, "GitHub Copilot Workspace": https://github.blog/news-insights/product-news/github-copilot-workspace/
8. "Competition-Level Code Generation with AlphaCode" (sections 4.4-4.6, 5.3.5, appendix on tag conditioning): https://arxiv.org/abs/2203.07814
9. "CodeT: Code Generation with Generated Tests": https://arxiv.org/abs/2207.10397
10. "Natural Language to Code Translation with Execution": https://arxiv.org/abs/2204.11454
11. "Self-Consistency Improves Chain of Thought Reasoning in Language Models": https://arxiv.org/abs/2203.11171
12. "Universal Self-Consistency for Large Language Model Generation": https://arxiv.org/abs/2311.17311
13. "Coder Reviewer Reranking for Code Generation": https://arxiv.org/abs/2211.16490
14. "Planning In Natural Language Improves LLM Search For Code Generation": https://arxiv.org/abs/2409.03733
15. "S*: Test Time Scaling for Code Generation": https://arxiv.org/abs/2502.14382
16. "Agentless: Demystifying LLM-based Software Engineering Agents": https://arxiv.org/abs/2407.01489
17. "CodeMonkeys: Scaling Test-Time Compute for Software Engineering": https://arxiv.org/abs/2501.14723
18. "Diversity Empowers Intelligence: Integrating Expertise of Software Engineering Agents": https://arxiv.org/abs/2408.07060
19. "Trae Agent: An LLM-based Agent for Software Engineering with Test-time Scaling": https://arxiv.org/abs/2507.23370
20. Anthropic, "Claude 3.7 Sonnet and Claude Code" (appendix, SWE-bench Verified): https://www.anthropic.com/news/claude-3-7-sonnet
21. Augment Code, "#1 open-source agent on SWE-bench Verified by combining Claude 3.7 and O1": https://www.augmentcode.com/blog/1-open-source-agent-on-swe-bench-verified-by-combining-claude-3-7-and-o1
22. Cursor, "Introducing Cursor 2.0": https://cursor.com/blog/2-0
23. Cursor changelog 2.2: https://cursor.com/changelog/2-2
24. Cursor docs, Worktrees: https://cursor.com/docs/configuration/worktrees
25. Google DeepMind, "FunSearch: Making new discoveries in mathematical sciences using Large Language Models": https://deepmind.google/discover/blog/funsearch-making-new-discoveries-in-mathematical-sciences-using-large-language-models/
26. "Large Language Monkeys: Scaling Inference Compute with Repeated Sampling": https://arxiv.org/abs/2407.21787
27. Knight and Leveson, "An Experimental Evaluation of the Assumption of Independence in Multiversion Programming", IEEE TSE 12(1), 1986: https://doi.org/10.1109/TSE.1986.6312924 (abstract only was read, via a secondary copy; the full text was not checked)
