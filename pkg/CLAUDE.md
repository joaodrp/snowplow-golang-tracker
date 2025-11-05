# Package Library - Utilities and Storage Guide

## Package Overview

The `pkg` directory contains reusable utility packages and storage implementations that support the core tracker functionality. These packages provide payload construction, event persistence, and helper functions.

## Package Structure

```
pkg/
├── payload/          # Event payload construction
├── storage/          # Storage abstraction and implementations
│   ├── storageiface/ # Storage interface definition
│   ├── memory/       # In-memory storage (using go-memdb)
│   └── sqlite3/      # SQLite persistent storage
└── common/           # Helper functions and utilities
```

## Payload Package Patterns

### Payload Construction
```go
// ✅ Correct: Initialize then add fields
p := payload.Init()
p.Add("key", common.NewString("value"))
p.AddDict(map[string]string{"k": "v"})

// ❌ Wrong: Direct map manipulation
p.Pairs["key"] = "value"  // Bypasses validation
```

### JSON Encoding Pattern
```go
// ✅ Correct: Use AddJson with base64 flag
p.AddJson(data, true, "cx", "co")  // Encoded/plain keys

// ❌ Wrong: Manual JSON encoding
json, _ := json.Marshal(data)
p.Add("cx", &json)  // Missing base64
```

### Nil Safety Pattern
```go
// ✅ Correct: Add checks for nil/empty
p.Add(key, value)  // Only adds if non-nil, non-empty

// ❌ Wrong: Assume all values valid
if value != nil {
    p.Pairs[key] = *value  // Still might be empty
}
```

## Storage Interface Patterns

### Interface Definition
```go
type Storage interface {
    AddEventRow(payload.Payload) bool
    DeleteEventRows([]int) int64
    GetEventRowsWithinRange(int) []EventRow
    GetAllEventRows() []EventRow
    DeleteAllEventRows() int64
}
```

### Storage Selection
```go
// ✅ Correct: Choose based on requirements
// Memory: Fast, volatile, good for high throughput
storage := memory.Init()

// SQLite: Persistent, survives restarts
storage := sqlite3.Init("events.db")

// ❌ Wrong: Hardcode storage type
var storage *memory.StorageMemory  // Breaks abstraction
```

## Memory Storage Patterns

### Initialization
```go
// ✅ Correct: Use Init() factory
storage := memory.Init()

// ❌ Wrong: Direct struct creation
storage := &memory.StorageMemory{}  // Missing setup
```

### Index Overflow Handling
```go
// Automatic uint32 rollover handling
// Index wraps from 4294967295 → 0
// Handles collision detection
```

### Transaction Safety
```go
// All operations use memdb transactions
// Read operations use snapshots
// Write operations are atomic
```

## SQLite Storage Patterns

### Database Setup
```go
// ✅ Correct: Let Init handle schema
storage := sqlite3.Init("events.db")

// ❌ Wrong: Manual DB creation
db, _ := sql.Open("sqlite3", "events.db")
```

### Connection Management
```go
// Single connection, thread-safe
// Automatic retry on busy
// Prepared statements cached
```

### Schema Migration
```sql
-- Automatic table creation
CREATE TABLE IF NOT EXISTS events (
    id INTEGER PRIMARY KEY,
    event BLOB
)
```

## Common Package Patterns

### Pointer Helpers
```go
// ✅ Correct: Use helper functions
str := common.NewString("value")
num := common.NewInt64(123)
flt := common.NewFloat64(45.67)

// ❌ Wrong: Manual pointer creation
str := &"value"  // Syntax error
```

### UUID Generation
```go
// ✅ Correct: Use GetUUID for v4 UUIDs
eventId := common.GetUUID()

// ❌ Wrong: Custom ID generation
eventId := fmt.Sprintf("%d", time.Now().Unix())
```

