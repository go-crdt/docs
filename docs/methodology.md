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
