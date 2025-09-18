---
layout: post
title: "Building a Custom OpenTelemetry Receiver for IoT Device Telemetry"
comments: false
keywords: "iot opentelemetry otel mqtt metrics google cloud"
---

When you're dealing with IoT devices that communicate over message brokers, getting their telemetry data into your observability stack can be challenging. Standard OpenTelemetry receivers expect HTTP or gRPC endpoints, but many IoT devices publish data to message brokers using lightweight protocols. This is the story of how I built a custom OpenTelemetry receiver to bridge this gap.

**The Problem**

I had a fleet of IoT devices sending telemetry data to a message broker. These devices were publishing structured JSON messages containing logs, heartbeats, and status information. Each message included a device identifier, message type, and payload data.

The challenge was getting this data into my OpenTelemetry-based observability pipeline. I needed a way to:

1. Subscribe to message broker topics
2. Parse the incoming JSON messages
3. Convert them to OpenTelemetry log format
4. Extract metrics from specific message types (like heartbeats)

**The Solution: A Custom Receiver**

OpenTelemetry Collector's architecture makes it possible to build custom components that fit your specific needs. Rather than trying to force-fit existing receivers or writing complex middleware, I decided to build a receiver that natively understood my device communication protocol.

### Key Design Decisions

**Message Broker Integration**: The receiver establishes a persistent connection to the message broker and subscribes to device telemetry topics. This ensures real-time data flow without polling or batch processing delays.

**Intelligent Message Parsing**: Different device message types require different handling. Error messages become error-level logs, while heartbeat messages are processed to extract operational metrics like uptime and device status.

**Dual-Purpose Data Flow**: The same incoming data stream serves two purposes - generating structured logs for debugging and troubleshooting, while simultaneously creating metrics for monitoring and alerting.

**Implementation Approach**

Instead of modifying the core OpenTelemetry Collector, I built this as a proper extension using the collector's plugin architecture. This meant:

- Creating a receiver factory that integrates with the collector's component system
- Implementing proper lifecycle management (start, stop, error handling)
- Converting raw device messages into OpenTelemetry's native log format
- Adding appropriate metadata and attributes for downstream processing

The receiver handles connection management, automatic reconnection, and graceful shutdown scenarios.

**Advanced Pipeline Configuration**

One of the most powerful aspects of this solution is leveraging OpenTelemetry Collector's pipeline system. I configured multiple pipelines to handle different aspects of the data:

- **Device Logs Pipeline**: Processes general device logs with appropriate filtering and routing
- **Metrics Pipeline**: Converts heartbeat messages into time-series metrics for monitoring
- **Error Pipeline**: Handles device error messages with specific alerting rules

This separation allows for different retention policies, processing rules, and export destinations based on data type and importance.

**Custom Collector Distribution**

To deploy this receiver, I built a custom OpenTelemetry Collector distribution using the official collector builder. This approach ensures:

- Version compatibility with the broader OpenTelemetry ecosystem
- Easy deployment and configuration management
- Integration with existing observability infrastructure
- Future upgrade path as OpenTelemetry evolves

**Results and Benefits**

This custom receiver solved several critical problems:

**Real-time Visibility**: Device telemetry now flows directly into our observability stack in real-time, enabling immediate alerting on device issues.

**Unified Data Model**: All device data follows OpenTelemetry standards, making it compatible with any OpenTelemetry-compliant backend (Jaeger, Prometheus, cloud providers, etc.).

**Operational Metrics**: Heartbeat data automatically becomes metrics for monitoring device health, uptime, and connectivity status.

**Cost Optimization**: Intelligent filtering and processing rules ensure we're only storing and processing relevant data, reducing storage and compute costs.

**Scalability**: The receiver handles connection management and can scale with the device fleet without requiring architectural changes.

**Lessons Learned**

Building this custom receiver taught me several valuable lessons:

**OpenTelemetry's Flexibility**: The collector's plugin architecture is more powerful than I initially realized. You can extend it to handle virtually any data source or protocol.

**Configuration Complexity**: As you add more processors and filters, configuration management becomes critical. Documentation and testing are essential.

**Signal Transformation**: The ability to convert logs to metrics (and vice versa) opens up interesting possibilities for deriving insights from different signal types.

**Production Considerations**: Real-world deployment requires careful attention to error handling, connection resilience, and monitoring the collector itself.

**When to Build Custom Components**

A custom receiver makes sense when:

- Your data sources don't speak standard protocols
- You need real-time processing of specialized data formats
- Existing receivers would require significant preprocessing
- You want to optimize for your specific use case

The investment in building custom OpenTelemetry components pays off when you have unique requirements that don't fit standard patterns. The result is a robust, maintainable solution that integrates seamlessly with the broader observability ecosystem.

For teams dealing with IoT devices, industrial equipment, or other specialized telemetry sources, custom OpenTelemetry receivers offer a path to unified observability without compromising on functionality or performance.
