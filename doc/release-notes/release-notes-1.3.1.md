Whippet Core version 1.3.1 is now available from:

  <https://github.com/chromatic/whippet-experimental-blockchain/releases/tag/v1.3.1/>

This is an urgent bugfix release. It fixes a chain-halting validation bug:
every node running 1.3.0 or earlier stops accepting *any* block, including
its own, at block height 80,000. Mainnet reached that height on 2026-09-21
and has been unable to progress since. The primary fix corrects validation
lookups that used the wrong block height; "Other changes" describes two
small safeguards and a consensus rule change that activates at height 100,000.

Please report bugs using the issue tracker at GitHub:

  <https://github.com/chromatic/whippet-experimental-blockchain/issues>


Compatibility
==============

Whippet Core is extensively tested on Ubuntu Server LTS, macOS and Windows.
Minimum OS compatibility can be found [in the INSTALL guide](../../INSTALL.md).

The chain-ID fix carries no new consensus activation height. A node running
the old, buggy code simply cannot build or accept any block at height 80,000
or above, so there is no way for it to diverge onto a different chain.
Upgrading is what lets a node make progress again; not upgrading leaves it
stalled, never wrong.

This release also introduces one new consensus rule: at block height 100,000
the minimum block-spacing floor is lowered from 6 s to 4 s. See "Other
changes" for the rationale. Every node and miner must be upgraded before
height 100,000 or it will reject valid post-100,000 blocks.


The bug
========

At block height 80,000, `chainIdFixConsensus` (added in 1.2.0 to resolve an
AuxPoW chain ID collision with Dogecoin) changes the network's expected
AuxPoW chain ID from `0x0062` to `0x5750`. `ConnectBlock()` already looked
this up per-height correctly. But an earlier, more permissive check —
`CheckBlockHeader()`, which runs before a block's real height is known (for
example, before its previous block has even been looked up) — always
validated against `Params().GetConsensus(0)`: the chain ID as of height 0,
which never changes. That was harmless for as long as the chain ID itself
never changed. Once it did, at height 80,000, `CheckBlockHeader()` started
rejecting every block, mined locally or received from a peer, with:

    CheckAuxPowProofOfWork : block does not have our chain ID
    (got 22352, expected 98, full nVersion ...)

Because this check runs ahead of the real per-height validation in
`ContextualCheckBlockHeader()`, no block ever got far enough to reach the
correct check. The chain simply stopped.


The fix
========

`CheckBlockHeader()` now estimates the height as `chainActive.Height() + 1`
instead of a fixed `0`. A block being validated almost always extends the
current tip, so this estimate is correct in every normal case. It does not
need to be exact: the authoritative, correctly-scoped check still happens
afterward in `ContextualCheckBlockHeader()`, which has the block's real
height via `pindexPrev` and is unchanged by this release.

Two follow-up changes complete the fix:

- `ConnectBlock()`, `AcceptBlock()`, `TestBlockValidity()` and
  `CVerifyDB::VerifyDB()` pass the block's real height instead of relying
  on the tip estimate, so re-verifying blocks below 80,000 on a node whose
  tip is above it no longer fails.
- `AcceptBlockHeader()` passes the parent's height plus one. During
  headers-first sync the active tip lags far behind the headers being
  accepted, so the tip estimate was wrong there: a node that had not yet
  connected block 80,000 could reject valid post-80,000 headers from its
  peers and stall its initial sync at that height.

A new regression test (`src/test/auxpow_tests.cpp`,
`checkblockheader_uses_tip_height_for_chain_id`) proves the fix directly: it
points `chainActive`'s tip at a synthetic height-79999 index and checks that
a block carrying the height-80000 chain ID is now accepted, while one
carrying the pre-80000 chain ID is rejected — the reverse of what the
original, fixed `GetConsensus(0)` lookup did.


Other changes
==============

- `whippetd -help-debug` now prints the extended help and exits, like
  `-help`, instead of silently starting a node. (Inherited from Bitcoin
  Core 0.13; also submitted to Dogecoin.)
- The `generate`, `generatetoaddress` and `submitblock` RPCs now refuse to
  run while a reindex or import is in progress (`fReindex || fImporting`),
  returning the same "downloading blocks" error as during initial sync.
  Mining on a tip that is still being rebuilt can fork the chain before a
  hardcoded checkpoint once the reindex catches up.
- At height 100,000, the minimum block-spacing floor drops from 6 s to 4 s
  (`minSpacingConsensus`). With the floor equal to the 6 s target, the LWMA
  retarget algorithm never observed blocks arriving faster than target, so
  difficulty could only fall to `powLimit` regardless of hashrate. A 4 s
  floor gives the algorithm real headroom below the target to respond to
  hashrate changes. **This is a consensus change: all nodes and miners must
  upgrade before height 100,000.**
- RPC basic-auth scheme validation is now case-insensitive (`Basic`,
  `basic`, etc. are all accepted), per RFC 7235.
- Inbound connections from the same /16 (IPv4) or /32 (IPv6) subnet are
  capped at 4 (`DEFAULT_MAX_INBOUND_PER_SUBNET`); whitelisted peers and
  non-routable addresses are exempt. This improves peer diversity on a
  network recovering from a stall.


Upgrading from Prior Versions
==============================

Every node currently on 1.3.0 or earlier is stalled at or below height
79,999 and needs this upgrade to make further progress. There is no wallet
or chain-data migration; the on-disk format is unchanged.

1. **Stop your Whippet daemon**:
   ```bash
   whippet-cli stop
   ```

2. **Upgrade Whippet Core** using your package manager or by building from
   source:
   ```bash
   ./autogen.sh
   ./configure
   make
   make check
   sudo make install
   ```

3. **Start the daemon**:
   ```bash
   whippetd
   ```

4. **Verify the upgrade**:
   ```bash
   whippet-cli getnetworkinfo
   # subversion should read /Devotoshi:1.3.1/
   ```

5. **Confirm the chain is moving again**:
   ```bash
   whippet-cli getblockcount
   # run again a few minutes later and confirm it has increased
   ```


Testing
========

- `src/test/auxpow_tests.cpp` gained
  `checkblockheader_uses_tip_height_for_chain_id`, which fails against the
  pre-fix code (reintroducing `Params().GetConsensus(0)` makes both of its
  assertions fail in the direction the original bug did) and passes against
  the fix, without mining through height 80,000.
- `auxpow_tests` gained `acceptblockheader_uses_parent_height_for_chain_id`
  (headers-first case, active chain empty; fails without the
  `AcceptBlockHeader()` change) and an end-to-end `CVerifyDB` test across
  the chain-ID boundary.
- `mining_tests` covers the reindex/import guard on `generate` and
  `submitblock`; `getarg_tests` covers `-help-debug`.
- Full unit test suite: 328 test cases, all passing.


Release Information
===================

**Release Date:** 2026-09-28
**Version:** 1.3.1
**GitHub Tag:** v1.3.1
**Fixes:** chain-halting `CheckBlockHeader` chain-ID validation bug at block
height 80,000, including during headers-first sync
**Consensus change:** minimum block-spacing floor lowered from 6 s to 4 s
at height 100,000 (`minSpacingConsensus`)

For the latest information, visit:
- GitHub: <https://github.com/chromatic/whippet-experimental-blockchain/>
