# LinkedIn post

Paste the block below as the post text. Attach `img/cover.png` as the image. Replace `[article link]` with the dev.to URL once published.

---

Kafka says consumed. Lag is zero. The JVM is idle. The site says 404.

Part 5 is the silent failure: a consumer that acknowledges messages and then fails to persist them. No queue, no latency, nothing for a lag-based alert to see. The open-source assistant found it from a failure counter, one ERROR line per article, and a single red span inside an otherwise green trace.

Then, for the first time in the series, it wrote back: an annotation on the dashboard, a new alert rule, and a panel comparing consumed vs published. All through the same MCP server, with an Editor token handed out for that session only.

An assistant that only reads leaves you a transcript. One that can write, inside a narrow allow-list, leaves you a better Grafana than you had before the incident.

Full article: [article link]

#grafana #observability #kafka #ai #sre #opensource
