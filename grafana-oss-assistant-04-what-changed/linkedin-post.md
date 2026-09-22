# LinkedIn post

Paste the block below as the post text. Attach `img/cover.png` as the image. Replace `[article link]` with the dev.to URL once published.

---

Two configuration changes, two minutes apart. Both visible in Grafana. Only one is the root cause.

Part 4: the assistant builds a timeline instead of a guess. Range queries at 15-second steps for the first breach of each signal, Grafana's alert history read straight out of Loki, and the change annotations that CI/CD, feature flags and batch jobs should always write.

Verdict: the quiet change (a Go service told to cache images without a TTL) was the trigger; the loud one (a forced re-render batch) was the amplifier. It even used a counter-example from the previous incident to rule the batch out.

If you take one thing from this series: write change annotations. Root cause becomes lookup instead of inference.

Full article: [article link]

#grafana #observability #sre #ai #opensource
