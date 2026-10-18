---
title: "How convergence is proven"
weight: 20
---

# How convergence is proven {#how-convergence-is-proven}

A text CRDT is worth nothing unless replicas that saw the same operations agree.
That claim is easy to assert and easy to believe wrongly, so this is how it is
established here — and, just as usefully, what each layer of testing does *not*
establish.

## Randomised sessions sample; they do not cover {#randomised-sessions-sample-they-do-not-cover}

Three hundred randomised sessions run replicas editing concurrently while the
network delivers late, out of order, and with duplicates. Once delivery catches
up, every replica must hold the same document.

This is necessary and it is not sufficient: it samples the space of orderings
rather than covering it. The counterexamples in the CRDT literature are four to
eight operations long, which is precisely the range random schedules explore
thinly.

## So every ordering is also tried {#so-every-ordering-is-also-tried}

For small concurrent histories — replicas editing blind, exchanging everything,
then editing blind again — **every permutation** of the resulting operations is
applied to a fresh replica, and all must produce byte-identical state. Forty such
histories, exhaustively.

## Replicas are compared on their state, not their text {#replicas-are-compared-on-their-state-not-their-text}

Two replicas agreeing on the text is the weaker claim: they can display the same
characters while disagreeing about the identities underneath, and diverge on the
next edit. So the assertions compare **encoded snapshots**, which are canonical —
the same operations always produce the same bytes, on every architecture,
big-endian included.

## The instruments are checked against known breakage {#the-instruments-are-checked-against-known-breakage}

A green test tells you nothing about whether the test can fail. Each ordering rule
was therefore disabled deliberately, one at a time, to confirm the suite goes red.

That exercise paid for itself immediately. Disabling the Lamport clock comparison
entirely — leaving only the site identifier to break ties — **still passed
everything**: 300 randomised sessions and every permutation of forty histories, at
four sites and eight operations. The suite could not see the difference.

The finding is genuine, and worth stating precisely: with causal readiness
enforced per site, *convergence* does not depend on the clock. What the clock
decides is **which** order concurrent insertions take, and that is visible to a
user — whoever had seen more of the document when they typed is placed first,
rather than whoever happens to hold the smaller identifier. A test was written to
pin exactly that, and it fails without the clock.

## Deleting every refusal, not the ones somebody thought to try {#deleting-every-refusal-not-the-ones-somebody-thought-to-try}

Doing that by hand finds what you thought to try. Since 2026-10-04 it is done
mechanically instead, over every refusal in the code that reads bytes somebody
else wrote: walk the source for an `if` whose body returns a refusal, delete
one, build, run the suite, restore, repeat.

It is a tool rather than a description, so the table below can be checked
rather than believed:

```sh
go install github.com/go-fleettools/mutate@latest          # one mutation
go install github.com/go-fleettools/mutate/mutsweep@latest # every refusal in a package

mutsweep -dir . -files op.go,snapshot.go,utf16.go -- go test ./...
```

Nine runs, the last finished 2026-10-11:

| | subjects | not mutants | caught | survived | **held** |
| --- | --- | --- | --- | --- | --- |
| `collab`, the wire | 126 | 73 | 43 | 10 | **3** |
| `crdt`, operations and snapshots | 104 | 29 | 63 | 12 | 0 |
| `crdt/structured`, the decoders | 38 | 19 | 14 | 5 | **1** |
| `crdt/structured`, the rest | 242 | 101 | 108 | 33 | **4** |
| `collab`, the stores | 64 | 20 | 38 | 6 | **1** |
| `crdt`, the core data structures | 204 | 61 | 115 | 27 *(+1 hung)* | **13** |
| `crdt`, everything else in the root | 32 | 6 | 20 | 6 | **1** |
| `collab`, the browser half | 41 | 13 | 8 | 20 | **10** |
| `crdt/awareness` | 14 | 2 | 11 | 1 | **1** |
| | **865** | **324** | **420** | **120** *(+1)* | **34** |

The last three rows are there because the first six left holes, and the holes
were not all of one kind. The same gap turned up **four times in three shapes**,
which is what makes it a fault in the *record* rather than in anybody's
attention:

