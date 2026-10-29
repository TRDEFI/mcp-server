---
name: trdefi-liquidity
description: Live non-custodial stablecoin liquidity for AI agents. Use when an agent needs to find real maker positions to trade against, quote a swap on-chain at the current block, or earn swap fees on idle stablecoins without moving custody. Covers listing positions, quoting a swap, and reading settled volume across 20 networks (Ethereum, Base, Arbitrum, Optimism, Polygon, BNB Chain and 14 more). Read-only tools are free and need no key.
license: MIT
---

# TRDEFI Liquidity

Live, non-custodial stablecoin liquidity, exposed as a remote MCP server at
`https://yield.trdefi.com/mcp`.

## When to use this

Use it when an agent needs to:

- Find a **real on-chain counterparty** for a stablecoin swap, instead of a custodial venue.
- **Quote** a swap against a maker position at the current block, before committing to anything.
- See where liquidity actually sits across chains, with settled volume.
- Understand how a treasury can **earn swap fees on idle stablecoins without depositing them anywhere**:
  the funds stay in the holder's own wallet, backed by a bounded, revocable allowance.

## When not to use this

- It is not a price oracle for arbitrary tokens. Positions are liquidity-provider positions on the
  1inch Aqua shared-liquidity engine, and what an agent trades against is the pair the maker chose,
  not a pair the agent picked. See *How a position is bound* below.
- It does not sign or broadcast anything. Nothing here can move a wallet's funds.
- It is not a yield aggregator. It reports positions that exist; it does not rank or recommend them.

## Tools

| Tool | Use it for |
|---|---|
| `trdefi_stats` | Aggregate counts and settled volume over 1, 7 and 30 days. Start here to check the venue is alive. |
| `trdefi_chains` | Which networks are live, their settlement engine address, and the tokens a position can be built from. Use this instead of hardcoding chain or token identifiers. |
| `trdefi_positions` | List open maker positions. Filter by network and pair. Each row carries the strategy hash, network, pair, tokens and settled volume. |
| `trdefi_position_detail` | One position in full, by its 32-byte strategy hash. |
| `trdefi_quote` | Price a swap against one position, on-chain, at the current block. |
| `trdefi_fusion_quote` | A 1inch Fusion quote for a pair. Use it when a position answers "Strategy is gated" - the caller needs no access token, a resolver fills the order. |
| `trdefi_fusion_prepare` | Quote and build the EIP-712 Fusion order to sign. Returns typedData, order, extension, quoteId, orderHash. Nothing is signed or submitted. |
| `trdefi_fusion_submit` | Forward an already-signed Fusion order to 1inch's relayer. The server never signs and never holds a key. |
| `trdefi_badge` | A shields.io payload for the catalogue metrics, for a README. |

All nine require **no API key**. Eight are pure reads. The ninth,
`trdefi_fusion_submit`, only forwards a signature the caller has already produced - it cannot
sign, cannot hold a key, and cannot move a token on its own.

## How a position is bound, and how the swap completes

A position is not a deposit and not a pool you join. The maker signs a **bounded, revocable
allowance**: one token, one amount, one deadline. Nothing is deposited, nothing is locked, the
capital never leaves the maker's wallet, and the exit is a single transaction.

That is the whole mechanism, and it is what these tools expose. An agent does not manage a
position and does not need to understand what the maker's two tokens are. It asks for a position
to trade against, gets a price for the swap it wants, and the engine settles the swap on its
own once the swap is signed.

For the live counts, read `trdefi_stats`. They refresh on a timer, so nothing in this document
hardcodes them.

## Working with quotes

A maker may gate a position on the caller holding an access licence token, and on most networks
most of them do. Measured 2026-10-19 by sampling 12 positions on each of 17 networks, 146 quotes in
total: **62 percent** answered "Strategy is gated" and **17 percent** returned a price. Those
positions answer with a clear reason instead of a number. **That is a property of the maker's
strategy, not an error** - treat it as a signal to try a different position rather than a failure
to retry.

The split is strongly per-network. On Ethereum 58 of 60 sampled positions are gated and none
returned a price. The credential contract itself is only deployed on the 13 Aqua chains, so **Arc,
MMonad, Cronos and HyperEVM carry no such gate** and quote cleanly.

Each position carries a `quote_eligible` flag. It means the position is router-attached and its
network has a quote route. **It does not mean the quote will succeed**, and `quote_ready=true`
narrows to that routable set without removing the gate: of 50 positions sampled from the filtered
set, 49 still answered "Strategy is gated" and none returned a price.

### Pass `from` and you get the real answer

The gate is checked on `tx.origin`, so the simulation has to know who the caller is. Without
`from` the API answers as `address(0)` and returns the least specific revert available,
`0x89c62b64` - "the caller holds none". With `from` it returns `0x39c4052c`, the tx-origin
balance check, which names the actual problem.

So pass the address that will sign the swap. A caller that holds the maker's credential gets a
real price. Anyone else gets an accurate "you do not hold it" instead of the generic refusal,
which is the difference between a failed quote and a wrong request.

## Prepare endpoints

Outside the MCP surface, two HTTPS endpoints prepare unsigned transactions:

- `POST /v1/positions` - the transactions that create a maker position.
- `POST /v1/swaps` - the transaction that executes a swap.

Both answer **HTTP 402** with their payment terms when unpaid, and settle USDC on Base
(`eip155:8453`) at **$0.02 per request**, so an agent needs no account and no API key. An API key is
accepted as an alternative. **The transactions are unsigned**: the caller signs and broadcasts with its
own wallet, and TRDEFI cannot sign on the caller's behalf.

The service is x402-native - discovery is free, execution is metered - and is listed in the **x402
Bazaar** (the CDP catalogue), so agents that discover services through the Bazaar find it with no
registration.

## What TRDEFI does not do

It never takes custody, never holds funds, and never signs. A position is backed by a bounded,
revocable on-chain allowance (a plain ERC-20 `approve` - no deposit, no permit required), so exit is
one transaction: dock and revoke.
