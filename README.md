# Donut

**the perpetual session**

A session that doesn't end when the window fills.

Donut is a compaction backend for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness). It keeps a long-running agent inside its context window without summarizing the whole conversation into one lossy message, and without restarting the harness.

The model is stateless: every request carries the whole conversation. Donut changes what the next request carries, not what happened. A session is an append-only log; the model sees a surface folded over it. Donut replaces only the oldest part of the middle band, and only enough to land near 55% of the window.

```
[ system ][ HEAD frozen ][ cards... ][ STATE ][ band ][ TAIL ]
  node 0     <= 20 %      per chunk    one    oldest  newest 25 %
```

- **HEAD** -- the opening and the context injected with it, verbatim. Once the first card exists it never moves again.
- **Cards** -- one frozen index card per compressed chunk: a WHAT line and a KEYS line of exact literals with their sequence numbers. Written once, never rewritten.
- **STATE** -- one working-memory document: task, instructions in force, decisions, current state, verified results, rejected approaches, unfinished work.
- **TAIL** -- the newest 25% of the window, raw.

The raw log is never edited. Every card names the seq range it replaced and the recall call that reads it back. Nothing compressed is lost; it is indexed.

## Status

Experimental. The design is specified and measured in a scratch harness against DeepSeek Harness Desktop 0.2.1-rc.3. **The plugin code is not published yet.** This repository currently holds the design record; the implementation follows when the end-to-end tests are done.

- Design record: [`docs/SPEC-v1.md`](docs/SPEC-v1.md)
- The spec's two open placeholders (`RUN_FULL5_RESULT`, `REVIEW_STATUS`) mark the work still in flight.
- The code will land as `dsh-compaction-donut`, MIT.

## Why

Every harness eventually meets the window. The usual answers are to summarize everything into one message (and lose the nuance) or to stop. Donut automates the manual loop power users already run by hand: write a rehydration prompt, cut the oldest third, restart, rehydrate. It does it live, in one session, with nothing deleted.

## Roadmap

- [ ] Publish the plugin (`dsh-compaction-donut`) after the full-scale run and review are recorded
- [ ] Three-pass recall test: a question whose answer lives only in a card
- [ ] Card rollup for sessions past ~5M tokens
- [ ] `/donut` status command
- [ ] A static rule telling the model to recall originals before acting on a vague card

## License

MIT. See [LICENSE](LICENSE).
