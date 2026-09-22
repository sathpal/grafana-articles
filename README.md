# Grafana articles

Long-form, hands-on write-ups about Grafana Cloud, OpenTelemetry and the open-source tooling around them.
Each folder holds one article: `README.md` is the dev.to source with front matter, `medium.md` is the
Medium-ready copy generated from it, `linkedin-post.md` is the announcement, and `img/` holds the screenshots.

## Series: An Open-Source AI Assistant for Grafana Cloud

Six parts. An open-source agent ([goose](https://github.com/block/goose)) talks to Grafana Cloud through the
open-source [Grafana MCP server](https://github.com/grafana/mcp-grafana), and solves three incidents on a
polyglot pipeline (Python, Kafka, Java, Go, Redis, a batch job). Companion repo with the stack, the chaos
scenarios, the recipes and a scripted demo per part: [grafana-assistant-demo](https://github.com/sathpal/grafana-assistant-demo).

| part | article | demo |
|---|---|---|
| 1 | [I gave Grafana Cloud an open-source AI assistant. Here's why](grafana-oss-assistant-01-why-an-oss-assistant/README.md) | `make demo P=1` |
| 2 | [Setup in 15 minutes: mcp-grafana + goose + Grafana Cloud](grafana-oss-assistant-02-setup-in-15-minutes/README.md) | `make demo P=2` |
| 3 | ["Publishing is stuck": the assistant follows one article across four languages](grafana-oss-assistant-03-publishing-is-stuck/README.md) | `make demo P=3` |
| 4 | ["What changed?": the assistant builds a timeline from annotations and alert history](grafana-oss-assistant-04-what-changed/README.md) | `make demo P=4` |
| 5 | [The silent failure: green dashboards, missing articles, and the assistant writes back](grafana-oss-assistant-05-the-silent-failure/README.md) | `make demo P=5` |
| 6 | [Guard rails: keeping an AI assistant safe, cheap and honest in Grafana Cloud](grafana-oss-assistant-06-guard-rails/README.md) | `make demo P=6` |

Every number, log line, trace id and screenshot comes from real runs against a Grafana Cloud stack on 2026-09-22.
Publishing: one part every two or three days; the `series` front-matter field gives dev.to the "Part x of 6" navigation.
