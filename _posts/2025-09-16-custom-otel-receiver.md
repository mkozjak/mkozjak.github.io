---
layout: post
title: "Building a Custom OpenTelemetry Receiver for IoT Device Telemetry"
comments: false
keywords: "iot opentelemetry otel mqtt metrics google cloud"
---

So there I was, staring at a fleet of IoT devices communicating with a message broker, and trying to get their data into my observability stack.

The thing is, these devices aren't your typical web services. They're running on ESP32 microcontrollers with limited memory and processing power, so they can't handle protocols like HTTP or gRPC. But they still needed to be monitored just like everything else in my infrastructure.

**When Standard Solutions Don't Cut It**

I had a bunch of devices publishing JSON messages to MQTT topics - stuff like heartbeats, error logs, and status updates. Each message had a device ID, message type, and some payload data. Pretty straightforward, except for one problem: OpenTelemetry receivers expect standard protocols, not lightweight pub/sub messaging.

I needed something that could:

1. Subscribe to message broker topics without breaking a sweat
2. Parse incoming JSON reliably
3. Convert everything to proper OpenTelemetry logs
4. Extract metrics from heartbeats for monitoring

I could've tried forcing existing receivers to work, or built some middleware. But that felt like working around the problem instead of solving it properly.

**Going Custom: Because Sometimes You Have To**

OpenTelemetry Collector's plugin architecture is quite powerful once you dig into it. Instead of fighting the system, I decided to build a receiver that understood my devices natively.

Here's what I learned works well:

**Keep the Connection Alive**: The receiver maintains a persistent connection to the message broker and subscribes to all the relevant topics. No polling, no batch processing delays - just real-time data flowing like it should.

**Smart Message Handling**: Different message types get different treatment. Error messages become error-level logs for when things go sideways. Heartbeat messages get processed into metrics because nothing says "device health" like a good uptime graph.

**Double Duty**: The same data stream serves two purposes - structured logs for debugging and troubleshooting, and metrics for monitoring dashboards.

**Building It Right**

I built this as a proper OpenTelemetry extension using the official plugin architecture. This meant:

- Creating a receiver factory that plays nicely with the collector's component system
- Implementing proper lifecycle management and error handling
- Converting device messages into OpenTelemetry's native log format
- Adding metadata that actually helps downstream processing

The receiver handles all the boring stuff automatically - connection management, reconnection when things go wrong, graceful shutdowns when you want to deploy updates.

**Pipeline Magic**

One of the coolest parts of this whole setup is leveraging OpenTelemetry's pipeline system. I ended up with multiple pipelines handling different aspects of the same data:

- **Device Logs Pipeline**: General device logs with filtering and routing based on severity
- **Metrics Pipeline**: Heartbeat messages converted to time-series data for monitoring
- **Error Pipeline**: Dedicated handling for device errors with appropriate alerting

This separation allows for different retention policies, processing rules, and destinations. Error logs might go to alerting systems while general telemetry goes to long-term storage.

**Deployment Reality Check**

To deploy this, I built a custom OpenTelemetry Collector distribution using the official builder tool. This involves telling the builder to include my custom receiver along with all the standard components.

This approach gives me:

- Version compatibility with the rest of the OpenTelemetry ecosystem
- Easy deployment (it's just another container)
- Integration with existing infrastructure
- A clear upgrade path as OpenTelemetry evolves

**What Actually Happened**

This custom receiver solved problems I didn't even know I had:

**Real-time Processing**: Device telemetry flows directly into my observability stack the moment it happens. No delays, no batch processing windows, no wondering about device status.

**Standards Compliance**: All device data follows OpenTelemetry standards, which means it works with literally any OpenTelemetry-compatible backend. Jaeger, Prometheus, cloud providers - pick your poison.

**Automatic Metrics**: Heartbeat data automatically becomes metrics for monitoring device health without any extra processing. Device status changes are immediately visible.

**Cost Optimization**: Smart filtering and processing rules mean I'm only storing and processing data that actually matters, keeping cloud costs reasonable.

**Scale When Ready**: The receiver handles connection management and can grow with my device fleet without requiring architectural changes. Started with dozens of devices, now handling hundreds.

**Things I Wish I'd Known Earlier**

**OpenTelemetry is More Flexible Than You Think**: The collector's plugin architecture is seriously powerful. You can extend it to handle virtually any data source or protocol. Don't be afraid to go custom when it makes sense.

**Configuration Gets Messy Fast**: As you add more processors and filters, keeping track of configuration becomes a real challenge. Document everything and test religiously.

**Signal Transformation is Gold**: The ability to convert logs to metrics (and vice versa) opens up crazy possibilities for deriving insights from your data.

**Monitor the Monitor**: In production, you need to watch the collector itself. Connection failures, processing errors, resource usage - it all matters when you're depending on it for observability.

**When Custom Makes Sense**

Building a custom receiver is worth it when:

- Your data sources speak protocols that don't exist in standard receivers
- You need real-time processing of specialized data formats
- Existing receivers would require significant preprocessing that defeats the purpose
- You want to optimize for your specific use case instead of general solutions

The time investment pays off when you have unique requirements that don't fit the standard mold. Plus, you end up with something that integrates seamlessly with the broader observability ecosystem.

For anyone dealing with IoT devices, industrial equipment, or other specialized telemetry sources, custom OpenTelemetry receivers offer a path to unified observability without compromising on functionality. Just don't underestimate the configuration complexity - future you will thank present you for good documentation.

*Got similar observability challenges? I'd love to hear how you're solving them. Hit me up!*
