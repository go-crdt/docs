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

Five runs, finished 2026-10-07:

| | subjects | not mutants | caught | survived | **real** |
| --- | --- | --- | --- | --- | --- |
| `collab`, the wire | 126 | 73 | 43 | 10 | **3** |
| `crdt`, operations and snapshots | 104 | 29 | 63 | 12 | 0 |
| `crdt/structured`, the decoders | 38 | 19 | 14 | 5 | **1** |
| `crdt/structured`, the rest | 242 | 101 | 108 | 33 | **4** |
| `collab`, the stores | 64 | 20 | 38 | 6 | **1** |
| | **574** | **242** | **266** | **66** | **9** |

The second column is not a result about the tests. Deleting
`if err != nil { return err }` orphans the `err` the line above declared and the
package stops building; a run that counts those as killed, or as survived, is
wrong either way, so they are a third verdict and nearly half of everything
tried.

There is a fourth, which these five runs did not need and a later one did: a
mutant can **hang** rather than fail. Deleting the check that a retry policy is
honourable does not make a suite red, it makes it wait out an hour somebody
mistyped into the wrong field — the guard is load-bearing *and* nothing says so.
Four of the five verdicts are PIT's under other names (*Killed*, *Survived*,
*Timed Out*, *Non viable*, *Run error*), which is some evidence they are the
joints of the thing rather than one tool's habits.

## The nine

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

## And the fifty-seven that were not

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
which is affordable because there are sixty-six of them.

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