### Timestamp Handling
```go
// ✅ Correct: Millisecond precision
ts := common.GetTimestamp()  // Unix ms

// ❌ Wrong: Second precision
ts := time.Now().Unix()  // Missing precision
```

### String Conversions
```go
// ✅ Correct: Nil-safe conversions
str := common.Int64ToString(nilableInt64)
str := common.Float64ToString(nilableFloat, 2)

// ❌ Wrong: Direct conversion
str := strconv.FormatInt(*nilableInt64, 10)  // Panic if nil
```

## Error Handling Patterns

### Storage Errors
```go
// ✅ Correct: Check return values
if !storage.AddEventRow(payload) {
    log.Println("Failed to store event")
}

// ❌ Wrong: Ignore storage failures
storage.AddEventRow(payload)  // Assume success
```

### Serialization Errors
```go
// ✅ Correct: Handle marshaling errors
data, err := common.DeserializeMap(bytes)
if err != nil {
    return nil, err
}

// ❌ Wrong: Ignore errors
data, _ := common.DeserializeMap(bytes)
```

## Performance Patterns

### Batch Operations
```go
// ✅ Correct: Use range queries
events := storage.GetEventRowsWithinRange(100)

// ❌ Wrong: Get all then slice
events := storage.GetAllEventRows()[:100]
```

### Memory Management
```go
// ✅ Correct: Delete processed events
ids := []int{1, 2, 3}
storage.DeleteEventRows(ids)

// ❌ Wrong: Let storage grow unbounded
// Never delete events
```

### String Building
```go
// ✅ Correct: Use bytes.Buffer
var buf bytes.Buffer
buf.WriteString(part1)
buf.WriteString(part2)
result := buf.String()

// ❌ Wrong: String concatenation
result := part1 + part2  // Creates intermediate strings
```

## Testing Patterns

### Storage Testing
```go
// ✅ Correct: Test both implementations
storages := []storageiface.Storage{
    memory.Init(),
    sqlite3.Init(":memory:"),
}
for _, s := range storages {
    // Run same tests
}

// ❌ Wrong: Test only one implementation
storage := memory.Init()
// Test only memory
```

### Payload Testing
```go
// ✅ Correct: Verify serialization
p := payload.Init()
p.Add("key", common.NewString("value"))
assert.Equal(`{"key":"value"}`, p.String())

// ❌ Wrong: Only check non-nil
assert.NotNil(p.String())
```

## Integration Patterns

### Custom Storage Implementation
```go
// ✅ Correct: Implement full interface
type CustomStorage struct{}

func (c *CustomStorage) AddEventRow(p payload.Payload) bool {
    // Implementation
}
// ... implement all methods

// ❌ Wrong: Partial implementation
type CustomStorage struct {
    storageiface.Storage  // Embed interface
}
// Missing method implementations
```

### Payload Extension
```go
// ✅ Correct: Use existing methods
func addCustomField(p *payload.Payload, value string) {
    p.Add("custom", common.NewString(value))
}

// ❌ Wrong: Modify internals
p.Pairs["custom"] = value  // Bypasses validation
```

## Quick Reference

### Package Imports
```go
import (
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/payload"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/common"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/storage/memory"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/storage/sqlite3"
    "github.com/snowplow/snowplow-golang-tracker/v3/pkg/storage/storageiface"
)
```

### Storage Comparison
| Feature | Memory | SQLite |
|---------|--------|---------|
| Speed | Fast | Moderate |
| Persistence | No | Yes |
| Memory Usage | High | Low |
| Concurrency | Excellent | Good |
| Use Case | High throughput | Reliability |

### Common Helpers
- `NewString()` - String pointer
- `NewInt64()` - Int64 pointer  
- `NewFloat64()` - Float64 pointer
- `GetUUID()` - Version 4 UUID
- `GetTimestamp()` - Unix ms timestamp
- `MapToJson()` - JSON serialization
- `MapToQueryParams()` - URL encoding