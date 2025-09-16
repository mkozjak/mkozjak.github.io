---
layout: post
title: "Debugging OpenTelemetry: When Your Metrics Disappear Into The Void"
comments: false
keywords: "iot opentelemetry otel mqtt metrics google cloud"
---
Debugging OpenTelemetry: When Your Metrics Disappear Into The Void
==================================================================

*September 16, 2025*

Sometimes you think you've got everything set up perfectly, but then reality hits you like a freight train.
That's exactly what happened when I was trying to wire and convert custom logs (simple heartbeats) from our IoT devices as metrics into Google Cloud Monitoring using OpenTelemetry.

**The setup seemed straightforward:**
- Receive and filter logs
- Parse JSON log bodies containing device status and uptime
- Transform them into gauge metrics using the `signaltometrics` connector  
- Export to Google Cloud Monitoring
- Profit! 📈

**What actually happened:** Metrics showed up as "active" in Google Cloud but with zero data points. Classic.

## The Detective Work Begins

First, I had to verify that the data pipeline was actually working. The beauty of OpenTelemetry is you can add a `file` exporter to see exactly what's being generated:

```yaml
exporters:
  googlecloud:
  file/metrics:
    path: /tmp/metrics.json
    format: json
```

Running this showed me that metrics were being created perfectly:

```json
{
  "name": "device.uptime",
  "gauge": {
    "dataPoints": [{
      "attributes": [{"key": "uuid", "value": {"stringValue": "12423535"}}],
      "asInt": "200"
    }]
  }
}
```

So the data was there. Google Cloud just wasn't showing it.

## The Real Culprit: OTTL Version Mismatch

When I tried to send test data using `telemetrygen`, I hit this error:

```
failed processing logs: failed to execute statement: 
set(log.attributes["uuid"], ParseJSON(log.body)["uuid"]), 
key not found in map
```

Wait, what? The error showed `log.attributes` and `log.body`, but my config used the newer syntax:

```yaml
# What I had
- set(attributes["uuid"], ParseJSON(body)["uuid"])

# What the collector actually expected  
- set(log.attributes["uuid"], ParseJSON(log.body)["uuid"])
```

Turns out OpenTelemetry Collector v0.130.0 still expected the older OTTL syntax with explicit `log.` prefixes. The newer documentation shows the simplified syntax, but not all collector versions support it yet.

## The Missing Piece: Resource Detection

Even after fixing the OTTL syntax, metrics still appeared under a "generic node" resource type instead of proper Kubernetes resources. The solution was adding the `resourcedetection` processor:

```yaml
processors:
  resourcedetection:
    detectors: [env, gcp]
    timeout: 5s
    override: false

service:
  pipelines:
    metrics:
      receivers: [signaltometrics]
      processors: [resourcedetection]  # This was the key
      exporters: [googlecloud]
```

I'm running Kubernetes on Google Cloud, so the `gcp` detector was sufficient.

This automatically detected and added the proper GKE resource attributes:
- `k8s.cluster.name`
- `cloud.availability_zone` 
- `host.name`

Now Google Cloud could properly categorize the metrics instead of lumping them under "generic node."

## The Final Working Configuration

Here's what the complete working setup looked like:

```yaml
processors:
  transform/parse_device_body:
    log_statements:
      - context: log
        statements:
          # Note the log. prefixes - crucial for older collector versions
          - set(log.attributes["uuid"], ParseJSON(log.body)["uui"])
          - set(log.attributes["uptime"], Int(ParseJSON(log.body)["uptime"]))
          - set(log.attributes["status"], 0) where ParseJSON(log.body)["status"] == "ok"
          - set(log.attributes["status"], 1) where ParseJSON(log.body)["status"] != "ok"

  resourcedetection:
    detectors: [env, gcp]
    timeout: 10s
    override: false

connectors:
  signaltometrics:
    logs:
      - name: device.uptime
        description: iot device heartbeat signals
        gauge:
          value: attributes["uptime"]
        attributes:
          - key: uuid
      - name: device.status
        description: iot device status signals  
        gauge:
          value: attributes["status"]
        attributes:
          - key: uuid

service:
  pipelines:
    logs/mqtt_metrics:
      receivers: [otlp]
      processors: [transform/parse_device_body]
      exporters: [signaltometrics]
    metrics:
      receivers: [signaltometrics] 
      processors: [resourcedetection]
      exporters: [googlecloud]
```

## Lessons Learned

1. **Always add debug exporters** when troubleshooting. The `file` and `debug` exporters are lifesavers for seeing what's actually happening in your pipeline. Do not use them in production indefinitely, though!

2. **OTTL syntax varies by collector version.** Don't assume the latest documentation matches your collector version - check what your specific version expects.

3. **Resource detection is crucial for cloud platforms.** Without proper resource attributes, your metrics might get categorized incorrectly or not display at all.

4. **Test with known data first.** Using `telemetrygen` to send controlled test data helped isolate the OTTL parsing issue quickly.

The whole debugging process took a few hours, but now our iot devices are happily sending uptime and status metrics to Google Cloud Monitoring. Sometimes the best solutions come from methodically working through each piece of the pipeline until you find where it's actually breaking.

*Have you run into similar OpenTelemetry gotchas? I'd love to hear about your debugging adventures.*

## Bonus tip

1. Never, and I mean, never, leave `debug` exporter running in production. You might skyrocket your bill after ingesting telemetry data for a small period of time. =)

---

**Mario Kozjak** · [GitHub](https://github.com/mariokozjak) | [LinkedIn](https://linkedin.com/in/mariokozjak) | [Twitter
