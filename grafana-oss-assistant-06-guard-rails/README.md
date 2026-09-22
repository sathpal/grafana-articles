---
title: "Guard rails: keeping an AI assistant safe, cheap and honest in Grafana Cloud (Part 6 of 6)"
published: false
description: "What the open-source assistant got wrong (a fictional 4.75 s p95, invented metric names, 81 tools), the controls that contain it, and the five things that made it work."
tags: grafana, observability, ai, opensource
series: An Open-Source AI Assistant for Grafana Cloud
cover_image: img/cover.png
---

**TL;DR** The assistant in this series solved three incidents. It also confidently reported a p95 of 4.75 seconds that did not exist, would have invented metric names without a rule against it, and had 81 tools when it needed 15. This part is the list of what went wrong, the controls that contain it, and the five things that made the whole thing work. Read it before you point an agent at production.

## What it gets wrong without help

**Metric names.** Left alone, a model writes `kafka_consumer_lag` or `jvm_heap_used` from memory. Neither exists. The house rule "start with `list_prometheus_metric_names`" is not decoration; the OTLP gateway's naming (dots to underscores, `_seconds` and `_milliseconds` suffixes, `_total` on counters) is exactly the detail it will not guess.

**Histograms with the wrong buckets.** This one bit me, not the model. My first Java timer exported a histogram with only a `+Inf` bucket, and my Python and Go histograms used millisecond-shaped default buckets for values in seconds. `histogram_quantile` returned `NaN` in one case and a flat, fictional 4.75 s in the other. The assistant reported both as fact, because they *were* what Grafana said. Fix the instrumentation (explicit bucket boundaries, or record in ms with the default buckets). Do not ask the model to reason around a broken p95.

**Range windows.** `rate(x[5m])` on a counter that started two minutes ago is empty. Tell it to widen before it concludes "no traffic".

**Time zones.** The MCP server treats timestamps without an offset as UTC. Ask for relative ranges (`now-15m`) and let Grafana render local time.

**Log volume.** `query_loki_logs` returns 100 lines by default; "find the error" can pull megabytes into the model's context and your bill. Prefer `query_loki_stats`, `format: compact`, and metric queries (`count_over_time`) before raw lines. Part 5's "32 errors, all on one service" was one metric query, not 32 log lines.

**Alert state.** A paused rule looks healthy. If the question is "why did nobody get paged", make the assistant check `alerting_manage_rules` for `is_paused` and read the alert history, not just the current state.

## Keeping it safe

| Control | How | Why |
| --- | --- | --- |
| Read-only by default | `mcp-grafana --disable-write`, and a Viewer service account | Parts 3 and 4 needed no writes at all |
| Scoped writes | An Editor token only for the session that does Part 5; house rule 5 ("confirm the exact object") | Annotations, alert rules and panels are the only things it should create |
| Fewer tools | `--disable-oncall --disable-incident --disable-admin --disable-sift --disable-pyroscope ...` | 81 tools cost context and invite detours; this series used about 15 |
| One server per team, not per laptop | `-t streamable-http`, `--server-auth-token`, behind your usual ingress | The token lives in one place, and the server's log is the audit trail of every Grafana API call |
| No secrets in prompts | Credentials are environment variables of the server process | The model never sees the token, so it can never leak it into a transcript |
| Pin the model and log the run | `GOOSE_MODEL`, recipes in git, `goose run --recipe` | An investigation you cannot replay is an anecdote |
| Budget | goose's `--max-tool-repetitions`, and a per-run token cap at your provider | An assistant that loops on `query_prometheus` is an expensive way to find nothing |

## What made the difference

1. **Change annotations.** The single most valuable signal in all three incidents. Here the batch job and the chaos script wrote them; in production that is your CI/CD, your feature-flag service and your autoscaler. Without them, root cause is inference. With them, it is lookup.
2. **One `service_name` across metrics, logs and traces.** Alloy sets it on container logs, the OpenTelemetry SDKs set it on spans and metrics, and the exporters get it from scrape labels. Every pivot the assistant made relied on it.
3. **Trace context through Kafka.** The Python producer put `traceparent` in the message headers, the OpenTelemetry Java agent read it in the consumer, and the Go service passed it on to its own producer. That is why one trace id shows a 66-second wait and a 165-millisecond publish (Part 3), and why a red Java span sits inside a green trace (Part 5).
4. **Six lines of house rules.** They turned a general agent into one that starts with `user_info`, refuses to assert without a query, and ends with a report in a fixed shape you can paste into an incident channel.
5. **The exporters you would run anyway.** Kafka consumer lag and Redis evictions came from `kafka-exporter` and `redis_exporter`, scraped by Alloy. The assistant did not need anything bespoke, only the metrics an SRE would already have.

## Use cases: where the assistant saves time, and why open source

