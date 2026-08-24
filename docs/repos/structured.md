# structured — the documents

`crdt/structured` is where documents are built, and the first thing to say about
it is that it re-implements nothing.

There are three merging structures underneath — a text, a list, a map — and
every type here is a way of using them. Convergence, commutativity, idempotence
and associativity are proved once, of those, and inherited by everything
composed from them. The package's own acceptance gate runs those four laws plus
a snapshot round trip over four document types at once, against **byte-equal
snapshots** rather than equal readings, because two replicas can agree on every
value and still disagree about which write produced it.

| Type | What it is |
|---|---|
| [`Blocks`](#a-document-of-blocks) | a document of paragraphs, headings, list items, quotes, code — nested |
| [`RichText`](#formatting) | text carrying bold, links, comments — anything covering a stretch |
| `Sheet` | a spreadsheet: rows and columns that move, cells addressed by identity |
| `Diagram` | an isometric diagram: nodes, connectors, zones, layers, free text |
| `Tree` | a tree whose nodes move — a file tree, an outline, threaded replies |
| `Sequence` | an ordered collection whose items move — slides, a board, layers |
| [`Counter`](#three-things-a-register-cannot-be) | a number several people add to at once |
| [`MultiRegister`](#three-things-a-register-cannot-be) | a value two replicas are allowed to disagree about |
| [`Set`](#three-things-a-register-cannot-be) | a set of names — labels, participants, what is showing |
| `Ink` | what is drawn by hand |
| `Blobs` | the files a document refers to but is not made of |
| [`Undo`](#three-things-that-are-not-state) | putting back what this replica did |
| [`Proposals`](#three-things-that-are-not-state) | changes that are not part of the document yet |
| `Register`, `RecordMap`, `Cell` | the pieces the rest compose |

What follows is why each is shaped the way it is. In every case the shape is
forced by something the obvious version gets wrong.

## Records whose fields merge apart

A record stored as one opaque value loses an edit: two replicas that change two
different fields collide, and last-writer-wins discards one of them wholesale.
`RecordMap` gives each field its own map key, so only a genuine same-field
conflict is one. `Register` is the other degenerate case — one key, and nothing
to merge that the map does not already merge.

Stable identities come from operations. A type that needs one writes a map key
whose value nobody reads and takes the identity of the write: unique across
replicas, reload-safe, never reused.

## Things that move

`crdt.List` is an RGA. It decides where a new element goes against every other
element arriving at the same moment, per element, which is exactly right for
text — and it has no operation for moving something already in it. Written with
the operations it does have, a move is a delete and an insert: two operations,
which a concurrent move of the same item splits, leaving the item in both places
or in neither.

So `Sequence` and `Tree` carry position as a **rank**, a string with another
always available between any two. A move is one field write, and two replicas
moving the same item are two writes to one field, which the map already settles.

A `Tree` has a second problem: a parent merges on its own, but the *shape* two
legal concurrent moves make between them is a ring. It is resolved when the tree
is read, by rules that are a function of the state alone. A node whose parent is
not a live node reads as a child of the root, so deleting a folder does not
delete a file somebody concurrently moved into it — the file resurfaces, and
that direction is deliberate: a tree that loses work to a concurrent delete
cannot be trusted with a project.

## Three things a register cannot be

**A count.** Read it, add one, write it back: two replicas holding 7 both write 8
and a vote is lost. The mistake is not in the register — *"add one" is not a
value*, and writing a value cannot say it. `Counter` keys the map by site, so
concurrent additions are concurrent writes to different keys and nothing
conflicts at all.

**A disagreement.** Two people rename the same thing at once and one of the names
is gone, with nothing anywhere recording that there was a second — not in the
state, not in the operations, not in anything a reader could show.
`MultiRegister` keeps a version vector beside each replica's value and reads the
ones nothing dominates. Choosing between them is *writing the one chosen*, since
a write dominates everything its writer could see, so there is no operation for
settling and none is needed. A vector rather than a clock, because a Lamport
clock gives a total order and the question here is which pairs are *unordered*.

**A set.** Keyed by the name, one replica adds a label while another, which has
never seen it, takes it away; one write wins by an order that has nothing to do
with what either knew. In `Set` every addition mints a tag and a removal takes
away the tags it can see. An addition nobody had seen is untouched — not as a
policy but for want of anything to base one on.

## Formatting

Written into the sequence — a bold-on character, a bold-off character — two
replicas bolding overlapping stretches produce interleaved markers and the text
between them reads as bold on one replica and not on the other. Written per
character it converges and costs a write per letter, forever, and each of those
writes is stored forever.

A `RichText` mark is one operation naming two boundaries, and the formatting is
worked out when the text is read. A boundary is a character and a *side* of it,
which is exactly the bold-continues / link-does-not distinction: bold ends at
the boundary before the next character and grows as you type; a link ends at the
boundary after its last character and does not.

## A document of blocks

A `RichText` per block converges and does not scale, for a reason that has
nothing to do with merging: a part cannot be taken out of a composite, and a
version carries one entry per part. A thousand-block document is then a thousand
entries exchanged on **every** sync, and the version of an empty document that
once had a thousand blocks is the same size as the version of a full one.

`Blocks` is one text, one marks map and one blocks map — **three parts however
many blocks there are.** Measured in its own tests, and asserted there so the
claim fails if it stops being true:

| a thousand blocks | version vector | parts |
|---|---|---|
| a part apiece | 18894 bytes | 1000 |
| `Blocks` | **35 bytes** | 3 |

A block begins at a **marker character** rather than at a boundary between two
of them, and that is the whole of why it works. Two people can edit the same
seam at the same moment and mean different things — one is finishing a
paragraph, the other is starting the next — and both are the same offset. A
boundary faces one way or the other and not both, because there is one place in
the sequence to insert at; a character has two sides. So each intention is its
own insertion at its own place, and the sequence already knows how to keep two
of those apart.

Nesting is a **depth** and not a parent. The order of the blocks is already
decided, by the text, and a parent pointer is a second statement about the same
arrangement — one that can contradict the first, leaving a block reading before
the block it hangs under, with no answer that is not an arbitration. Every
sequence of depths is a document somebody can read.

## Handwriting, and files

A stroke held as one value has its whole path rewritten by every point, so the
person watching sees the line redrawn rather than extended. `Ink` appends points
to a stream of their own, each saying which stroke it belongs to — one stream
rather than one part per stroke, for the reason `Blocks` gives.

A file as one map value is one operation the size of the file: it cannot be sent
as it is read, resumed if the connection drops, or recognised as one a peer
already has. `Blobs` cuts it into chunks stored under the hash of their own
bytes, so the same chunk written by two replicas is the same key *and* the same
value — nothing to merge, and nothing stored twice.

## Three things that are not state

**Undo is not a stack of states.** Restoring one throws away what everybody else
has done since, and travels to them as an instruction to do the same. An undo
here is a *new edit*, made now, whose effect is that the old one did not happen,
and it reaches a peer as an ordinary edit with no code needed in front of it.
Edits go through the manager, because inverting one afterwards is impossible
from what the document keeps: a reported change carries the text that was
inserted, never the text that was removed.

**History costs nothing, because it is already kept.** A document that merges
without a server carries the identity of every operation and of every deletion,
so *"was this character there, and was it visible"* is a question a version
vector already answers. `TextAt`, `LenAt` and `ChangesSince` ask it. No log is
added. A map keeps one record per key, so a map-backed type can say when its
current value was written and not what preceded it — which the documentation
says rather than hides.

**A proposal is a replica that has not synced.** Two copies of a text can be
compared, and what a comparison produces is a *difference*; applying a
difference mints new characters, so every comment and cursor anchored to the
ones it replaced is left pointing at nothing — a review that accepted a wording
change would take the comments off the paragraph around it. `Proposals` records
the document's own operations instead, made against the document's own
identities. Accepting is `Apply`, which merges with whatever happened meanwhile
because that is what a replica returning from offline does; turning one down
costs nothing, because the operations were never applied.

## Start here

```go
docs := structured.NewBlocks(1)

title, _, _ := docs.Insert(structured.DocStart, "heading")
docs.InsertText(title, 0, "On rivers")

body, _, _ := docs.Insert(title, "paragraph")
docs.InsertText(body, 0, "They run downhill.")
docs.Mark(body, 0, body, 4, "bold", nil, structured.ExpandEnd)

for _, block := range docs.List() {
    fmt.Printf("%s: %s\n", block.Type, block.Text)
}
// heading: On rivers
// paragraph: They run downhill.
```

Every type here is a [`crdt.Composite`](crdt.md) underneath, so it is one
snapshot, one version and one thing to authorise — and
[`collab`](collab.md) carries any of them between people without knowing which
it is.
