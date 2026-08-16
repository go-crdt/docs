# Contributing

## The bar

- **Pure Go, `CGO_ENABLED=0`.** The engine has no dependencies at all.
- **100% statement coverage**, error branches included. It is a CI gate, not a
  goal. A branch no input can reach is a design smell — restructure until the
  branch is either reachable or gone.
- **Six architectures.** amd64 and arm64 natively; riscv64, loong64, ppc64le and
  s390x under qemu. s390x is big-endian, which is what keeps the deterministic
  encodings honest.
- **Every decoder is fuzzed.** Anything read from a peer is hostile input until
  proven otherwise.

## Proving things

A green test run is not evidence that a test is doing its job. Before trusting a
new assertion, break what it is meant to catch and confirm it goes red — the
ordering rules in `crdt` are pinned by tests validated exactly that way, and one
of them was written only after an experiment showed the existing suite could not
tell the difference.

A skipped test is not a passing one. CI sets `COLLAB_REQUIRE_WASM` on the job
whose whole purpose is the WebAssembly end-to-end run, so a missing toolchain
fails it rather than turning it green.

## Running everything

```sh
go test ./...                 # unit, property and end-to-end
go test -race ./...
go test -coverprofile=c.out ./... && go tool cover -func=c.out | tail -1

# as WebAssembly, the way a browser will run it
GOOS=js GOARCH=wasm go test -exec="$(go env GOROOT)/lib/wasm/go_js_wasm_exec" ./...

# the fuzzers
go test -run '^$' -fuzz FuzzLoad -fuzztime 120s
```

## Licence

BSD-3-Clause. Copyright the go-crdt authors.