| # | Case | Human baseline | Assistant | Why it is efficient |
|---|---|---|---|---|
| 1 | Cross-signal root cause (Part 3) | 6 PromQL queries, a LogQL search, copy a trace id, read 28 spans: 20 to 30 min for someone fluent in all three | 9 tool calls, one report with every query quoted, under 2 min | No PromQL, LogQL or TraceQL needed on call; the log-to-trace pivot is automatic |
| 2 | "What changed, in what order?" (Part 4) | Scrub five dashboards, note timestamps by eye, open alert history separately | Range queries at 15 s steps, first-breach times, alert history read from Loki, one timeline table | Ordering events is mechanical and error-prone for humans, trivial for a tool loop |
| 3 | Silent failure (Part 5) | Nobody looks, because nothing is red; found hours later by an editor | Consumed-minus-published arithmetic across two counters, one log line per article, one red span | Finds a class of bug no lag or latency alert can see |
| 4 | Write back the guard rails (Part 5) | Three UI workflows: annotation, alert rule with threshold and summary, dashboard JSON edit | Three tool calls, attributable to the service account, in the same session | The investigation leaves a better Grafana behind, not a transcript |
| 5 | "Why did nobody get paged?" (`assistant/recipes/paged.yaml`) | Open each rule: state, threshold, pause flag, history | Lists paused rules and thresholds tuned for floods that miss six lost articles | One-prompt audit of things that are found only after the incident |
| 6 | Incident update for humans | Written by hand from memory | "Write a Slack-ready update with deep links" from evidence it already holds | Free once the investigation is done; every claim is clickable |
| 7 | Dashboard and metric hygiene (`assistant/recipes/hygiene.yaml`) | Notice a NaN p95 weeks later | Checks every panel query against Mimir, flags histograms without buckets and windows shorter than 4x the scrape interval | A scheduled recipe instead of a person; this series lost an hour to exactly this |
| 8 | Capacity sanity check | A spreadsheet, eventually | 300 articles x 10 variants x 20 KiB = 60 MiB into a 48 MiB tier, in the Part 4 verdict | The model is good at "does this fit", given the numbers |
| 9 | Scheduled health report | Not done, or done badly | `goose run --recipe` from cron or CI | Recipes are files in git: no seat licence, no UI |
| 10 | Regulated or air-gapped estates | Cloud assistants are not allowed | Same recipes against self-hosted Grafana OSS (`make oss-up`, tested below) and, with Ollama, a local model; prompts and telemetry never leave the network | Only possible because both halves are open source |

Case 10 is not hypothetical. The same dashboard, alert rules, recipes and demos ran against a self-hosted Grafana OSS 13.2 (the `grafana/otel-lgtm` all-in-one, fed by the same Alloy) with nothing changed except three datasource UIDs and the credential:

![The Publishing tier dashboard on self-hosted Grafana OSS 13.2, fed by the same Alloy pipeline as Grafana Cloud](img/06-oss-grafana-dashboard.png)

Why open source, in efficiency terms: it runs where the work is (terminal, CI, cron); cost is a dial (a frontier model
for the 02:00 root cause, a small or local model for the daily report); every call is in the MCP server log; the same
server drives the managed Grafana Assistant too, so there is no lock-in in either direction; and a recipe in git is a
runbook that executes itself.

## Where this leaves you

An open-source agent plus the open-source Grafana MCP server gives you an assistant that reads the same Grafana Cloud your team reads, cites the queries it ran, and can be swapped, audited and budgeted like any other tool. It is not magic. It is a fast, tireless colleague who has read the runbook and will show you their work, provided you give them good telemetry and a short list of rules.

Everything here is reproducible:

```bash
git clone https://github.com/sathpal/grafana-assistant-demo && cd grafana-assistant-demo
cp .env.example .env                # Grafana Cloud OTLP creds + GRAFANA_URL + GRAFANA_SA_TOKEN
make pub-up && make pub-seed        # the polyglot publishing tier, shipping to your stack
make pub-grafana                    # dashboard + alert rules
mcp-grafana -t streamable-http -address localhost:8300 &
make pub-chaos S=batch-flood        # or no-ttl / batch-flood-media / db-write-fail
goose run --recipe assistant/recipes/problem1.yaml
```

Scripted walkthrough of this part: `make demo P=6` in the [demo repo](https://github.com/sathpal/grafana-assistant-demo) (narrated, with pauses; `DEMO_AUTO=1` to record).

## Further reading

- [grafana/mcp-grafana](https://github.com/grafana/mcp-grafana): the server, its tool list, transports and flags
- [block/goose](https://github.com/block/goose): the agent, providers, recipes and extensions
- [Model Context Protocol](https://modelcontextprotocol.io): the standard both sides speak
- [Grafana Cloud OTLP endpoint](https://grafana.com/docs/grafana-cloud/send-data/otlp/) and [Grafana Alloy](https://grafana.com/docs/alloy/latest/): how the telemetry got there
- [OpenTelemetry Java agent](https://github.com/open-telemetry/opentelemetry-java-instrumentation), [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/), [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/): the instrumentation on each side of Kafka
- [kafka-exporter](https://github.com/danielqsj/kafka_exporter) and [redis_exporter](https://github.com/oliver006/redis_exporter): the two Prometheus exporters that supplied lag and evictions
- [Grafana Assistant](https://grafana.com/docs/grafana-cloud/machine-learning/assistant/): the managed alternative, which the same MCP server can also drive

Thanks for reading all six. If you build one of these, I would like to hear which tool it reached for first.
