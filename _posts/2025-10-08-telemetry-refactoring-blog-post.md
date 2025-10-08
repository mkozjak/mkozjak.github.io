---
layout: post
title: "From Telemetry Chaos to Maintainable Code: How We Eliminated 900 Lines of Boilerplate in Go"
comments: false
keywords: "opentelemetry tracing golang refactoring observability telemetry go"
---

When you're building production systems, observability isn't optional—it's mandatory. But there's a fine line between good telemetry and telemetry that makes your codebase a nightmare to maintain. Especially when auto-instrumentation is not used, and wehn a demand for manually instrumenting business logic is a must-have.

I recently tackled a refactoring project that perfectly illustrates this challenge. Our Go backend had comprehensive OpenTelemetry tracing, which was great for debugging production issues. But, unfortunately, the tracing code was drowning out the business logic.

**The Problem: When Good Intentions Create Bad Code**

Here's what our handlers looked like before the refactoring:

```go
func (h *DeviceHandler) HandleDevice(res http.ResponseWriter, req *http.Request) {
    // 15+ lines of tracing setup
    tracer := otel.Tracer("device-service")
    ctx, span := tracer.Start(req.Context(), "handler.get_device")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.method", "GET"),
        attribute.String("http.route", "/api/v2/devices/{id}"),
        attribute.String("device.id", deviceId),
        attribute.String("user.id", userId),
    )

    // Finally, the actual business logic (3 lines)
    device, err := h.service.GetDevice(ctx, userId, deviceId)
    if err != nil {
        // More tracing boilerplate
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(res, "Failed fetching the device", http.StatusInternalServerError)
        return
    }

    // Success path tracing
    span.SetStatus(codes.Ok, "")

    // JSON marshaling...
}
```

This pattern was repeated across **38 methods** in our codebase. The signal-to-noise ratio was terrible—you had to wade through telemetry setup to understand what the code actually did.

**The Solution: A Clean Telemetry Helper**

The fix was surprisingly simple. I created a `telemetry.Trace` helper that encapsulates all the boilerplate:

```go
func Trace(ctx context.Context, tracerName, operationName string, attributes map[string]string) (context.Context, func(error)) {
    tracer := otel.Tracer(tracerName)
    ctx, span := tracer.Start(ctx, operationName)

    // Set attributes
    for key, value := range attributes {
        span.SetAttribute(attribute.String(key, value))
    }

    return ctx, func(err error) {
        defer span.End()
        if err != nil {
            span.RecordError(err)
            span.SetStatus(codes.Error, err.Error())
        } else {
            span.SetStatus(codes.Ok, "")
        }
    }
}
```

Now the same handler looks like this:

```go
func (h *DeviceHandler) HandleDevice(res http.ResponseWriter, req *http.Request) {
    var tErr error
    ctx, done := telemetry.Trace(req.Context(), tracerName, "handler.get_device", map[string]string{
        "http.method": "GET",
        "device.id":   deviceId,
        "user.id":     userId,
    })
    defer func() { done(tErr) }()

    // Pure business logic
    device, err := h.service.GetDevice(ctx, userId, deviceId)
    if err != nil {
        tErr = err
        http.Error(res, "Failed fetching the device", http.StatusInternalServerError)
        return
    }

    // JSON marshaling...
}
```

I wanted to avoid using heavier abstraction methods like decorators and extensive use of interfaces in order to [KISS](https://en.wikipedia.org/wiki/KISS_principle).

**The Variable Shadowing Challenge**

During the refactoring, I hit an interesting Go idiom problem. My first instinct was to use `err` everywhere, but handlers have a unique challenge—they perform multiple operations that create different `err` variables in different scopes:

```go
device, err := h.service.GetDevice(...)  // First err
// ...
b, err := json.Marshal(device)           // Second err (shadows the first!)
```

The solution was to use `tErr` (telemetry error) as a dedicated variable for tracking errors across the entire handler scope, while still using `err` for local operations:

```go
var tErr error                           // Telemetry tracker
defer func() { done(tErr) }()

device, err := h.service.GetDevice(...)  // Local operation
if err != nil {
    tErr = err                           // Assign for telemetry
    // handle error
}

b, err := json.Marshal(device)           // Different local operation
if err != nil {
    tErr = err                           // Assign for telemetry
    // handle error
}
```

This keeps the code idiomatic (short variable names) while preventing variable shadowing issues.

**The Results: Numbers Don't Lie**

The refactoring delivered impressive results:

- **600 insertions, 1,495 deletions**: Net removal of ~900 lines of code
- **5 files changed**: Handlers, services, and repositories
- **Zero regressions**: All integration tests passed after the migration
- **19 methods migrated**: From verbose to clean telemetry

We went from 15-20 lines of tracing boilerplate per method to 4 lines. That's a 75-80% reduction in telemetry noise.

**What This Means for Engineering Teams**

This refactoring illustrates a broader principle: **observability should enhance your code, not obscure it**.

When telemetry becomes the dominant pattern in your functions, you've lost the plot. The business logic should be the star, with observability working quietly in the background.

Here are the key benefits we gained:

1. **Faster development**: Engineers can focus on business logic instead of tracing setup
2. **Easier debugging**: The actual code flow is immediately visible
3. **Consistent observability**: Centralized error handling means better trace data
4. **Reduced cognitive load**: Less context switching between telemetry and business logic

**The Broader Lesson**

Good engineering isn't just about making things work—it's about making them maintainable. When we added comprehensive tracing to our codebase, we solved one problem (observability) but created another (maintainability).

The refactoring solved both. We kept all the observability benefits while dramatically improving code readability.

If your codebase has similar telemetry sprawl, consider creating helper functions that encapsulate the repetitive patterns. Your future self (and your teammates) will thank you.

---

The refactoring is complete, all tests are passing, and we've shipped cleaner, more maintainable code. Sometimes the best engineering solutions are the simplest ones.

*Want to discuss observability patterns or similar refactoring challenges? [Drop me a line](mailto:mario.kozjak@elpheria.com) — I'd love to hear about your experiences with telemetry in production systems.*
