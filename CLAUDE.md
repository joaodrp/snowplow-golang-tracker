# Snowplow Golang Tracker - Architecture Guide

## Project Overview

The Snowplow Golang Tracker is an event tracking library for sending analytics events to Snowplow collectors. It provides a robust, concurrent-safe implementation for tracking user interactions, page views, and custom events with support for batching, persistence, and retry mechanisms.

**Key Technologies**: Go 1.19+, HTTP/HTTPS protocols, JSON payloads, SQLite3/Memory storage, UUID generation, Base64 encoding

## Development Commands

```bash
make all          # Build all packages
make test         # Run tests with coverage
make format       # Format code with gofmt
make lint         # Run golint
make tidy         # Tidy go modules
make goveralls    # Run tests and submit to Coveralls
make clean        # Clean build artifacts
```

## Architecture

The tracker follows a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────┐
│              Client Application              │
├─────────────────────────────────────────────┤
│                   Tracker                    │ ← Event creation & validation
├─────────────────────────────────────────────┤
│     Emitter      │      Subject              │ ← Event enrichment & batching
├─────────────────────────────────────────────┤
│              Storage Interface               │ ← Event persistence
├─────────────────────────────────────────────┤
│   Memory Storage │   SQLite3 Storage         │ ← Storage implementations
└─────────────────────────────────────────────┘
```

## Core Architectural Principles

1. **Functional Options Pattern**: All components use functional options for initialization
2. **Interface-Based Storage**: Storage abstraction allows pluggable implementations
3. **Concurrent Safety**: Thread-safe operations using channels and mutexes
4. **Fail-Safe Persistence**: Events persisted before sending to handle failures
5. **Batch Processing**: Automatic batching with configurable limits
6. **Builder Pattern**: Events built using structured types with validation

## Layer Organization & Responsibilities

### `/tracker` - Core Tracking Layer
- **tracker.go**: Main tracker orchestration, event routing
- **emitter.go**: HTTP transport, batching, retry logic
- **events.go**: Event type definitions and validation
- **subject.go**: User/session context management
- **self_describing_json.go**: Iglu schema support
- **constants.go**: Protocol constants and schemas

### `/pkg` - Shared Utilities
- **payload/**: Event payload construction and serialization
- **storage/**: Storage interface and implementations
- **common/**: Helper functions and utilities

### `/examples` - Usage Examples
- Demonstrates proper initialization and event tracking patterns

## Critical Import Patterns

```go
// ✅ Correct: Use v3 module path
import (
    sp "github.com/snowplow/snowplow-golang-tracker/v3/tracker"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/common"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/storage/memory"
)

// ❌ Wrong: Missing v3 version
import "github.com/snowplow/snowplow-golang-tracker/tracker"
```

## Essential Library Patterns

### Tracker Initialization Pattern
```go
// ✅ Correct: Use functional options with required emitter
emitter := sp.InitEmitter(
    sp.RequireCollectorUri("collector.example.com"),
    sp.RequireStorage(*memory.Init()),
)
tracker := sp.InitTracker(sp.RequireEmitter(emitter))

// ❌ Wrong: Missing required emitter
tracker := sp.InitTracker() // Will panic
```

### Event Tracking Pattern
```go
// ✅ Correct: Initialize event struct, then track
event := sp.PageViewEvent{
    PageUrl: common.NewString("https://example.com"),
}
tracker.TrackPageView(event)

// ❌ Wrong: Direct string values without pointers
event := sp.PageViewEvent{PageUrl: "example.com"}
```

### Storage Selection Pattern
```go
// ✅ Correct: Initialize storage before emitter
storage := memory.Init() // or sqlite3.Init("events.db")
emitter := sp.InitEmitter(
    sp.RequireStorage(*storage),
)

// ❌ Wrong: Nil storage
emitter := sp.InitEmitter(sp.RequireStorage(nil))
```

### Blocking Flush Pattern
```go
// ✅ Correct: Stop emitter then flush
tracker.Emitter.Stop()
tracker.BlockingFlush(5, 10) // attempts, sleep ms

