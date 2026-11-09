---
layout: default
title: State Divergence Localization and Repair
nav_order: 3
---

- Feature Name: divergence_localization_and_repair
- Start Date: 2026-11-01
- RFC PR: -
- Fabric-X Component: fabric-x-committer (snapshot hasher, sidecar, validator-committer), fabric-x-common
- Fabric-X Issue: -

# Summary
[summary]: #summary

[RFC 0004 (Database Snapshot and Checkpoint)](https://github.com/hyperledger/fabric-x-rfcs/pull/8)
lets organizations attest to a state hash, and it halts a committer whose state does not match the
attested hash. It leaves the next steps as future work: finding where the states differ, and
repairing them. This RFC defines those steps as State Divergence Localization and Repair (SDLR).

1. **Bucketed state digests.** The snapshot hasher stores a digest per table and per *bucket*, where
   a bucket is the set of rows whose keys share their first K bits. The root hash is derived from
   these digests, so the extra data adds no measurable cost to hashing. The snapshot transaction
   carries K, so every organization uses the same value.
2. **An offline compare tool.** Each organization exports its digests from its state database and
   sends them to the other. The tool compares the received digests with the local ones, finds the
   differing buckets, and then the two exchange only the rows of those buckets. The tool writes a report that names, for each
   differing key, the transactions that wrote it and their block numbers.
3. **Offline repair tools.** *Rollback* restores an older verified clone and replays the ledger.
   *Patch* restores the diverging clone, replaces the differing buckets with the peer's rows,
   verifies the result, and replays. *Reset* replays from genesis. Each one ends by checking the
   repaired hash against the checkpoint, which the endorsers signed.

# Motivation
[motivation]: #motivation

RFC 0004 detects divergence. When a checkpoint's hash differs from the local hash, the
validator-committer (VC) records both hashes and the sidecar stops taking blocks. The committer then
sits halted, and the clone of the diverging state is kept as evidence. The only thing the operator
has is two 32-byte hashes that differ.

That is not enough to act on. Today an operator who wants to know *what* differs has to dump and
compare two full state databases. The cost grows with the size of the database, not with the size of
the divergence. For example, `tx_status` gains one row per committed transaction. At 100,000
transactions per second that is 8.6 billion rows a day. A full dump exchanged between two
organizations over a support call is not a realistic incident response.

Fixing the divergence is equally manual. RFC 0004 says that recovery "depends on bootstrap-from-clone
and rollback-to-a-block capabilities that are out of scope". `committer db clone` lists and deletes
snapshot clones, but no tool restores the state database from a clone, replays the ledger from it, or
proves that the result matches the other organizations.

The expected outcome:

- Two organizations can find the differing rows by exchanging megabytes of digests and the rows of
  the differing buckets, not a database dump.
- The report starts the root-cause analysis from a block number and a transaction ID, not from a hex
  key.
- A repair ends with a hash that matches the checkpoint the endorsers signed, not merely the hash of
  one peer. A repair that has not been verified is not done.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## New concepts

- **Bucket.** A contiguous range of a table's keys: all rows whose key starts with the same K bits.
  A table has at most 2^K buckets. Each bucket has a SHA-256 digest of its rows.
- **Bucket bits (K).** The number of leading key bits that select a bucket. The snapshot
  transaction carries K, so every organization hashes with the same K. `tx_status` uses 2K bits,
  because its keys are hex text (see [Key distribution](#key-distribution)).
- **Snapshot digests.** The output of hashing one snapshot: the root hash, each table's digest,
  and its non-empty bucket digests, with the snapshot's block number, K and a format version. The
  hasher stores them in the state database. They take at most about 2.3 MB per table at K = 16.
- **Digest file.** The snapshot digests exported from the state database, for sending to another
  organization. It is a transfer format only; the state database holds the stored copy.
- **Diverging clone.** The clone of the snapshot whose hash did not match the checkpoint. It is never
  modified, and it stays available as evidence.

## Taking a snapshot

The flow is the one from RFC 0004, with one addition: the snapshot transaction now carries K.

```go
tx := &applicationpb.Tx{
    Namespaces: []*applicationpb.TxNamespace{{
        NsId: committerpb.SnapshotNamespaceID, // "_snapshot"
        ReadWrites: []*applicationpb.ReadWrite{{
            Key:     []byte("bucket-bits"),
            Version: nil,          // nil, as for a checkpoint
            Value:   []byte{16},   // K = 16, i.e. 65,536 buckets per table
        }},
    }},
    // Endorsements satisfying SnapshotEndorsement.
}
```

The hasher folds each row into its bucket digest. It derives each table's digest from the table's
bucket digests, and the root from the table digests. It stores the digests in the state database, in
the same transaction that records the root. Nothing else about snapshotting changes.

## When a checkpoint halts the committer

The sidecar log shows the halt:

```
Halting block intake: checkpoint TX [3f9a…] for snapshot block [120000] diverged:
local snapshot hash 7c21… for block 120000 does not match the checkpoint hash 0b4e…
```

The operator then works through these steps:

1. **Contact another organization** whose hash matches the checkpoint. Each organization exports
   its digests for snapshot block 120000 and sends the file to the other:

   ```
   $ committer db divergence export --config vc.yaml --snapshot-block 120000 \
       --out digests-120000-orgB.bin
   ```

2. **Compare the peer's digests** with the local ones. The command takes only the peer's file; it
   reads the local digests from the state database:

   ```
   $ committer db divergence compare --config vc.yaml --peer digests-120000-orgB.bin
   snapshot block 120000, K=16, format v1
   root      MISMATCH
   tables    41 match, 2 differ
     tx_status     3 of 65,536 buckets differ: 0x3a41 0x9c02 0xe117
     ns_token      1 of 65,536 buckets differ: 0x0f20
   wrote bucket request: request-120000.json (4 buckets)
   ```

3. **Exchange rows.** The peer exports the rows of exactly those buckets from its clone, and sends
   back a small file:

   ```
   $ committer db divergence export-rows --clone snapshot_120000 --request request-120000.json \
       --out rows-120000-orgB.bin
   ```

4. **Write the report:**

   ```
   $ committer db divergence report --clone snapshot_120000 --peer-rows rows-120000-orgB.bin \
       --ledger /var/hyperledger/ledger --out divergence-120000/
   ```

   The report lists every differing row, and the transactions that wrote each one:

   ```
   tx_status  tx 5be1…  local: COMMITTED @ block 119873 tx 4
                        peer:  ABORTED_MVCC_CONFLICT @ block 119873 tx 4
   ns_token   key "acct/7741"  local: value 0x2d… version 12
                               peer:  value 0x91… version 11
              last written by tx 5be1… @ block 119873 tx 4
   ```

   Here both differences point at one transaction. The two organizations disagreed on whether it
   committed. That is where root-cause analysis starts.

5. **Fix the bug, then repair** with one of the modes below.

## Repair modes

Every repair restores the state database from a clone, then replays the ledger from that clone's
point. The modes differ in which clone they restore, and in what they do before replaying.

| mode     | restores from               | before replay                                        | replays                    | needs a peer |
|----------|-----------------------------|------------------------------------------------------|----------------------------|--------------|
| patch    | the diverging clone         | replace the differing buckets with peer rows, verify | snapshot → checkpoint      | yes          |
| rollback | the previous verified clone | none                                                 | one full snapshot interval | no           |
| reset    | none (empty database)       | none                                                 | from genesis               | no           |

```
$ committer db divergence repair patch --clone snapshot_120000 --peer-rows rows-120000-orgB.bin
restored state database from snapshot_120000 (clone renamed to snapshot_120000_evidence)
replaced 4 buckets (2 tables, 9 rows)
recomputed 4 bucket digests, reused 2,818,044
root 0b4e… matches the checkpoint hash for block 120000
state database ready: resume from block 120000
```

All three modes run offline, with the committer stopped. Each one can be re-run safely if it is
interrupted.

Patch does not have to trust the peer. The checkpoint transaction that halted the committer carries
the hash a majority of endorsers signed. The repair compares the repaired root with that hash, so a
peer that sent wrong rows produces a repair that fails, not a wrong state that commits. When the
replay reaches the checkpoint again, the VC checks the hash once more, as it does for any checkpoint.

Patch's remaining cost is that the repaired state at the snapshot block was produced from the peer's
rows, not by replaying the ledger. The hash proves it is the same state the endorsers attested to.
Rollback and reset rebuild everything from the ledger, so they do not have this cost.

## Security and administration impact

- Digest files reveal only hashes. Exported rows reveal actual state, but only for the differing
  buckets, and the peer already holds its own rows for the same keys. A corrupted value is the
  exception: it is data the peer has never seen (see [Drawbacks](#drawbacks)). Organizations decide
  what to share, and the files move through whatever channel they agree on.
- A repair writes the state database and runs with the committer stopped, so it needs database
  administrator credentials, not channel credentials.
- Repair never modifies the diverging clone, so the evidence survives a repair that turns out to be
  wrong.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Hashing today

The snapshot hasher, as implemented in fabric-x-committer, scans every hashed table in primary-key
order, in pages, and folds each row into one SHA-256 per table:

```
row       = len(key) ‖ key ‖ len(value) ‖ value [‖ version]   (version: 8 bytes, namespace tables only)
table     = SHA256( row₀ ‖ row₁ ‖ … )                          (length prefixes: 8-byte big-endian)
root      = SHA256( Σ_tables  len(name) ‖ name ‖ len(table) ‖ table )
```

The hashed set is every user namespace table `ns_<id>` registered in `ns__meta`, plus `ns__config`,
`ns__meta`, `tx_status` and `ns__checkpoint`. Tables are combined in a fixed order: the system tables
first, then `ns__meta` registry order. `metadata` and `ns__snapshot` are excluded, because each
organization writes them locally.

Only the root is stored. The table digests are computed and then discarded.

## Bucketed digest scheme

### Definition

```
K          = bucket bits, carried by the snapshot transaction
K_t        = 2K for tx_status, K for every other table
bucket(k)  = the first K_t bits of key k, read as a big-endian integer;
             a key shorter than ⌈K_t/8⌉ bytes is right-padded with zero bytes
B(t, b)    = the rows of table t whose key falls in bucket b, in primary-key order
bucket     = SHA256( row₀ ‖ row₁ ‖ … )   over B(t, b), with the same row encoding as today
table      = SHA256( Σ_{b non-empty, ascending}  uint32_be(b) ‖ bucket(t, b) )
root       = SHA256( Σ_tables  len(name) ‖ name ‖ len(table) ‖ table )       (unchanged combine)
```

Each row is folded into exactly one hash. The table digest folds only 36 bytes per non-empty bucket.
So the only extra work over today's single fold is one SHA-256 finalization per non-empty bucket.

The bucket function must never decrease as the scan moves forward in primary-key order. That is what
lets the scan keep one hasher live, and what makes every bucket a contiguous key range. A prefix of
the raw key bytes has this property by construction. Any other bucket function must preserve it.

Bucket numbers are encoded as `uint32`, so 2K ≤ 32, which means K ≤ 16.

### Why it costs nothing extra

Keys arrive in primary-key order, so the scan visits buckets in ascending order. One hasher is live
at a time. When the bucket changes, the hasher finalizes 32 bytes and resets. No extra reads, no
extra connections, and O(1) memory beyond the digests themselves.

Measured on an Intel Xeon (Cascadelake), with one core, 8,000,000 rows of 32-byte keys and 40-byte
values, and K = 16:

| scheme                                          | rows/s (M) | vs today               | stored per table |
|-------------------------------------------------|------------|------------------------|------------------|
| today: single fold, root only                   | 3.0–3.1    | —                      | 32 B             |
| **bucketed: root derived from bucket digests**  | 2.9–3.1    | −0.7% to +5.1% (noise) | 2 MB             |
| root kept byte-identical + separate bucket fold | 1.5        | +110%                  | 2 MB             |
| per-row binary Merkle tree                      | 0.8        | +279%                  | 64 B per row     |

The third row is the cost of keeping today's root encoding byte-identical: every row has to be
hashed twice. SDLR accepts a new root encoding to avoid that cost (see
[Rationale](#alternatives)).

The scan itself dominates the elapsed time. The hasher fetches and hashes each page in sequence.
For a 1-billion-row table read in 1,000-row pages, an estimated 10–50% of the elapsed time is
hashing, for page latencies between 3 ms and 0.3 ms. This has not been measured on a cluster.

### Determinism

The digests depend only on table content, K and the format version. They do not depend on page size,
worker count, the order in which tables finish, or the number of nodes. Two organizations with the
same content, the same K and the same format version store identical digests, and export
byte-identical digest files.

### Where K comes from

The snapshot transaction carries K as its single read-write. This is the shape the `_checkpoint`
namespace already requires: exactly one read-write with a nil version, and no reads-only or blind
writes. The key is `bucket-bits` and the value is one byte holding K. RFC 0004's `_snapshot`
namespace is marker-only, and the sidecar rejects any write in it with
`MALFORMED_SNAPSHOT_NOT_MARKER_ONLY`. SDLR changes that form check to require the read-write.

- **The read-write is never applied.** The VC's preparer replaces the `_snapshot` namespace with the
  snapshot record it creates, and does not apply the transaction's own writes. So no snapshot
  overwrites the previous snapshot's K. The preparer copies K into a new field of the record.
- **K must survive the clone step.** When the VC creates the clone, it rebuilds the snapshot record
  from scratch, so it must copy K forward. Every later status change edits the existing record, so K
  survives those.
- **The dependency it creates costs nothing.** The coordinator adds the read-write's key to its
  dependency graph, so every snapshot transaction depends on the previous one. The sidecar already
  drains before and after each snapshot transaction, so two are never in flight together.
- **K is endorsed.** The namespace signature covers its read-writes, so `SnapshotEndorsement`
  covers K.

- **K's permitted range is a constant in code**, never node configuration. If two organizations
  allowed different ranges, a K inside one range and outside the other would make one organization
  accept the snapshot transaction and the other reject it. That would fork their `tx_status` tables,
  which is the very divergence SDLR exists to find.
- **The submitter chooses K** within that range, for example from its own estimate of the row count.
  Agreement comes from the endorsed, ordered transaction, so a poor estimate costs a larger rescan
  of each differing bucket, and nothing else.

### Key distribution

The keys of most tables are not uniformly distributed bytes, and bucketing by leading bits inherits
their skew:

- **`tx_status`** keys are transaction IDs. A Fabric-X transaction ID is the hex encoding of a
  SHA-256 digest, stored as its UTF-8 bytes. Each key byte is one of 16 characters, so the first K
  bits carry only K/2 bits of entropy. With K = 16, two million transaction IDs fill 256 of the
  65,536 buckets. The other 65,280 buckets are empty.
- **User namespaces** use keys that applications choose, such as `acct/0001`, `acct/0002`, …. Keys
  with a common prefix fall into a few buckets.

Empty buckets cost nothing: they are not stored and not folded. Skew only makes the populated buckets
larger, which makes the rescan of a differing bucket more expensive. It never affects correctness.

**Proposal.** `tx_status` takes 2K bits instead of K. Each hex character carries 4 bits, so 2K bits
of hex text carry K bits of entropy: with K = 16, the 32-bit prefix fills all 65,536 populated
buckets that a uniform table would. This was measured with two million hex transaction IDs.

Decoding the hex before taking the prefix was considered and rejected. The committer accepts any
non-empty transaction ID, and a decoded prefix for hex IDs mixed with a raw prefix for other IDs is
not monotonic in key order, which breaks the scan. With 2K raw bits, a non-hex ID only lands in a
sparser bucket.

User namespaces keep K bits, and their skew remains. See
[Unresolved questions](#unresolved-questions).

### Storage and lifetime

The digests of a snapshot are kept for as long as its clone exists, and deleted with it. That ties
them to `committer db clone delete`, so they need no retention policy of their own.

The digests are excluded from the hashed set, for the same reason `ns__snapshot` is: hashing them
would fold each snapshot's digest into the next snapshot.

The hasher writes the digests in the same transaction that moves the snapshot record to
`COMPLETED` and records the root. So a committed root always has its digests, and a crash during
hashing leaves neither.

At K = 16, the stored digests take at most 36 bytes per non-empty bucket: about 2.3 MB per table,
and about 100 MB for a channel with 40 namespaces. This storage is the main overhead of SDLR. Which
database structure holds them is an [unresolved question](#unresolved-questions).

### Digest file format

```
header   magic "FXSD" ‖ uint16 format_version ‖ uint64 snapshot_block ‖ uint8 K
root     32 bytes
tables   uint32 count, then for each table in combine order:
           len(name) ‖ name ‖ table_digest (32) ‖ uint32 non_empty_buckets
           ‖ { uint32 bucket ‖ digest (32) } × non_empty_buckets
```

`committer db divergence export` writes this format. The header makes the file self-describing, so
the receiving organization learns the snapshot block, K and format version from the file alone, and
no operator has to supply them by hand, under pressure, during an incident. The stored digests carry
the same three values. They cannot be read from the clone, because a clone does not contain its own
`_snapshot` record: it is created before the snapshot transaction commits.

The format version matters for repairs over time, not between organizations. A rollback rehashes an
older clone, and if the hashing scheme changed since, the tool must refuse and say so. It must not
report the change as a divergence.

## Compare protocol

Compare is always two-sided. When a checkpoint verifies, no proof is needed. When it fails, the
operator contacts the other organizations. So SDLR defines no inclusion proofs.

Each round is a file exchange between operators. A sent file always holds the sender's data; the
receiver's side always comes from its own state database:

| round | sent                          | size (1-billion-row table, K = 16) | narrows to             |
|-------|-------------------------------|------------------------------------|------------------------|
| 0     | root hash                     | already known from the halt        | does the state differ? |
| 1     | exported digests              | ~2 MB per table                    | the differing buckets  |
| 2     | rows of the differing buckets | ~15,000 rows per bucket (~1 MB)    | the differing rows     |

The tool reads the peer's rows and its own rows for each requested bucket, with a range scan, and
diffs them by key. Each bucket is a contiguous key range, so this is one index seek per bucket. A
missing or extra row affects only the bucket that contains it.

Compare takes only the peer's digest file. The local digests come from the state database, never
from a file, so an operator cannot compare against a stale or mistaken local export. The tool checks
that the peer's file names the same snapshot block, K and format version as the local record, and
refuses to compare them otherwise.

## Report

The report is a directory of human-readable files, written once, with no service and no API:

- **`summary.txt`.** Which tables match, which differ, and how many buckets differ.
- **`tx_status.txt`.** For each differing transaction: its ID, and the status and height on each
  side.
- **`<namespace>.txt`.** For each differing key: the key, and the value and version on each side.
- **`writers.txt`.** For each differing key, the transactions that wrote it, with block number,
  transaction number and status, found by scanning the local block store from the previous verified
  snapshot to this one. The divergence must have started after that snapshot, because its checkpoint
  verified. The sidecar's block store holds each block with the committer's per-TX
  status in its metadata, so the scan needs no running service.

The report sorts entries by the lowest block number involved, so the earliest divergence comes
first.

## Repair

### What the repair restores from

When the committer halts, its state is not at the snapshot point. Three facts determine where the
repair starts:

- **A clone holds the state just before its snapshot transaction.** The VC creates the clone before
  the snapshot transaction's batch commits. So the clone contains the regular transactions of the
  snapshot's own block, but not the snapshot transaction's `tx_status` row or its `_snapshot`
  record.
- **The state database is ahead of the clone when the committer halts.** The sidecar drains only
  around the snapshot transaction. Intake continues until the checkpoint transaction arrives, and
  that is when the committer halts. So the database holds every block between the snapshot and the
  checkpoint.
- **The halted checkpoint is not committed.** It gets no transaction status, and the sidecar
  re-delivers its block on the next start.
- **The halt must record the checkpoint hash as a field.** The patch step verifies against it
  offline, before the replay delivers the checkpoint again. Today the VC writes both hashes only into
  the record's error text. fabric-x-common already defines a `SnapshotState.checkpoint_hash` field
  for the hash that differed, and the halt should set it.

### Procedure

1. **Rename the clones the replay will pass.** Before replaying, the repair renames every clone
   named `snapshot_<block>` for a snapshot block the replay covers, and keeps each one as evidence.
   This step is necessary because the restored state has no `tx_status` row for those snapshot
   transactions. The VC therefore treats each re-delivered one as new and creates its clone, and it
   reuses any database that already has the name. For the diverging snapshot, that would rehash the
   unrepaired clone, and the replayed checkpoint would halt again on the same mismatch.
2. **Restore** the state database from the selected clone, discarding the current state. Reset
   instead drops the database and initializes it empty, as `committer db init` does.
3. **Patch only:** replace each differing bucket's whole key range with the peer's rows for that
   range. Replacing whole buckets handles missing and extra rows without per-row diff logic, and it
   makes the set of buckets to re-verify exactly the set that was replaced.
4. **Patch only: verify before replay.** Recompute the digests of the replaced buckets, re-fold them
   with the stored digests of every other bucket into the table digests and the root, and check that
   the root equals the checkpoint hash. The checkpoint was signed by a majority of endorsers, so this
   proves the result, not just agreement with one peer. Verification has to happen here, at the
   clone's point, because that is the only height the checkpoint attests to. Recomputing only the
   replaced buckets is correct because step 2 guarantees that every other key is byte-identical to
   the content those stored digests were computed from.
5. **Roll back the block store** to the block before the restored clone's snapshot block, using
   fabric-x-common's `blkstorage.Rollback`. Reset clears it with `blkstorage.ResetBlockStore`. The sidecar's block store records each block with its
   per-TX statuses, and on restart it skips any block below its own height. Without this step, the
   replayed blocks would keep their pre-repair statuses, and the block store would disagree with the
   repaired state.
6. **Set the resume point** to the restored clone's snapshot block, so the sidecar and coordinator
   restart from it instead of their pre-repair progress. Replay starts at the snapshot's own block,
   not the block after it: the clone holds that block's regular transactions but not its snapshot
   transaction. The regular transactions replay as duplicates, and the VC already resolves a
   duplicate to its original status.
7. **Start the committer.** It replays forward. Rollback and reset are verified when the replay
   reaches the halted checkpoint, by a full rehash, because they change every table and no stored
   bucket digest survives them.

### Constraints

- **The replay needs its blocks.** The first block to replay must be at or after the ledger's pruned
  start block. Otherwise the repair fails before changing anything, and names the missing range.
- **Re-runnable.** Each step either checks whether it already ran or can safely run again, so an
  interrupted repair can be resumed by running the same command.
- **Offline.** The tool refuses to run while any committer service holds a connection to the state
  database.
- **Report first.** Write the report before repairing. Rolling back the block store discards the
  pre-repair statuses that the report reads.

### The hold and halt path during replay

RFC 0004's checkpoint verification still applies during replay. The halted checkpoint was never
committed, so it gets no transaction status, and the VC verifies it again when the replay
re-delivers it. It compares the checkpoint with the hash of the snapshot it just recreated. After a
correct repair the hashes match and the checkpoint commits. After an incorrect repair the committer
halts again, and the evidence clone is still there for another attempt.

This is what lets every mode be checked against the endorsers rather than against a peer:

- **Patch** is checked twice: before the replay, against the stored checkpoint hash (step 4), and
  again when the replay reaches the checkpoint. It can start ingesting the ledger as soon as step 4
  passes.
- **Rollback and reset** are checked when the replay reaches the halted checkpoint, by a full rehash
  of the recreated snapshot. They do not have to wait for a later snapshot.

## Clone lifecycle

Rollback is near-immediate only while the previous verified clone still exists. Patch depends on the
diverging clone. The administrator decides whether to keep a clone, back it up and delete it, or
delete it outright, using `committer db clone list` and `committer db clone delete`. The delete
command works on the state database directly, and refuses a snapshot that is not `CHECKPOINTED` or
`ABORTED`, because until then its clone may still be hashed.

SDLR relies on that lifecycle in four ways:

- **A verified clone is a `CHECKPOINTED` one.** An `ABORTED` snapshot was closed without comparing its
  hash, so rollback never restores from its clone.
- **The diverging clone is already protected.** A halt records the mismatch but leaves the snapshot's
  status unchanged, so `db clone delete` refuses to delete the clone before a repair.
- **The repair tracks its evidence clones.** A repair renames the clones the replay will pass (see
  [Procedure](#procedure)). Restoring discards the `_snapshot` records that named them, and
  `db clone delete` deletes only a clone that a record names. So the repair command records the
  evidence clones it creates, and is the one that deletes them.
- **Documented guidance** to keep the last `CHECKPOINTED` clone until the next snapshot is
  checkpointed. `db clone delete` accepts any `CHECKPOINTED` clone, so the tool does not prevent
  deleting the one that rollback needs.

Backup files are produced only on demand, for a joining organization or a divergence, not routinely.
Producing them is the bootstrapping work's responsibility (see [Dependencies](#dependencies)).

## Impact per component

- **fabric-x-common**
  - `SnapshotState` gains `bucket_bits` and `digest_format_version`.
  - The bucketed digest scheme lives here, together with its normative test vectors, so that tools
    outside the committer can reproduce the same digests.
- **fabric-x-committer**
  - **Sidecar:** the `_snapshot` form check requires exactly one `bucket-bits` read-write with a nil
    version, as the `_checkpoint` check does.
  - **VC:** the preparer copies K into the `SnapshotState`, and the clone step carries it into the
    rebuilt record. On a halt, the VC sets `SnapshotState.checkpoint_hash` to the checkpoint's hash.
  - **Snapshot hasher:** computes the bucketed scheme, stores the digests in the state database with
    the root, and excludes them from the hashed set.
  - **CLI:** a new `committer db divergence` group, next to `db init` and `db clone`, with `export`,
    `compare`, `export-rows`, `report` and `repair {patch,rollback,reset}`. Repair rolls back the sidecar's block store with fabric-x-common's
    existing `blkstorage.Rollback`, and tracks the evidence clones it creates (see
    [Clone lifecycle](#clone-lifecycle)).

# Drawbacks
[drawbacks]: #drawbacks

- **The root hash changes.** Snapshots taken before the change and after it produce different
  roots for the same content. A network must switch at a point when no snapshot is awaiting a
  checkpoint, and every organization must switch together. Tools must check the format version
  before comparing.
- **Storage per snapshot.** About 2.3 MB per table at K = 16, kept for as long as the clone. This is
  small next to the clone, but it is new state with its own failure modes.
- **Patch's state does not come from the ledger.** The patched state at the snapshot block was
  produced from the peer's rows, not by replaying the ledger. The checkpoint hash proves it matches
  what the endorsers attested to, but operators who need the state rebuilt from the ledger itself
  must use rollback or reset.
- **Skew.** Keys that share prefixes put many rows in a few buckets, which makes rescanning a
  differing bucket slower. Giving `tx_status` 2K bits fixes this for the largest table, but not for
  application keys.
- **Exchanging rows exposes them.** The peer sees the actual differing rows. Both organizations
  already hold rows for these keys, but a corrupted value might reveal data the peer would not
  otherwise see.

# Rationale and alternatives
[alternatives]: #alternatives

**A deep Merkle tree or a Merkle-Patricia trie.** These are the usual structures for comparing two
states, and they support inclusion proofs. They are not chosen here, for two reasons:

- **Nothing consumes an inclusion proof.** When a checkpoint verifies, no proof is needed. When it
  fails, the operator contacts the other organizations.
- **They are expensive.** Interior levels only save bytes exchanged, and one level of bucket
  digests already costs only a few megabytes. A per-row binary Merkle tree measured 3.8× slower than
  today's fold, and storing it costs 64 bytes per row: 64 GB for a 1-billion-row table. A
  Merkle-Patricia trie adds key-addressed proofs without the peer holding the table, which this
  protocol does not need.

**Leaves indexed by row position.** Building the tree over row positions in primary-key order looks
natural, but one missing row shifts every later leaf, so the whole tree diverges. A missing row from
a mishandled transaction is exactly the case SDLR exists to find.

**Buckets by `H(key)`.** Hashing the key before taking its leading bits makes every table uniform,
but a differing bucket is then scattered across the key space. Rescanning it becomes a filtered full
scan instead of an index seek.

**Keep the root byte-identical.** This preserves today's root at the cost of hashing every row twice,
which is +110% on the hashing step. The real requirement is determinism, and the format version
covers the change.

**K from node configuration.** Configuration drift between organizations is silent until an
incident, and a different permitted range between organizations would fork them. The snapshot
transaction is ordered and endorsed, so every organization reads the same K.

**Recompute the digests at compare time instead of storing them.** Both clones are immutable, so the
digests could be recomputed when needed. But that is a full scan on each side, an estimated 22 to 55
minutes for a 1-billion-row table at 1–3 ms per page, while the committer is halted and two
organizations are on a call. Stored
digests also let a patch re-verify only the buckets it replaced.

**A committer API to fetch the peer's digests.** This makes compare a service call, but it requires
an authenticated cross-organization endpoint on a committer that is halted. Files fit the human step
already in the flow. It remains an [open question](#unresolved-questions).

**Do nothing.** A halted committer stays halted, and operators compare database dumps by hand.

# Prior art
[prior-art]: #prior-art

- **Hyperledger Fabric `peer ledgerutil`.** `compare` diffs two peers' snapshot files and writes all
  differences, plus the first N ordered by height. `identifytxs` scans a peer's block store for the
  transactions that wrote given keys. `verify` checks block-store hashes. The compare tool and report
  here follow `compare` and `identifytxs`. Fabric compares whole snapshot files, whereas SDLR
  compares digests first and exchanges only the differing rows. `verify` is out of scope.
- **Anti-entropy in Dynamo and Cassandra.** Replicas exchange Merkle trees over key ranges, and
  descend only into ranges that differ. The bucket digests here are a single-level version of the same
  idea.
- **rsync.** It finds differences without transferring the whole file, by exchanging block checksums
  first. The same principle applies here: exchange digests, then only the differing data.
- **RFC 0003 (Block Data Hash Scheme).** It also chose a flat two-layer hash over a Merkle tree, for
  the same reason: no current consumer needs per-item inclusion proofs. Its named, immutable scheme
  is the model for the format version here.

# Testing
[testing]: #testing

- **Normative test vectors** in fabric-x-common for the bucketed scheme: empty tables, single rows,
  keys shorter than ⌈K_t/8⌉ bytes, several values of K, and hex and non-hex transaction IDs.
- **Monotonic buckets:** for random and adversarial keys, the bucket number never decreases in
  primary-key order.
- **Determinism:** the same content hashed with different page sizes, worker counts and table
  completion orders produces byte-identical exported digest files.
- **Atomic storage:** a hasher stopped mid-job leaves no stored digests, and a `COMPLETED` record
  always has them.
- **Export and compare:** compare refuses a peer file whose snapshot block, K or format version
  differs from the local record.
- **Localization:** inject a modified value, a missing row and an extra row into a clone. Compare must
  report exactly the affected buckets and rows, and nothing else.
- **Report:** a differing key must name the transaction that wrote it, at the right block and
  transaction number.
- **Repair, end to end:** two committers with a deliberately diverged row. The checkpoint halts one.
  Patch, rollback and reset must each bring it back to the checkpoint hash, after which the replayed
  checkpoint must commit.
- **Patch with a wrong peer:** rows from a peer that also diverged must fail step 4 against the
  checkpoint hash, before any block is replayed.
- **Block store after repair:** after a repair that changes a transaction's status, the block store
  must record the new status for that transaction.
- **Repair negatives:** a patch with rows that do not fix the divergence fails verification and leaves
  the evidence clone intact. A rollback whose replay blocks were pruned fails before changing anything.
  An interrupted repair resumes when run again.
- **Form check:** a `_snapshot` transaction with K outside the permitted range, with a version on
  its read-write, with a second write or a blind write, or with no write is rejected the same way by
  every committer.
- **K survives the lifecycle:** K set by the snapshot transaction is still on the record after the
  clone step, `COMPLETED`, `CHECKPOINTED` and `ABORTED`. Two consecutive snapshots with different K
  each keep their own.

# Dependencies
[dependencies]: #dependencies

- **[RFC 0004 — Database Snapshot and Checkpoint](https://github.com/hyperledger/fabric-x-rfcs/pull/8).**
  SDLR extends its snapshot transaction, hash and clone lifecycle.
- **Backup-to-file and bootstrapping**, which are being designed separately. Comparing against a
  backup file and resetting from a bootstrap snapshot depend on that work.
- **Block-store pruning.** It decides which blocks a repair can replay.
- **fabric-x-common:** the `SnapshotState` fields and the bucketed scheme with its test vectors.

# Unresolved questions
[unresolved]: #unresolved-questions

To resolve through the RFC process:

- **How digests and rows are exchanged.** Strictly out-of-band files, as SDLR proposes, or a
  committer API? A file fits the human step and the halted committer, but an API would allow
  automation.
- **Which database structure holds the digests.** A new table keyed by snapshot, table and bucket,
  which needs a new exclusion from hashing and writes up to 2^K rows per table per snapshot? Or a
  field on the `_snapshot` record, which avoids the table, but which `committer db clone list` and the
  hasher's scheduler would then read far more often than the digests are used?
- **Skewed application keys.** Accept the skew, give each table its own K, or bucket user namespaces
  by `H(key)` and accept that a differing bucket must be rescanned with a full scan?
- **Multi-organization coordination.** Does a rollback need coordinated action across organizations,
  or is each organization independent once the bug is fixed?

To resolve during implementation:

- The exact permitted range of K and its default.
- The exact flags of the `committer db divergence` commands.
- How the tool detects that no committer service is connected to the database.

Out of scope:

- Block-store and chain-hash verification (the analogue of `ledgerutil verify`).
- Pruning, bootstrapping and external transaction indexing.
- Automated cross-organization comparison.
