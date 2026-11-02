# ADR 0006: Distributed tracing with Tempo and OpenTelemetry

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites (always on for russetleaf.com demos; on demand for russetleaf.dev)

## Context
The design already has metrics (Prometheus), logs (Loki + Alloy), network flows (Hubble) and security events (Falco). Tracing was listed only as an optional WordPress-only extra, and the AWS plan had no step for it. The AI summary pipeline (WordPress → SQS → summarizer → LiteLLM gateway → Bedrock or llama.cpp) is asynchronous and crosses several services: when a summary is slow or fails, metrics and logs show *that* something went wrong, but not easily *where*.

Constraints: box 1 of the home lab has 8 GB of RAM and is close to its limit; EKS (demos only) has memory to spare; cost must stay near zero.

## Decision
- **Instrument with OpenTelemetry everywhere it matters**, built into the images and off unless an endpoint is configured:
  - WordPress: the OpenTelemetry PHP extension and Composer packages (auto-instrumentation for WordPress, database access and HTTP calls). **No WordPress plugin.**
  - The custom summary-queue must-use plugin adds the W3C `traceparent` to each SQS message's attributes.
  - The summarizer: the OpenTelemetry Python SDK, continuing the trace from the message.
  - The LiteLLM gateway: its built-in OpenTelemetry support.
- **Grafana Alloy** (already deployed for logs) receives traces over OTLP and forwards them to **Grafana Tempo** (single-binary mode, S3 storage).
- **russetleaf.com:** tracing **on for every demo**, 100% sampling, 7-day retention in a bucket that survives teardown.
- **russetleaf.dev:** tracing **on demand**, ~10% sampling of page requests and 100% of summary jobs, 3-day retention; Tempo removed when not needed if memory is tight.
- **Spans hold metadata only**: no post content, prompts or summaries.

## Alternatives considered
- **WordPress-only tracing (the previous plan):** misses the asynchronous AI pipeline, where tracing helps most.
- **Jaeger:** a well-known tracing backend, but a separate UI from Grafana, and its storage options are heavier for this setup; Tempo integrates directly with Grafana, Loki and Prometheus.
- **AWS X-Ray:** AWS-native, but doesn't cover the home site, and would split tracing between two tools.
- **A separate OpenTelemetry Collector:** unnecessary, since Alloy already runs on every node and handles OTLP.
- **Always-on Tempo at home:** doesn't fit box 1's memory alongside everything else.

## Consequences
- One trace shows a post's whole journey, including which model answered and how long each step took: useful for debugging and a strong demo.
- Trace IDs in logs link traces and logs in Grafana; exemplars link latency metrics to example traces.
- The summarizer and summary-queue plugin carry a little extra code for propagating trace context through SQS.
- Home tracing isn't continuous, so intermittent problems there may need tracing turned on and the issue reproduced.

## When to revisit
- More memory on box 1: keep Tempo on permanently at home.
- Trace volume growing beyond what single-binary Tempo handles: move to its scalable deployment mode.
- A move fully into AWS: reconsider X-Ray or AWS's managed options.

## Open items
- Home plan step 10.4: record Tempo's memory use and the sampling rates that work.
- AWS plan step 4.4: confirm the end-to-end trace (publish → summary saved) in a demo and add a screenshot to the docs.
- Confirm the current OpenTelemetry PHP package names for WordPress, database and HTTP auto-instrumentation when building the image.