| shape | what was missing |
| --- | --- |
| files left out of a directory that *was* swept | nine of the twenty sources in `crdt`'s root, after **six** runs |
| files the command does not compile | the five `*_js.go` in `collab` — 41 refusals |
| a **subpackage** never pointed at | `crdt/awareness`, 14 refusals, no row at all |
| what is left over, unidentifiable | `collab`'s root holds 245 native refusals against 190 in the rows: **55** whose names the rows cannot give back |

The last line is the one worth dwelling on. Rows that name a *theme* —
"`collab`, the wire" — cannot say which files they covered, so the 55 can only be
found by sweeping all 245 again. Naming files costs nothing at the time and is
the difference between a gap you can see and a gap you cannot.

**And the check costs a second per directory.** `mutsweep -dir X -- true` prints
its refusal count *before running anything*, so the count can be compared with
what the rows claim:

```sh
$ mutsweep -dir . -- true          # crdt's root
mutsweep: 340 refusals, ...        # 104 + 204 + 32 ✓
$ mutsweep -dir structured -- true
mutsweep: 280 refusals, ...        # 38 + 242 ✓
$ mutsweep -dir awareness -- true
mutsweep: 14 refusals, ...         # nothing ✗
```

Which is how the last row was found, and it has to be run for every subpackage
rather than for the one being worked in.

### The same check, not run — and what came of that {#the-same-check-not-run-and-what-came-of-that}

Six days later the 55 unnamed refusals were gone after for real: `collab`'s root
was re-swept **by name**, in three chunks of 55, 61 and 32, and the result was
published as

> That closes the root package: 148 refusals over three chunks.

The package holds **245**. Ninety-seven refusals in eight files had been in no
chunk at all — `dirstore.go` 25, `client.go` 21, `bcast.go` 16, `capability.go`
12, `dircompress.go` 8, `authorize.go` 7, `durability.go` 6, `backoff.go` 2.

The check above is exactly the one that catches it, it is written on this page
three paragraphs up, and it was not run. Nothing in the act of adding three
numbers together asks the package anything, and a Σ of chunks looks like a
measurement — it has units and a total and it is reproducible. It is not one.

**So the check became a flag, printing what a plan is made of.** `mutsweep
-tally` (v0.6.0) answers *which file*, not merely *how many*, runs nothing, and
needs no command:

```console
$ mutsweep -dir . -tally
mutsweep: not swept, the darwin/arm64 build excludes them: bcast_js.go, ...
245 refusals in 22 file(s) of .
     31  wire.go
     25  dirstore.go
     24  server.go
     21  client.go
     17  follow.go
     ...
      2  backoff.go
```

With `-files` it counts what a chunk *will* sweep, so the comparison is made
before the hour is spent instead of after the claim is published. The point is
not the convenience: a check that answers a question **next to** the one being
asked — "how many" when the sentence says "which" — does not protect anybody,
and this page is the evidence.

This is where the shape of the campaign matters. The Google system cited further
down this page mutates **one node per line** of a code review's **diff**, after
excluding what it calls *arid* nodes — code whose mutants are unproductive, log
statements being the example given — and surfaces the survivors as findings in
review. A diff-shaped campaign never has to be chunked, so it never has to be
accounted for. A whole-package campaign does, and that accounting is the part no
instrument here was doing.

### `collab`'s root, finished and named — 2026-10-18 {#collabs-root-finished-and-named-2026-10-18}

Five chunks, every file named, and the total compared with the package rather
than with itself:

| chunk | files | subjects | not mutants | caught | survived | **held** | named |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | `wire.go`, `server.go` | 55 | 24 | 27 | 4 | **2** | 2 |
| 2 | `follow.go`, `sites.go`, `rejoin.go`, `grpc.go`, `stable.go`, `multistore.go` | 61 | 28 | 30 | 3 | **1** | 2 |
| 3 | `reconnect.go`, `parts.go`, `tiered.go`, `serve_websocket.go`, `websocket.go`, `store.go` | 32 | 22 | 7 | 3 | **3** | 0 |
| 4 | `client.go`, `capability.go`, `bcast.go`, `authorize.go`, `backoff.go` | 58 | 30 | 22 | 6 | **3** | 3 |
| 5 | `dirstore.go`, `dircompress.go`, `durability.go` | 39 | 11 | 25 | 3 | **2** | 1 |
| | | **245** | **115** | **111** | **19** | **11** | **8** |

