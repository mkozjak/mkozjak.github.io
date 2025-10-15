---
layout: post
title: "Interface Segregation in Go: Lessons from a TUI Keyboard Handler"
comments: false
keywords: "ddd golang refactoring go isp interfaces"
---

While working on [Blutui](https://github.com/mkozjak/blutui), a terminal music player my Bluesound Hi-Fi amplifier, I ran into a classic interface design problem that perfectly illustrates the Interface Segregation Principle in Go.

**Setup**

The keyboard package needs to coordinate between several components:

```go
type GlobalHandler struct {
    app     app.FocusStopper
    player  player.Controller
    library library.Command
    pages   pagesManager
    bar     *bar.Bar
}
```

At first glance, this looks clean. But let's examine what these interfaces actually require.

**Challenges**

The original approach would have been to create a massive interface:

```go
// BAD: Fat interface violating ISP
type AppManager interface {
    // Focus methods
    SetFocus(p tview.Primitive) *tview.Application
    SetPrevFocused(p string)

    // App lifecycle
    Stop()
    Draw() *tview.Application

    // Status methods
    ShowBarComponent(p tview.Primitive)
    CurrentPage() string

    // Player methods
    Play(url string)
    Playpause()
    Stop()
    VolumeHold(bool)
    // ... 20+ more methods
}
```

Every component would be forced to depend on this monolithic interface, even if they only needed one or two methods.

**Solution: Focused Interfaces**

Instead, I segregated responsibilities into focused interfaces:

```go
// Each interface has a single, clear purpose
type FocusStopper interface {
    Focuser
    Stopper
}

type Focuser interface {
    SetFocus(p tview.Primitive) *tview.Application
    SetPrevFocused(p string)
}

type Stopper interface {
    Stop()
}

type Controller interface {
    Play(url string)
    Playpause()
    Stop()
    Next()
    Previous()
    VolumeHold(bool)
    ToggleMute()
    ToggleRepeatMode()
    State() string
}
```

**Payoff**

Now each component depends only on what it actually uses:

```go
// Keyboard handler only gets what it needs
func NewGlobalHandler(
    a app.FocusStopper,     // Just focus + stop
    p player.Controller,     // Just playback control
    l library.Command,       // Just library operations
    pg pagesManager,         // Just page management
    b *bar.Bar,             // Concrete type - it's simple
) *GlobalHandler
```

The benefits are immediate:

- **Testability**: Mock only the methods you care about
- **Clarity**: Interface names clearly communicate intent
- **Flexibility**: Components can implement just what they need
- **Maintainability**: Changes to unrelated functionality don't ripple through

**Interface Composition**

Go's interface embedding makes this even more elegant:

```go
type FocusStopper interface {
    Focuser    // Embedded interface
    Stopper    // Embedded interface
}
```

The keyboard handler gets a clean API while the underlying `App` struct implements all the focused interfaces naturally.

**Takeaways**

If you can't give your interface a specific, meaningful name that describes exactly what it does, it's probably too broad.

`Controller` - controls playback
`FocusStopper` - manages focus and stops
`AppManager` - manages... everything?

Go's small interfaces aren't a limitation—they're a feature that forces better design.

---

*This approach reduced my testing complexity significantly and made the codebase much easier to navigate. Sometimes the best architecture decisions are the ones that make you write more interfaces, not fewer.
