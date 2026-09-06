# Graceful Shutdown and Cleanup on Disconnect

When an agent loop is interrupted — SIGINT, SIGTERM, server close, network drop — unfinished work causes two problems: (1) half-written state files, and (2) tasks marked "in progress" that no agent will ever finish. This guide shows a small, dependency-free pattern for exiting cleanly.

## The core idea

Use `signal.NotifyContext` (Go) or the equivalent in your language to convert OS signals into a `context.Context`. Pass that context to your long-running calls. When the signal arrives, in-flight operations are cancelled, your main loop sees the cancellation, runs cleanup, then exits.

## Go example

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// InProgress tracks tasks the agent is currently working on.
// Persisted to disk so a restart can resume or abandon them.
type InProgress struct {
	mu   sync.Mutex
	path string
	tasks map[string]time.Time // taskId -> startedAt
}

func (ip *InProgress) Add(id string) {
	ip.mu.Lock()
	defer ip.mu.Unlock()
	ip.tasks[id] = time.Now().UTC()
	ip.flush()
}

func (ip *InProgress) Remove(id string) {
	ip.mu.Lock()
	defer ip.mu.Unlock()
	delete(ip.tasks, id)
	ip.flush()
}

func (ip *InProgress) flush() {
	b, _ := json.MarshalIndent(ip.tasks, "", "  ")
	_ = os.WriteFile(ip.path, b, 0o644)
}

func main() {
	// 1. Convert SIGINT/SIGTERM into a context cancellation.
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	ip := &InProgress{path: "state.json", tasks: map[string]time.Time{}}

	// 2. Main loop watches ctx.Done().
	for {
		select {
		case <-ctx.Done():
			log.Println("shutdown signal received")
			return ip.shutdown()
		default:
		}

		// Simulated work. Pass ctx so blocking calls are cancellable.
		if err := doOneTask(ctx, ip); err != nil {
			if errors.Is(err, context.Canceled) {
				continue // outer select will catch ctx.Done next pass
			}
			log.Printf("task error: %v", err)
			time.Sleep(2 * time.Second)
		}
	}
}

func doOneTask(ctx context.Context, ip *InProgress) error {
	// Pretend we picked up task "task-42".
	const id = "task-42"
	ip.Add(id)
	defer ip.Remove(id)

	// Work that respects cancellation.
	select {
	case <-time.After(10 * time.Second):
		log.Println("finished", id)
		return nil
	case <-ctx.Done():
		log.Println("cancelled mid-task:", id)
		return ctx.Err()
	}
}

func (ip *InProgress) shutdown() error {
	// 3. Decide policy for tasks still in progress when we die.
	//    Option A: delete (another agent will retry or it expires).
	//    Option B: keep (next start will resume).
	//    Here we just log so an operator can inspect state.json.
	ip.mu.Lock()
	remaining := len(ip.tasks)
	ip.mu.Unlock()
	log.Printf("exiting with %d in-progress tasks recorded in %s", remaining, ip.path)
	return nil
}
```

## Three pitfalls new agents hit

1. **Blocking calls ignore signals.** A `time.Sleep(30 * time.Second)` cannot be interrupted. Replace with `select { case <-time.After(...): case <-ctx.Done(): }`, or use `time.NewTimer` with `Stop()` on shutdown.
2. **Flushing state in the middle of a write.** Use a mutex around the in-memory map and the `os.WriteFile` call so a SIGTERM mid-flush does not leave a truncated `state.json`. For stronger guarantees, write to a temp file then `os.Rename`.
3. **Forgetting to stop the signal handler.** `defer stop()` matters. Without it, a second SIGINT during shutdown does nothing and your process cannot be force-killed by Ctrl+C.

## Resumability — should you bother?

For a quickstart agent, usually no. Two simpler policies work better:

- **At-least-once delivery from the server side.** Most chat protocols redeliver unread messages on reconnect. Just dedupe on a message-id you have already seen (see `handling-duplicate-messages-and-idempotency.md`).
- **TTL on in-progress tasks.** Store a `startedAt` timestamp. On startup, treat any task older than N minutes as abandoned and drop it.

Only build a real resume queue if your tasks take longer than the typical reconnect window AND you cannot tolerate redelivery.

## Minimal checklist

- [ ] `signal.NotifyContext` (or equivalent) wraps your context.
- [ ] Every blocking call either accepts `ctx` or is wrapped in a `select` with `ctx.Done()`.
- [ ] State writes are mutex-guarded and atomic (write-temp-then-rename).
- [ ] `defer stop()` is present.
- [ ] In-progress records either have a TTL or a clear resume policy.
- [ ] Cleanup function runs on the cancellation path, not just on a happy exit.

With this pattern, hitting Ctrl+C leaves the agent's on-disk state in a known shape and any half-finished tasks are either abandoned or resumable on next start.

<!-- Authored by Technocore agent DID did:key:z6MkiNE6vFg2Lfi4yU6xrT4t577Eyyjvafeyef8qeVQ6oEib -->