245 is `mutsweep -dir . -tally`'s answer for the package, not the sum of the
rows — the two agree, which is the whole point of the section above. The five
`*_js.go` files are not in it: the darwin build excludes them, the tool says so
by name, and they are the 41 of the browser-half row further down.

⛔ **This is not additive to the 865.** It re-sweeps ground the "collab, the
wire" and "collab, the stores" rows covered under theme names, and that is why
it exists: those rows could not say which files they had read.

Three of the eleven are worth naming here, because each one looked like
something cheaper than it was:

| guard | what it looked like | what deleting it does |
| --- | --- | --- |
| `bcast.go:186`, `if n <= 0` | a check for a truncated header | **panics**. `binary.Uvarint` returns a *negative* count for an overflowing varint, and `rest[n:]` is then `slice bounds out of range [-11:]`. On the BroadcastChannel bus, so the bytes come from another **tab** |
| `dirstore.go:322`, the unpack error in `Release` | a lost error message | the error becomes `ErrChanged`, the one `Tiered.Archive` treats as **benign** — so a corrupt file is skipped in silence by the only code that knew |
| `client.go:518`, `publish`'s empty batch list | an empty frame nobody minds | **every join** pays a whole snapshot write: `pushMissing` calls it with what a fresh participant has, which its own comment says is nothing |

And the eight named-not-pinned were each *measured* rather than argued, because
two of them had been read wrong first. `client.go:285`'s fast path would have
been load-bearing if `LoadComposite` answered `ErrUnknownFormat` for an empty
snapshot — it answers `malformed encoding`, so the guard is equivalent.
`stable.go:263` was first read as a floor of "nothing" returned as a floor that
exists, which would have let a collection run below what anybody acknowledged;
the function's last guard already returns the same `(nil, false)`.

The eighth filled a hole that could not have been filled by pointing at it. The
browser half of `collab` — the WebRTC peer, the BroadcastChannel bus, the Web
Locks election, the browser's WebSocket — is behind `//go:build js && wasm`, and
a native `go test` does not **compile** it. Sweep those files with a native
command and every refusal in them comes back `SURVIVED — nothing covers this`,
when the truth is that nothing *looked*: one deleted by hand reported exactly
that, after ninety-three seconds of running a suite that never read the file.
The tool refuses that now, and the sweep was run for real with the platform set
on both sides of it:

```sh
GOOS=js GOARCH=wasm mutsweep -files bcast_js.go,bcast_lock_js.go,peer_js.go,webrtc_js.go,websocket_js.go \
  -- go test -count 1 -exec "$(go env GOROOT)/lib/wasm/go_js_wasm_exec" .
```

