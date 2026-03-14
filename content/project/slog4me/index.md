+++
draft = false
date = '2026-03-14T21:39:35+07:00'
title = 'slog4me'
type = 'project'
description = 'A technical deep dive into building a lightweight Go library that brings template-based formatting and level-based routing to the standard log/slog package — using generics, text/template, and a functional options API.'
image = ''
repository = 'https://github.com/mnabila/slog4me'
languages = ['go']
tools = ['slog', 'text-template']
+++

Go's `log/slog` package, introduced in Go 1.21, gave the standard library a structured logging interface that most Go developers had been reaching for third-party libraries to get. It is well-designed, extensible, and production-ready. But it ships with two built-in handlers — `TextHandler` and `JSONHandler` — and neither supports custom output formatting.

If you want your logs to follow a specific layout, route different levels to different files, or transform log records into domain-specific structures before rendering, you are on your own. slog4me is a small library that fills that gap: template-based log formatting with level-based routing, built on Go generics.

## Problem Background

In several projects, I needed log output that did not fit the default `slog` formats. The requirements were specific:

- **Custom layouts** — logs needed to follow a particular format for consumption by downstream systems. Neither the default text nor JSON output matched the expected structure
- **Level-based routing** — debug logs should go to a debug file, info and error logs to a separate file, and critical errors to stderr. The standard handlers write everything to a single destination
- **Structured transformation** — log records contained attributes that needed to be mapped to specific fields in the output, not just dumped as key-value pairs

The standard approach would be to implement the `slog.Handler` interface from scratch for each project. But the pattern was always the same: transform the record, render it through a template, write it to the right destination. This repetition suggested a reusable abstraction.

## Solution Overview

slog4me is a Go library that provides a single, generic `slog.Handler` implementation with three capabilities:

1. **Template-based formatting** — define log output layouts using Go's `text/template` syntax
2. **Level-based routing** — send different log levels to different `io.Writer` destinations
3. **Custom data mapping** — transform `slog.Record` into any user-defined struct via a mapper function, then pass that struct to the template for rendering

The library exposes a functional options API and leverages Go generics so the mapper and template are fully type-safe.

**Tech stack:** Go 1.24+, `log/slog`, `text/template`, `golang.org/x/sync`

**My role:** Sole developer — API design, implementation, and documentation

## System Architecture

The library is intentionally minimal — two files, one external dependency:

```
template_handler.go    — TemplateHandler[T] implementation (slog.Handler interface)
option.go              — Functional option constructors (WithTemplateWriter, WithMapper)
examples/main.go       — Usage example
```

The core architecture has three components:

**`TemplateHandler[T]`** — a generic struct that implements `slog.Handler`. It holds a list of `TemplateWriter` destinations and a user-supplied mapper function. When `Handle()` is called, it maps the `slog.Record` to the user's struct `T`, then iterates over writers and executes the matching templates concurrently.

**`TemplateWriter[T]`** — pairs an `io.Writer` with a parsed `text/template` and a list of `slog.Level` values. Only log records whose level matches one of the configured levels are written to this destination.

**`Mapper[T]`** — a function type `func(slog.Record) T` that the user defines. It transforms raw `slog.Record` data (timestamp, level, message, attributes) into a domain-specific struct that the template can reference by field name.

The data flow for each log call:

```
slog.Info("message", attrs...)
  → TemplateHandler.Handle(record)
    → mapper(record) → Data{...}         // Transform to user struct
    → for each writer:
        if level matches:
          template.Execute(writer, data)  // Render and write (concurrent)
```

### API Surface

The public API consists of three functions:

```go
// Create a new handler with functional options
func NewTemplateHandler[T any](opts ...HandlerOption[T]) (slog.Handler, error)

// Add an output destination with a template layout and level filter
func WithTemplateWriter[T any](w io.Writer, layout string, levels ...slog.Level) HandlerOption[T]

// Set the record-to-struct mapper function
func WithMapper[T any](mapper Mapper[T]) HandlerOption[T]
```

## Key Features

- **Go generics for type safety** — the `T` type parameter flows through the entire API. The mapper produces a `T`, the template renders a `T`, and the compiler ensures they agree. No `interface{}` casting, no runtime type assertions
- **Multiple outputs with independent templates** — each `WithTemplateWriter` call adds a new destination with its own layout string and level filter. A debug file can use a verbose layout with all fields, while a production log can use a compact format with only timestamp, level, and message
- **Concurrent writes** — when a log record matches multiple writers, template execution happens concurrently using `errgroup.Group`, with a shared mutex protecting against interleaved output
- **Standard `slog.Handler` interface** — the handler plugs directly into `slog.New()`, so it works with any code that already uses the standard `slog` logger. No custom logger type, no wrapper — just a handler
- **Functional options pattern** — configuration uses the idiomatic Go options pattern, making the API extensible without breaking changes
- **Minimal dependencies** — the only external dependency is `golang.org/x/sync` for `errgroup`. Everything else uses the standard library