// ❌ Wrong: Flush without stopping
tracker.BlockingFlush(5, 10)
```

## Model Organization Pattern

### Event Models
- All event fields use **pointer types** for optional values
- Required fields validated in `Init()` methods
- Timestamp and EventId auto-generated if nil

### Payload Structure
```go
type Payload struct {
    Pairs map[string]string
}
// Only non-nil, non-empty values added to payload
```

### Storage Interface
```go
type Storage interface {
    AddEventRow(payload.Payload) bool
    DeleteEventRows([]int) int64
    GetEventRowsWithinRange(int) []EventRow
}
```

## Common Pitfalls & Solutions

### Nil Pointer Issues
```go
// ❌ Wrong: Direct string assignment
event := sp.StructuredEvent{
    Category: "shop",  // Will fail
}

// ✅ Correct: Use common.NewString helper
event := sp.StructuredEvent{
    Category: common.NewString("shop"),
}
```

### Missing Required Fields
```go
// ❌ Wrong: Missing required fields
event := sp.PageViewEvent{}
tracker.TrackPageView(event) // Will panic

// ✅ Correct: Set required fields
event := sp.PageViewEvent{
    PageUrl: common.NewString("https://example.com"),
}
```

### Concurrent Access
```go
// ❌ Wrong: Shared tracker without synchronization
go tracker.TrackPageView(event1)
go tracker.TrackPageView(event2)

// ✅ Correct: Tracker handles concurrency internally
tracker.TrackPageView(event1) // Thread-safe
```

### Resource Cleanup
```go
// ❌ Wrong: Not stopping emitter
// Resources may leak

// ✅ Correct: Proper cleanup
defer tracker.Emitter.Stop()
defer tracker.BlockingFlush(5, 10)
```

## File Structure Template

```
snowplow-golang-tracker/
├── tracker/                    # Core tracking components
│   ├── tracker.go             # Main tracker
│   ├── emitter.go             # Event transport
│   ├── events.go              # Event types
│   ├── subject.go             # User context
│   ├── self_describing_json.go # Custom events
│   ├── constants.go           # Protocol constants
│   └── *_test.go              # Unit tests
├── pkg/                       # Shared packages
│   ├── payload/               # Payload construction
│   ├── storage/               # Storage layer
│   │   ├── storageiface/      # Interface definition
│   │   ├── memory/            # In-memory storage
│   │   └── sqlite3/           # SQLite storage
│   └── common/                # Utilities
├── examples/                  # Usage examples
├── Makefile                   # Build commands
└── go.mod                     # Module definition
```

## Testing

### Test Organization & Structure

Tests are **co-located** with implementation files using the `*_test.go` suffix pattern:

```
tracker/
├── tracker.go         # Implementation
├── tracker_test.go    # Tests for tracker.go
├── emitter.go
└── emitter_test.go    # Tests for emitter.go
```

### Test Execution Commands

```bash
make test         # Run all tests with coverage report
make goveralls    # Run tests and submit to Coveralls
go test ./...     # Direct test execution
go test -v ./...  # Verbose output with test names
```

### Essential Testing Libraries

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"  // Assertions
    "github.com/jarcoal/httpmock"         // HTTP mocking
)
```

### HTTP Mocking Pattern

```go
// ✅ Correct: Activate, register, defer cleanup
httpmock.Activate()
defer httpmock.DeactivateAndReset()
httpmock.RegisterResponder("GET", "http://collector/i",
    httpmock.NewStringResponder(200, ""))

// ❌ Wrong: Missing deactivation
httpmock.Activate()
httpmock.RegisterResponder("GET", url, responder)
```

### Assertion Pattern

```go
// ✅ Correct: Create assert instance for cleaner code
assert := assert.New(t)
assert.NotNil(tracker)
assert.Equal("expected", actual)

// ❌ Wrong: Verbose assertions
assert.NotNil(t, tracker)
assert.Equal(t, "expected", actual)
```

### Storage Testing Pattern

Both storage implementations share test logic via helper functions:

```go
// ✅ Correct: Test both storage backends
func TestMemoryStorage(t *testing.T) {
    storage := *memory.Init()
    assertDatabaseOperations(assert.New(t), storage)
}
func TestSQLiteStorage(t *testing.T) {
    storage := *sqlite3.Init("test.db")
    assertDatabaseOperations(assert.New(t), storage)
}
```

### Panic Testing Pattern