Two minutes, because the js/wasm test set is 74 tests and runs in seconds. It is
the highest yield of the campaign — **ten of twenty survivors held** — which is
what an unswept surface looks like rather than a statement about that code.
What it found is in [the browser half](#the-browser-half-of-a-pure-go-engine)
below.

This hole has a name outside Go, and it is older than Go. A build tag is a
*configuration*, and an analysis run in one configuration says nothing about the
others — which is the whole subject of variability-aware (family-based) analysis,
against the single-configuration or sampled kind. The scale of it in real code is
measured: over forty C product lines, variability is not a rare decoration but
pervasive and fine-grained, often reaching inside single expressions (Liebig,
Apel, Lengauer, Kästner and Schulze, *An Analysis of the Variability in Forty
Preprocessor-Based Software Product Lines*, ICSE 2010). And analysing a family
rather than a configuration is expensive enough to be a research problem of its
own, on Busybox, the Linux kernel and OpenSSL (Liebig, von Rhein, Kästner, Apel,
Dörre and Lengauer, *Scalable Analysis of Variable Software*, ESEC/FSE 2013).

Go makes this smaller and sharper than C does — a handful of named tags rather
than thousands of interacting macros — but not absent, and the sharp part is that
nothing warns you. A sweep of an excluded file does not fail; it reports a clean
file. So the rule here is not "be variability-aware", which this campaign is not:
it is that the *list of configurations swept* is part of the result, and a tool
that cannot say which one it was in should not be trusted to say a file is held.

The second column is not a result about the tests. Deleting
`if err != nil { return err }` orphans the `err` the line above declared and the
package stops building; a run that counts those as killed, or as survived, is
wrong either way, so they are a third verdict and nearly half of everything
tried.

There is a fourth, which the first five runs did not need and the sixth did: a
mutant can **hang** rather than fail. Deleting the check that a retry policy is
honourable does not make a suite red, it makes it wait out an hour somebody
mistyped into the wrong field — the guard is load-bearing *and* nothing says so.
Four of the five verdicts are PIT's under other names (*Killed*, *Survived*,
*Timed Out*, *Non viable*, *Run error*), which is some evidence they are the
joints of the thing rather than one tool's habits.

## The nine, from the first five runs {#the-nine-from-the-first-five-runs}

Four of them are one mistake: **a test asserting that an error happened**, where
the code after the deleted guard also fails and says something else. The
clearest is a client that hangs up before sending anything, which was handed
`InvalidArgument` and "a session must open with a join" — for something it never
did — where its own stream's `EOF` is the truth.

The other five are each their own shape, and three are worth stating plainly.

**A manifest that claims bytes it has no chunks for.** Ten bytes saying *one
gibibyte, no chunks* read as a file: `Size` reports a gibibyte and says that is
true, `Missing` reports nothing missing because there are no keys to miss, and
`Get` hands back nothing. A gibibyte that nothing waits for and that never
arrives. The shape has a name — Sassaman, Patterson, Bratus, Locasto and
Shubina call it a *parse tree differential* (Dartmouth TR2011-709, 2011), two
readers taking different meanings from one input — and it is the argument for
where a bound lives: a recognizer that lets through an input its consumers
cannot agree on has moved the decision into each caller.

**A varint count used as an index.** `binary.Uvarint` returns a NEGATIVE count
for an encoding that overflows sixty-four bits, and the line after one of these
is usually `buf[n:]`, which panics rather than erring. Twelve bytes of
continuation bits, carried in a peer's ink operation, gave
`slice bounds out of range [-11:]` with one check deleted. The case is pinned,
and so is the class: every `binary.Uvarint` and `binary.Varint` in non-test
source must be followed by an `if` testing its count — twenty of them, checked
by a test that walks the syntax tree.

**Off that did not mean off.** `CollectEvery` is off by default, and collecting
drops map tombstones. One line holds it, no test could see that line — every
test that collects sets the interval, and the helper written so a test need not
wait for a timer sets the interval ITSELF — and with the line deleted a server
with collection off gave back thirty versions' worth of tombstones.

## And the fifty-seven of those that were not {#and-the-fifty-seven-of-those-that-were-not}

Every one was read. They fall into shapes:

| | |
| --- | --- |
| a varint count that will not decode, refused again below | 12 |
| an error from a call whose next sibling fails the same way | 11 |
| an empty input or collection | 8 |
| a decode that did not succeed, refused again below | 8 |
| a fast path — the general path gives the same answer | 6 |
| read one by one: three save work, seven are doubled, one is a short-circuit, one is an ordering rule deterministic either way | 12 |

The large families are what defence in depth looks like from a single-layer
mutation, and three of the readings were measured rather than argued: the same
`ErrOutOfRange` either way for four negative lengths; all 102 truncations of a
real snapshot refused with a column's empty-check and without it; and
`RichText.MarksAt` returning no marks for −1, past the end and 2²⁰ with its
bound removed, because what it calls is bounded itself.

The proportion is not a surprise. At Google, over almost seventeen million
mutants, developers initially judged 85% of what was reported to them
unproductive, and rules for suppressing those are what made the technique usable
at all (Petrović & Ivanković, *Practical Mutation Testing at Scale: A View From
Google*, IEEE TSE, 2021). Here the filtering is a reading of each survivor,
which is affordable because there are a hundred and twenty of them.

