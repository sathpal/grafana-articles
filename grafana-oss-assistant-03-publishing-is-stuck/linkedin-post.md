# LinkedIn post

Paste the block below as the post text. Attach `img/cover.png` as the image. Replace `[article link]` with the dev.to URL once published.

---

"Publishing is stuck." Nothing is down. Every panel is green except one.

In Part 3 the open-source assistant follows a single approved article from a Python API, through Kafka, into a Java consumer, a Go service, Redis and Postgres, and proves in nine tool calls that a batch job's config change flooded the topic a single consumer serves.

The proof is one trace: 66 seconds waiting in Kafka, 165 milliseconds of actual work. Trace context crossed the message bus because the Python producer put it in the headers, the OpenTelemetry Java agent read it, and the Go service passed it on.

The part I liked: it named the design flaw (one consumer for two topics), not just the trigger.

Full article: [article link]

#grafana #observability #opentelemetry #kafka #ai #sre
