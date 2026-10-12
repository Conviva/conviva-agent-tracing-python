# Conviva agent tracing for Python

Trace AI agents built in Python and send the traces to Conviva.

| Package | Use it for |
|---|---|
| [`conviva-agent-tracing`](https://pypi.org/project/conviva-agent-tracing/) | Full SDK: auto-instrumentation and export in one `init()` |
| [`conviva-otlp-exporter`](https://pypi.org/project/conviva-otlp-exporter/) | Exporter only, for apps that already produce OpenTelemetry spans |
| [`conviva-agent-traces-connector`](https://pypi.org/project/conviva-agent-traces-connector/) | Links agent spans to the Conviva client sensor's conversation ids; use with either of the above |

```bash
# Full SDK
pip install conviva-agent-tracing

# Exporter only
pip install conviva-otlp-exporter

# Add the traces connector to either
pip install conviva-agent-traces-connector
```

See the [integration guide](INTEGRATION_GUIDE.md) for setup, configuration and
the full API.

## About this repository

It holds the published wheels only (`dist/`), not their source. Releases are
published to PyPI from here by the **Publish** workflow when a GitHub release
is created; running the workflow by hand publishes to TestPyPI instead.