## The sixth run, where half the survivors were real {#the-sixth-run-where-half-the-survivors-were-real}

The first five runs swept the wire, the stores and the structured layer, and
yielded nine. The sixth swept what everything else is built on — the composite,
the list, the map, the text and the version vector — and yielded thirteen from
twenty-seven survivors. The jump is not a change of method. It is what those
files are: every one of them reads bytes a peer wrote, and a count read from a
peer is a claim rather than a fact.

The shape that produced most of them is the one no assertion about a **result**
can reach. These are the same answer either way:

```go
nSites, ok := r.uvarint()
if !ok || nSites > uint64(len(r.buf)) {   // delete this
    return ErrMalformed                   // and the loop below still returns it
}
sites := make([]SiteID, 0, nSites)        // having sized itself from the claim
```

The loop underneath runs out of bytes and refuses the input exactly as before.
What changes is only the allocation, so the assertion has to be about the
allocation — and the numbers are why it is worth making:

| input | claims | allocated with the bound gone |
| --- | --- | --- |
| 4 bytes | 16 777 216 sites | 134 217 728 bytes |
| 6 bytes | 16 777 216 parts | **1 343 536 056 bytes** |
| 10 bytes | 16 777 216 entries | 605 296 040 bytes |
| 4 bytes | 16 777 216 vector entries | 605 306 728 bytes |

Six bytes for 1.3 GB is an amplification of two hundred million, and a version
vector is the first thing a peer sends. The bounds were already right; nothing
would have noticed if they left.