## Technical Challenges and Solutions

**Designing the generic type flow.** The central design question was how to connect the user's data struct, the mapper function, and the template in a way that is both type-safe and ergonomic. Go generics made this possible: the type parameter `T` is declared on `NewTemplateHandler[T]` and propagates to `WithTemplateWriter[T]`, `WithMapper[T]`, and the internal `TemplateHandler[T]` struct. The user declares their data type once, and the compiler enforces consistency everywhere else.

The alternative — using `interface{}` and runtime reflection — would have worked but would have pushed type errors to runtime and required more defensive code in the handler.

**Concurrent writes with shared state.** When a log record matches multiple writers, executing templates sequentially would add latency. Using `errgroup.Group` allows concurrent execution, but `text/template.Execute` writing to an `io.Writer` is not inherently thread-safe. The solution is a shared `sync.Mutex` that serializes the actual write operations while allowing template evaluation to overlap:

```go
g.Go(func() error {
    h.mu.Lock()
    defer h.mu.Unlock()
    return w.Template.Execute(w.Writer, data)
})
```

This is a pragmatic trade-off: the mutex prevents interleaved output at the cost of serializing writes. For logging workloads, where individual writes are small and fast, this is acceptable.

**Keeping the `slog.Handler` contract correct.** The `slog.Handler` interface requires four methods: `Enabled`, `Handle`, `WithAttrs`, and `WithGroup`. For this library's use case — where the mapper function handles all attribute extraction — `WithAttrs` and `WithGroup` return the handler unchanged. This is a deliberate simplification: the mapper receives the full `slog.Record` including all attributes, so it has complete control over how attributes are processed. The trade-off is that handler-level attribute accumulation (via `logger.With()`) is not supported, but for template-based formatting where the user defines the exact output structure, this has not been a limitation in practice.

**Template parsing at configuration time.** Templates are parsed once during `WithTemplateWriter` and reused for every log call. If a template has a syntax error, it fails at handler construction time with a clear error, not at the first log call in production. This fail-fast behavior is important for a logging library — you want to know immediately if your format string is broken.

## Lessons Learned

**Generics make library APIs dramatically cleaner.** Before Go generics, this library would have required either code generation or `interface{}` with runtime type assertions. Generics allowed the entire mapper-template pipeline to be type-checked at compile time, which is exactly the kind of problem they were designed to solve.

**The functional options pattern scales well.** Starting with `WithTemplateWriter` and `WithMapper` as the only options keeps the API surface small. But because each option is just a function, adding new configuration options (e.g., `WithErrorHandler`, `WithBuffering`) in the future requires no breaking changes to existing consumers.

**Small libraries are easier to trust.** The entire implementation is under 100 lines of code across two files. This makes it easy to audit, easy to understand, and easy to vendor if needed. For infrastructure-level concerns like logging, the ability to read and understand the entire library in five minutes is a feature.

**`slog.Handler` is a well-designed extension point.** Go's decision to make `slog` handler-based rather than logger-based means that custom formatting, routing, and transformation can all happen behind a standard interface. Any code using `slog.Logger` gets the custom behavior for free — no logger wrapper, no adapter pattern.

## Conclusion

slog4me is a small library solving a specific problem: bringing template-based formatting and level-based routing to Go's standard structured logging. It leverages generics for type safety, `text/template` for flexible output formatting, and the `slog.Handler` interface for seamless integration with existing code.

Installation:

```bash
go get github.com/mnabila/slog4me
```

Quick example:

```go
handler, err := slog4me.NewTemplateHandler(
    slog4me.WithTemplateWriter[Data](os.Stdout, "{{.Timestamp}}|{{.Level}}|{{.Message}}\n", slog.LevelInfo),
    slog4me.WithMapper(func(record slog.Record) Data {
        return Data{
            Timestamp: record.Time.Format("2006-01-02 15:04:05"),
            Level:     record.Level.String(),
            Message:   record.Message,
        }
    }),
)

logger := slog.New(handler)
logger.Info("server started")
// Output: 2026-03-14 12:00:00|INFO|server started
```

The full source is available on [GitHub](https://github.com/mnabila/slog4me). It is intentionally small — under 100 lines — because sometimes the best library is the one you can read in its entirety before deciding to use it.