```go
// ✅ Correct: Use defer/recover for panic assertions
defer func() {
    if err := recover(); err != nil {
        assert.Equal("Expected panic message", err)
    }
}()
tracker := InitTracker() // Code that should panic

// ❌ Wrong: Not testing panic conditions
tracker := InitTracker() // Will crash test
```

### Event Validation Testing

```go
// ✅ Correct: Test required fields and validation
event := PageViewEvent{
    PageUrl: common.NewString("http://example.com"),
}
event.Init()
assert.NotNil(event.EventId)    // Auto-generated
assert.NotNil(event.Timestamp)  // Auto-generated

// ✅ Correct: Test missing required fields
defer func() { recover() }()
event := PageViewEvent{} // Missing PageUrl
event.Init()             // Should panic
```

### Test Isolation Pattern

```go
// ✅ Correct: Clean state before each test
storage.DeleteAllEventRows()
// Run test operations
eventRows := storage.GetAllEventRows()
assert.Equal(0, len(eventRows))

// ❌ Wrong: Tests depending on previous state
// Test 1 adds data
// Test 2 assumes data exists
```

### Coverage Requirements

- Run with coverage: `make test` generates HTML report
- Coverage stored in: `build/coverage/coverage.html`
- CI/CD integration via GitHub Actions with Coveralls

### Common Testing Anti-patterns

```go
// ❌ Wrong: Hard-coded test data paths
storage := sqlite3.Init("/tmp/test.db")

// ✅ Correct: Use relative or temp paths
storage := sqlite3.Init("test.db")
```

```go
// ❌ Wrong: Not cleaning up HTTP mocks
httpmock.Activate() // Affects other tests

// ✅ Correct: Always defer cleanup
httpmock.Activate()
defer httpmock.DeactivateAndReset()
```

## Quick Reference

### Import Checklist
- [ ] Use v3 module path: `github.com/snowplow/snowplow-golang-tracker/v3`
- [ ] Import tracker as `sp` for clarity
- [ ] Import common for helper functions
- [ ] Import appropriate storage implementation

### Event Tracking Checklist
- [ ] Initialize storage (memory or sqlite3)
- [ ] Create emitter with required collector URI
- [ ] Initialize tracker with emitter
- [ ] Create event structs with pointer fields
- [ ] Call appropriate Track* method
- [ ] Stop emitter and flush on shutdown

### Testing Checklist
- [ ] Co-locate tests with implementation (`*_test.go`)
- [ ] Use `testify/assert` for clean assertions
- [ ] Mock HTTP calls with `httpmock` activation/cleanup
- [ ] Test both memory and sqlite3 storage backends
- [ ] Verify panics with defer/recover pattern
- [ ] Clean storage state between tests
- [ ] Check auto-generated fields (EventId, Timestamp)

## Contributing to CLAUDE.md

When adding or updating content in this document, please follow these guidelines:

### File Size Limit
- **CLAUDE.md must not exceed 40KB** (currently ~19KB)
- Check file size after updates: `wc -c CLAUDE.md`
- Remove outdated content if approaching the limit

### Code Examples
- Keep all code examples **4 lines or fewer**
- Focus on the essential pattern, not complete implementations
- Use `// ❌` and `// ✅` to clearly show wrong vs right approaches

### Content Organization
- Add new patterns to existing sections when possible
- Create new sections sparingly to maintain structure
- Update the architectural principles section for major changes
- Ensure examples follow current codebase conventions

### Quality Standards
- Test any new patterns in actual code before documenting
- Verify imports and syntax are correct for the codebase
- Keep language concise and actionable
- Focus on "what" and "how", minimize "why" explanations

### Multiple CLAUDE.md Files
- **Directory-specific CLAUDE.md files** can be created for specialized modules
- Follow the same structure and guidelines as this root CLAUDE.md
- Keep them focused on directory-specific patterns and conventions
- Maximum 20KB per directory-specific CLAUDE.md file

### Instructions for LLMs
When editing files in this repository, **always check for CLAUDE.md guidance**:

1. **Look for CLAUDE.md in the same directory** as the file being edited
2. **If not found, check parent directories** recursively up to project root
3. **Follow the patterns and conventions** described in the applicable CLAUDE.md
4. **Prioritize directory-specific guidance** over root-level guidance when conflicts exist

This Contributing section must be included at the end of every root CLAUDE.md file.