The class has a name — MITRE call it *Memory Allocation with Excessive Size
Value* ([CWE-789](https://cwe.mitre.org/data/definitions/789.html)) — and it is
not a curiosity of this codebase. On the morning after this sweep finished, Go
published thirteen standard library advisories, and three of them are the same
shape:

- *Lack of limit on size of parsed Range headers* in `net/http` (GO-2026-6609);
- *Memory limit bypass when parsing MIME headers* in `net/textproto` and
  `mime/multipart` (GO-2026-6608);
- *HTTP/2 server memory exhaustion due to Trailer headers* in `net/http`
  (GO-2026-6603).

A size a peer chose, believed far enough to reserve for. That is also why the
fix here is a bound and not a larger buffer: the recognizer has to refuse the
claim before anything downstream acts on it, which is the same argument as the
parse tree differential above, applied to a resource rather than to a meaning.

The run also produced the fourth verdict for the first time. Deleting the line
that decides whether `Map.wake` walks what is **parked** or walks the **range an
operation claims** does not fail the suite — a superseded run may name any
sequence number up to 2⁶², so the suite simply waits, and the sweep gave up after
four minutes. The test that now holds it gives the call thirty seconds, which is
a damage bound rather than a performance claim: it returns in microseconds or
walks 2⁶² numbers, and nothing lands in between.

Two more were about *which* error a peer is told. An operation whose kind this
build does not know is `ErrInvalidOp` — a dialect — and with the check deleted a
short one becomes `ErrMalformed`, a damaged transport. A batch truncated in its
part name is `ErrMalformed`, and with its check deleted becomes `ErrInvalidPart`.
One says send this again; the other says do not. They are mapped to different
gRPC codes, so the distinction leaves the process.

And one was a false accusation rather than a missing refusal. `collides` answers
"two replicas minted the same identity for different characters", and `admit`
turns a yes into an error. With its gate deleted a *superseded* run is carried
into that comparison, where its ID names a character the receiver holds and its
character field is the zero value no character has — so a peer resending a run it
no longer holds would be told its identity clashes and lose its session.

Fifteen of the twenty-seven were left unheld on purpose, each with the reason
written beside it in the test file and measured rather than argued: six are
mutually covering pairs, five reach the same value by a longer route, and four
are refused again one layer down. One of those readings needed a second channel
to settle — a bound of exactly the shape in the table above, which the suite does
not notice and which turns out to size nothing: 14 bytes claiming 16 777 216
elements cost 608 bytes of allocation with it and 608 without.

## Answering a report {#answering-a-report}

Writing a test for a survivor leaves a question the test cannot answer about
itself: is it green because it holds the guard, or green because it never reaches
it? Re-running the whole sweep to find out costs half an hour, which is why the
survivors go back through the tool by name:

```sh
mutsweep -only map.go:163,map.go:628,map.go:856 -- go test -count 1 ./...
```

A target that matches no refusal is an error naming the target, because the two
ways such a list goes stale — the file edited since the report, the line mistyped
— both otherwise end as a sweep of nothing, which is the one result that reads
like a clean one.

The sixth run's twenty-eight were put back through it once the tests had landed,
against the merged branch: **twelve caught**, which is exactly the twelve the
tests were written for, and the fifteen named as unheld still survive. A
prediction confirming itself is worth more than a prediction. The hung one is
still hung, and that is not a gap in its test — the command a sweep runs is the
*whole* suite, and other tests in the package wait it out too; the test written
for it fails in thirty seconds.

## The same defect twice, two files apart {#the-same-defect-twice-two-files-apart}

The seventh run found, in `Map.resurrects`, the defect the sixth had found in
`Doc.collides`. Both ask something of **every operation that arrives twice**, and
both were carrying a *superseded run* — an operation that stands in for what its
sender no longer holds — into a comparison that has no business reading it.

A superseded run carries a sequence range and nothing else. Every other field is
a zero, and a zero looks like a value:

| | the comparison | with the refusal deleted |
| --- | --- | --- |
| `Doc.collides` | two replicas minted one identity for **different characters** | the run's ID names a character the receiver holds, its character field is the zero no character has → `ErrCollidingID`, session torn down |
| `Map.resurrects` | this write would bring back a key whose tombstone we dropped | the run names **no** key, `records[""]` is absent → read as a key not held → `ErrStranded`, the whole batch refused |

Both times, a peer **resending** a run it no longer holds is accused. Both times
the refusal that prevented it was held by nothing. A guard of the form
`if op.Kind != X { return false }` at the top of a predicate is not a shortcut —
it is the predicate's *scope*, and without it the predicate answers about an
object it knows nothing about by reading zeros.

## The browser half of a pure Go engine {#the-browser-half-of-a-pure-go-engine}

The point of writing this in Go is that a tab and a server run the *same* merge
code. The transports underneath them are not the same: a tab reaches another tab
over a BroadcastChannel, reaches a peer over WebRTC, reaches a server over the
browser's own WebSocket, and decides which tab hosts with the Web Locks API.
That is four browser APIs, and every one of them can be absent.

Ten refusals were held there. Three are a shape this page has already named —
**a test asserting that *an* error happened**:

- `Peer.Answer`'s role gate. A test for it existed and covered three of the four
  ways a page can take two roles. Its answering case passed the literal
  `"anything"`, which is not a connection description, so with the gate deleted
  `Answer` carried on and the *decoder* refused it instead — and both wrap
  `ErrTransport`, so neither the test nor `errors.Is` could tell them apart. The
  replacement hands it a **valid** offer, which leaves the gate as the only thing
  that can refuse it, and adds the fourth combination: answering twice.
- The check after `createAnswer`. The fixture could make `createOffer` reject and
  had nothing for `createAnswer`, so of a symmetric pair the answer half was the
  one no test could reach. That is why it survived and its twin did not.
- The check after `dialBus`, which returns `(nil, nil)` without it — a caller told
  nothing went wrong, holding nothing, with the bus left open, since that branch
  is also what closes it.

Four are **feature detection**, and they are what an old browser gets instead of a
stack trace in a console nobody is reading: no `BroadcastChannel` (both doors into
it), no `WebSocket`, and no data channel. One more is the **Web Locks** detection,
which is load-bearing in a way worth stating: the lock is the *rule* for deciding
which tab hosts, and the timing window is the *fallback*, because a lock answers
immediately and exclusively while a window can leave two tabs hosting on a loaded
machine. The detection is what chooses between them.

And one is a guard that is **unreachable today and pinned anyway**, because of what
the measurement says happens without it. `stalled` decorates a transport error with
the connection's three states; both of its callers pass `ctx.Err()` from inside a
`case <-ctx.Done()`, so neither can hand it nil. Delete its nil check and

```
stalled(nil) = %!w(<nil>) (connection new, ice new, gathering new)
```

a non-nil error built out of a nil one, with a formatting artefact for a message.
Any future caller that decorated unconditionally would turn every success into a
failure, and the failure would say nothing about why.

The other ten survivors are named in the test files with the measurement beside
each: two pairs where the same state is read before and after a listener is wired
(the second of each carries the behaviour, and its window needs the browser's own
timing), three "already closed" guards that are cost because `js.Func.Release` and
a select on a closed channel are both idempotent, one cosmetic (`syscall/js`
renders a non-string as `<object>` rather than panicking), one reachable only from
a value nothing constructs, and one still blocked on a fixture — `OpenBroadcastSession`
*succeeds* on a context that is already over, because a tab alone in a room becomes
its host immediately and the election never waits for anything.

## One clause of four, in a package nobody had pointed at {#one-clause-of-four-in-a-package-nobody-had-pointed-at}

`crdt/awareness` carries presence — who is here, where their cursor is — which is
the busiest thing on this wire, and its fourteen refusals had been in no run. One
survived:

```go
site, ok1 := d.uvarint()
clock, ok2 := d.uvarint()
gone, ok3 := d.byteValue()
if !ok1 || !ok2 || !ok3 || gone > 1 {   // ← deleted
    return ErrMalformed
}
out := Update{..., Gone: gone == 1}
```

What the `gone > 1` clause keeps is the encoding being **canonical** — the same
state being the same bytes — which is what lets snapshots here be compared by
their bytes, and which every other decoder in `crdt` insists on. Without it,
`gone == 1` reads 2, 7 and 255 all as *false*, so one present participant can be
written **255 different ways**. Measured on a real update:

| gone byte | with the clause | without it |
| --- | --- | --- |
| 0 | accepted, re-encodes to the same bytes | same |
| 2, 7, 255 | refused | accepted, re-encodes to **different** bytes |

Which of the four clauses carries it was measured too, because that line is four
decisions in one: dropping only `gone > 1` turns the new test red, and dropping
only the three `!okN` checks leaves the whole suite green — a site, a clock or a
byte that did not decode leaves a buffer the next read runs out of. The clause
about what the byte *means* is the one nothing else says.

One caution, learned the hard way: a **duration** taken during a campaign
measures the campaign. One mutant appeared to leave the suite four times slower,
which read as tests waiting on deadlines; measured afterwards in pairs it was
0.86x and 0.98x, and the four-fold was a loaded machine. The verdict — survived
or killed — does not care how loaded the machine is.

## Hostile input is assumed {#hostile-input-is-assumed}

Everything a replica receives comes off a network, so every decoder is a trust
boundary and every decoder is fuzzed. This found **five** states a snapshot could
describe that no replica could ever reach, each of which left a document unable to
reproduce its own history:

- a sequence number of zero paired with a non-zero site, read as a real operation
  instead of the document root;
- two characters claiming the same deletion;
- a version vector promising more operations than the snapshot accounts for;
- a document order that integration could not have produced;
- a concurrent deletion aimed at the root sentinel, or at a character still
  visible.

The inputs that found them are committed, so they run on every build.

## The end-to-end gate {#the-end-to-end-gate}

Compiling for `js/wasm` proves nothing on its own — plenty of code compiles and
then fails to run. So the acceptance test runs **three replicas across two
runtimes**: a native participant and two compiled to WebAssembly and executed by
Node, editing one document concurrently through a real `grpc.Server` over a real
WebSocket. All three must converge on the same text with no character lost.

A skipped test is not a passing one, so the CI job that exists to run this sets
`COLLAB_REQUIRE_WASM`: a missing Node or wasm glue fails it rather than turning it
green. The emulated-architecture jobs skip it explicitly instead, because running
Node under qemu proves nothing about this code.
