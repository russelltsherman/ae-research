# ae-research

A research workspace — topic lists and the cited Markdown reports generated from them by the
`/ae-research` command.

The tooling that produces these reports (the `/ae-research` and `/ae-code-review` commands and their
retry-hardened workflows) used to live here as the `ae-rnd` plugin. It has moved to the **`rnd`
plugin** in [`ae-skills`](https://github.com/russelltsherman/ae-skills). This repo now holds only the
inputs (topics) and outputs (reports).

## Layout

```text
topics.md            # the topic list batch runs read by default
topics.example.md    # annotated template — copy to topics.md and edit
research/            # generated reports, one per topic, with an index (research/README.md)
```

## Producing reports

Install the `rnd` plugin, then run `/ae-research` from this directory:

```text
/plugin marketplace add russelltsherman/ae-skills
/plugin install rnd@ae-skills
```

```text
/ae-research test driven development   # single topic
/ae-research                           # batch over topics.md
/ae-research @my-list.md               # batch over a custom file (note the leading @)
```

- **Single** — a plain topic string researches just that topic.
- **Batch** — an `@`-prefixed file path (or no argument, which defaults to `topics.md`) researches
  every topic in the file. Create one by copying [`topics.example.md`](topics.example.md).

Each topic is its own research run with a fresh budget. Reports are written to `research/<slug>.md`,
with an index at [`research/README.md`](research/README.md). Batch re-running is safe: topics whose
report already exists are skipped, so an interrupted batch resumes where it left off (delete a report
to force a re-run). A single-topic run always re-runs and overwrites its report.
