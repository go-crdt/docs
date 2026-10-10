# How convergence is proven

A text CRDT is worth nothing unless replicas that saw the same operations agree.
That claim is easy to assert and easy to believe wrongly, so this is how it is
established here — and, just as usefully, what each layer of testing does *not*
establish.

## Randomised sessions sample; they do not cover

Three hundred randomised sessions run replicas editing concurrently while the
network delivers late, out of order, and with duplicates. Once delivery catches
up, every replica must hold the same document.

This is necessary and it is not sufficient: it samples the space of orderings
rather than covering it. The counterexamples in the CRDT literature are four to
eight operations long, which is precisely the range random schedules explore
thinly.

## So every ordering is also tried

For small concurrent histories — replicas editing blind, exchanging everything,
then editing blind again — **every permutation** of the resulting operations is
applied to a fresh replica, and all must produce byte-identical state. Forty such
histories, exhaustively.

## Replicas are compared on their state, not their text

Two replicas agreeing on the text is the weaker claim: they can display the same
characters while disagreeing about the identities underneath, and diverge on the
next edit. So the assertions compare **encoded snapshots**, which are canonical —
the same operations always produce the same bytes, on every architecture,
big-endian included.

## The instruments are checked against known breakage

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

## Deleting every refusal, not the ones somebody thought to try

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

Seven runs, the last finished 2026-10-09:

| | subjects | not mutants | caught | survived | **held** |
| --- | --- | --- | --- | --- | --- |
| `collab`, the wire | 126 | 73 | 43 | 10 | **3** |
| `crdt`, operations and snapshots | 104 | 29 | 63 | 12 | 0 |
| `crdt/structured`, the decoders | 38 | 19 | 14 | 5 | **1** |
| `crdt/structured`, the rest | 242 | 101 | 108 | 33 | **4** |
| `collab`, the stores | 64 | 20 | 38 | 6 | **1** |
| `crdt`, the core data structures | 204 | 61 | 115 | 27 *(+1 hung)* | **13** |
| `crdt`, everything else in the root | 32 | 6 | 20 | 6 | **1** |
| | **810** | **309** | **401** | **99** *(+1)* | **23** |

The last row is there because the first six left a hole worth naming. They were
each pointed at a set of files, and after six runs nine of the twenty sources in
`crdt`'s root had never been in any of them. A campaign total is a number about
whatever it was pointed at, so the question "what has not been swept?" has to be
asked of the directory rather than of the total. With that row the root is
complete.

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

## The nine, from the first five runs

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

## And the fifty-seven of those that were not

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
which is affordable because there are ninety-nine of them.

## The sixth run, where half the survivors were real

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

## Answering a report

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

## The same defect twice, two files apart

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

One caution, learned the hard way: a **duration** taken during a campaign
measures the campaign. One mutant appeared to leave the suite four times slower,
which read as tests waiting on deadlines; measured afterwards in pairs it was
0.86x and 0.98x, and the four-fold was a loaded machine. The verdict — survived
or killed — does not care how loaded the machine is.

## Hostile input is assumed

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

## The end-to-end gate

Compiling for `js/wasm` proves nothing on its own — plenty of code compiles and
then fails to run. So the acceptance test runs **three replicas across two
runtimes**: a native participant and two compiled to WebAssembly and executed by
Node, editing one document concurrently through a real `grpc.Server` over a real
WebSocket. All three must converge on the same text with no character lost.

A skipped test is not a passing one, so the CI job that exists to run this sets
`COLLAB_REQUIRE_WASM`: a missing Node or wasm glue fails it rather than turning it
green. The emulated-architecture jobs skip it explicitly instead, because running
Node under qemu proves nothing about this code.
