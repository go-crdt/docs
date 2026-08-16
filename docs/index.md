# go-crdt

Two pure-Go modules for collaborative editing.

| Module | What it is |
|---|---|
| [`crdt`](repos/crdt.md) | the replicated text document — `Doc`, operations, version vectors, snapshots, presence |
| [`collab`](repos/collab.md) | the gRPC service, server and client that carry a document between people |

Both are **CGO-free**, held at **100% statement coverage**, and validated on all six
of Go's 64-bit targets.

## Why a CRDT

A conflict-free replicated data type settles concurrent edits by the operations
themselves, so nothing has to be authoritative. Three things follow that the
alternative — operational transform, as used by ShareDB and Google Docs — cannot
offer:

- **no server decides who won**, so there is no transform function to get right
  for every pair of operation types;
- **a participant can keep typing with the network down** and reconcile later;
- **the server can be restarted or replaced** without a handover protocol.

## Why in Go

Because the browser can then run it too. The engine compiles to `js/wasm`, so a
browser tab and the server execute the **same** merge implementation rather than
two implementations that have to agree. A JavaScript client paired with a Go
server cannot make that claim, and the acceptance test in `collab` exists to hold
this one to it: three replicas across two runtimes, converging.

## Start here

```go
ada, grace := crdt.New(1), crdt.New(2)

opening, _ := ada.Insert(0, "the quick fox")
grace.Apply(opening...)

// Both edit at once, neither having seen the other.
fromAda, _ := ada.Insert(10, "brown ")
fromGrace, _ := grace.Insert(13, " jumps")

ada.Apply(fromGrace...)
grace.Apply(fromAda...)

fmt.Println(ada)   // the quick brown fox jumps
fmt.Println(grace) // the quick brown fox jumps
```

Over a network, `collab` does the carrying:

```go
c, _ := collab.Join(ctx, conn, collab.ClientConfig{Document: "notes", Site: 1})
c.Insert(0, "hello")
for range c.Changes() {
    render(c.Text(), c.Peers())
}
```
