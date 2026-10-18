# lookahead-detector

A procedure for finding and proving lookahead bias in a trading backtest: a simulated
decision that uses information which would only exist later than the time the simulation
says the decision happened. Backtests with this bug look profitable because they trade on
the future.

It is a plain-markdown playbook (`SKILL.md`). It needs no tool, language or model in
particular: a human or any LLM agent can follow it with whatever file-reading and shell
tools they have.

## What it does

1. **Static timing audit.** A 12-point checklist: open vs close bar stamps, fill price vs
   decision time, intrabar ordering, higher-timeframe merges, rolling windows, negative
   shifts, resampling labels, signal-then-execute on the same bar, reference data listed
   later, state carried across the loop, same-bar exits, partially filled caches.
2. **Shift test and truncation test.** Delay the entry by one bar and see whether results
   collapse; cut the data off part-way and check that past trades don't change.
3. **Replay test.** Rebuild the data up to a single trade's decision time and confirm the
   signal and fill are identical.
4. **Live-vs-backtest reconciliation.** Line real fills up against backtest trades. A
   constant offset equal to the bar size is the signature of a timestamp-convention bug.
5. **Report and fix.** A numbers-only report format, and a test-first fix recipe.

`SKILL.md` also contains a short case study from one of our own test strategies, where a
profit factor of 1.60 turned into 0.915 once the lookahead was removed.

## Using it

`SKILL.md` is a single self-contained file, so it works with any model that can read text
and, ideally, read files and run commands in your repo.

- **Agents with a skills folder** (any tool that loads a folder containing a `SKILL.md`
  with YAML frontmatter): copy this folder into that tool's skills directory and invoke
  it by name, `lookahead-detector`.
- **Agents with a rules / instructions file** (for example `AGENTS.md` or an equivalent):
  reference or paste `SKILL.md` into it.
- **Chat assistants and local models:** paste `SKILL.md` into the conversation, then give
  it your backtest loop, signal code and data loader.
- **Humans:** read it top to bottom and apply the checklist to your own code.

A prompt that works anywhere:

> Follow the steps in SKILL.md to audit my backtest for lookahead bias. The backtest loop
> is in `<path>`; the signal code is in `<path>`; the data loader is in `<path>`. Report
> in the format given in step 5.

Steps 2 to 4 (shift test, replay test, live reconciliation) need to run code or read real
fills; an assistant without shell access can still do the static audit and tell you what
to run.

## Scope

It looks for bias in timing and information. It does not prove a strategy will make money:
overfitting, costs and regime change are separate checks.

## License

MIT. See `LICENSE`.
