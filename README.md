# One person, three AI providers, about ten agents: how I build and check software

*By Steve Jensen (TCalvion Tech OÜ). This article is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).*

I'm a self-taught builder in Estonia. I run a small software operation where AI agents do the implementation under my direction. I design the systems, decide what gets built, define what "correct" means, and decide how the work is checked and when it is actually done. Most of that turns out to be **evaluation**.

## The setup

- **Three subscription providers, used for different jobs.**
  - Anthropic: Claude Opus, as a builder and as the lead agent that operates the workstation.
  - OpenAI: GPT models in Codex, as builders and auditors.
  - Google: Gemini models in Antigravity, as builders on smaller projects.
- **Local models** on two machines (a laptop with an RTX 5070 Ti, and an old ThinkPad running 24/7) through llama.cpp. They handle cheap overnight work and Estonian-language jobs. I choose them with my own [Estonian-language benchmark](https://github.com/TCalvion/estonian-llm-eval).
- **Automation for everything that does not need judgement** (see "What runs without me" below).
- **A quota gate.** A small meter reads the 5-hour and weekly usage of all three providers, from local session logs, from the official CLI's usage screen, and from a local language server, without touching account credentials such as OAuth tokens or login files. Automated workers stop at 50% and the lead agent at 70%, so the subscriptions stay usable for me. A stale reading counts as "unknown", never as "fine".
- **One HQ repository**: a decision log, a diary per project, one report template, and a shared rules file every agent reads first.

## What runs without me

Most of the work that does not need judgement never reaches an LLM agent. It goes to automation:

- **NightOwl, the overnight runner.** A scheduled job starts every night on the workstation and runs a plan file: model benchmarks, image and video generation, scripted checks. It has a lock, retries, a GPU guard (a language model and an image model never fight over the same 12 GB of VRAM), a stop file, and a morning report. Any job that would spend a subscription first asks the quota gate. If the answer is "over the limit" or "unknown", the job is skipped and the report says why.
- **A daytime twin** runs the same kind of jobs on demand, and a second track runs them on the ThinkPad, which is on 24/7.
- **Local agents that search.** A script collects job and bounty listings, a prefilter removes what clearly does not fit, and a small local model scores the rest against my criteria. Draft notes are never sent automatically. In one overnight pilot, the script collected 6,759 unique listings. The prefilter left seven new candidates for the local model to score, and six made the shortlist for human review.
- **A machine diary.** A daily script records which jobs ran and where to find their results. The lead agent reads it before the nightly hand-over.

The rule behind it: if a task can be written as a script, it becomes a script. Agents get the work that needs a decision.

## The rules that make it work

1. **Builder and auditor always come from different model families.** If Claude builds, GPT or Gemini audits, and the other way round. Reviewers from the same family tend to share the same blind spots.
2. **Every task ends with a report** in a fixed template: what was done, deviations from the task, what the agent ran into, a better way it noticed, open questions, and how to verify. Every claim is marked *verified* or *assumed*.
3. **One task, one fresh session.** Memory lives in files, not in long chats. A fix round stays with the same builder. The auditor starts fresh and sees only the repository and the report, never the builder's conversation.
4. **The audit is told to try to break the work, not to confirm it.** The standard instruction: *"Review the whole diff and check which workflows the change touches. Reproduce the original bug and choose your own extra probes that could disprove the fix. Also report findings the builder's report did not cover."*
5. **Freeze before you measure.** Test suites and answer keys are reviewed, frozen and hashed before the first run. A wrong key becomes a new version with a written reason, never a silent fix.
6. **Hard limits stay with the human.** Agents commit only to work branches. Pushing, merging to main, keys, real payments and deletions are mine.

## Three real examples

**Green tests, wrong behaviour (Nov 2026).** A data-cleaning product had a critical bug: files in Baltic and Cyrillic encodings were silently misread and then marked "clean". The builder (Claude) submitted a fix. **All 1544 tests passed.** The auditor (GPT) ran its own probes and returned FAIL anyway:
- single-letter fields still slipped through the same bug;
- one broken UTF-16 file would have been priced at $19 although the output was still corrupt;
- some perfectly readable files jumped from $6 to $19, because the new warning changed the pricing path.

Two of the three findings were about charging customers. Passing tests is evidence, not proof.

**A cross-family audit catches a security gap (17 Nov 2026).** A Gemini agent built the quota meter above in a few short rounds (the early ones took about seven minutes each), with all tests passing. Reading the third provider's quota needs a local token, which I allowed under one condition: *memory only, and only towards 127.0.0.1*. The Claude auditor confirmed that the token never reached a file, a log or a command line. It then showed, with two fake local servers, that an HTTP redirect would still carry the token to another host, so the "localhost only" promise depended on the other side behaving well. It also found that a change in the CLI's usage screen could produce a wrong number instead of an honest "unknown". A wrong number is worse than none, because the gate would let through work it should stop. Verdict: pass with notes. Both issues went back to the builder as a fix task the same morning.

**The person at the screen finds what two agents missed (18 Nov 2026).** The next release of the same meter added a language switch, so anyone can add their own language with one translation file. The builder's tests passed. A fresh Claude auditor confirmed that the overnight runner would get exactly the same gate decisions as before (30 of 30 cases), and found one real security problem: translation files could inject HTML into the page, which matters because "add your own language" is an open invitation to outside contributors. Then I opened the page myself and switched it to English. The labels changed, but the notes under each card stayed in Estonian. Neither agent had looked at the page the way a user does. The tests checked that every translation key existed, not what a person actually sees. Both findings went back to the builder as the next task, and the release needed a check of the rendered notes after a language switch. Some checks still need a person looking at the screen in a second language.

## What evaluation means to me in practice

- Writing a probe the builder did not think of is worth more than re-running the builder's tests.
- A plausible, fluent wrong answer is the dangerous one. My Estonian editing test exists because "is considering approving" turning into "approved" reads perfectly and changes the decision.
- "Unknown" is a valid result, and a measurement that cannot be trusted should say so instead of guessing.
- Every failure gets read by hand before it counts against a model, because sometimes the test is what's wrong.

## Work you can look at

- [CSV Doctor case study](https://github.com/TCalvion/CSV-Doctor-Case-Study): deterministic CSV repair, and what independent audits changed.
- [TC Kratt case study](https://github.com/TCalvion/TC-Kratt-Case-Study): a local harness for AI-assisted repository work with permissions, durable records and independent review.
- [Estonian LLM Eval](https://github.com/TCalvion/estonian-llm-eval): my Estonian benchmark, with sample items, deterministic validators and results. Open source.

Source code for the two case studies stays private. More will be added here as it is published.

## About me

Native Estonian, fluent Finnish, working English. Before this I spent years in jobs where details mattered and mistakes were expensive: border guard, ship security, emissions measurement with written technical reports in Finland, and eleven years as a taxi driver in Helsinki. I now work daily on rubrics, adversarial probes, reading model output critically, and documenting results so that someone else can check my work.
