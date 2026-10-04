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

### Systematically, over every refusal

Doing that by hand finds what you thought to try. On 2026-10-04 it was done
mechanically instead, over every refusal in the files that read bytes somebody
else wrote: walk the source for an `if` whose body returns a refusal, delete
one, build, run the suite, restore, repeat.

| | `collab` | `crdt` |
| --- | --- | --- |
| refusals and bounds taken as subjects | 126 | 104 |
| deletions that did not compile — **not mutants** | 73 | 29 |
| caught by the suite | 43 | 63 |
| survived, at 100% of statements | 10 | 12 |
| of those, **real** | **3** | 0 |

The first row of numbers is not a result about the tests. Deleting
`if err != nil { return err }` orphans the `err` the line above declared and
the package stops building; a run that counts those as killed, or as survived,
is wrong either way, so they are a third verdict.

The three real ones were the same mistake three times: **a test asserting that
an error happened**, where the code after the deleted guard also fails and says
something else. The clearest is a client that hangs up before sending anything,
which was handed `InvalidArgument` and "a session must open with a join" — for
something it never did — where its own stream's `EOF` is the truth. All three
are pinned now, each by naming the error its case is about rather than its
existence.

The nineteen that survived and were not real are worth naming rather than
carrying as a worry. Counted by why each one survives:

| | |
| --- | --- |
| a bound doubled one layer down | 9 |
| a fast path — "no character here is more than one UTF-16 code unit, so the offset is the offset" | 6 |
| only saves work: an empty batch not sent, a file not rewritten when nothing changed, one more `Recv` that returns the same error | 3 |
| the same value by another line: `append([]byte(nil))` of nothing is nil too | 1 |

The nine are what defence in depth looks like from a single-layer mutation: a
kind check in a decoder whose own last line validates the operation anyway, a
negative length refused again by the range check below it, a snapshot bound
standing in front of the accounting that every promised operation appears
exactly once, a lookup before a slow load that a second lookup under the lock
repeats afterwards.

Three of those were measured rather than argued: the same `ErrOutOfRange`
either way for four negative lengths; all 102 truncations of a real snapshot
refused with a column's empty-check and without it; and 27.7 million fuzz
executions against one mutant finding nothing, because that fuzz target asks
whether a load panics and not whether it accepted what it should have refused.

The proportion is not a surprise. At Google, over almost seventeen million
mutants, developers initially judged 85% of what was reported to them
unproductive, and rules for suppressing those are what made the technique
usable at all (Petrović & Ivanković, *Practical Mutation Testing at Scale: A
View From Google*, IEEE TSE, 2021). Here the filtering is a reading of each
survivor, which is affordable because there are twenty-two of them.

One caution, learned the hard way the same evening: a **duration** taken during
a campaign measures the campaign. One mutant appeared to leave the suite four
times slower, which read as tests waiting on deadlines; measured afterwards in
pairs it was 0.86x and 0.98x, and the four-fold was a loaded machine. The
verdict — survived or killed — does not care how loaded the machine is.

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
