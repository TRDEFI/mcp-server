# TRDEFI Liquidity — MCP server

[![Listed on mcpservers.org](https://mcpservers.org/badge.svg)](https://mcpservers.org/servers/trdefi/mcp-server)
[![M8ven Score](https://m8ven.ai/badge/mcp/trdefi-mcp-server-wq7ly5)](https://m8ven.ai/mcp/trdefi-mcp-server-wq7ly5?s=readme)
[![Glama score](https://glama.ai/mcp/connectors/io.github.TRDEFI/liquidity/badges/score.svg)](https://glama.ai/mcp/connectors/io.github.TRDEFI/liquidity)
[![Smithery](https://img.shields.io/badge/Smithery-listed-blue)](https://smithery.ai/server/@trdefi/liquidity)
[![Install TRDEFI MCP](https://replit.com/badge?caption=Install%20TRDEFI%20MCP)](https://replit.com/integrations?mcp=eyJkaXNwbGF5TmFtZSI6IlRSREVGSSBMaXF1aWRpdHkiLCJiYXNlVXJsIjoiaHR0cHM6Ly95aWVsZC50cmRlZmkuY29tL21jcCJ9)

A **remote, read-only** [Model Context Protocol](https://modelcontextprotocol.io) server for the live
TRDEFI non-custodial stablecoin liquidity catalogue.

**Endpoint:** `https://yield.trdefi.com/mcp` · **Transport:** Streamable HTTP · **Credential:** none

Add it to any MCP-capable client and the ten tools below appear. There is nothing to install and no
key to request. Nine are reads. The tenth, `trdefi_fusion_submit`, only forwards a signature the
caller already made - it never signs, never holds a key, and cannot move a token on its own.

This repository holds only the registry manifest (`server.json`). The server itself runs on the same
host as the public API it reads.

## Tools

| Tool | Answers |
|---|---|
| `trdefi_stats` | How much liquidity is open, across how many networks, and how much has settled over 1, 7 and 30 days |
| `trdefi_chains` | Which networks are live, their engine addresses, and the tokens a position can be built from |
| `trdefi_positions` | The open positions — filter by network and by pair |
| `trdefi_position_detail` | One position, by its strategy hash |
| `trdefi_position_inventory` | Whether a position can actually be filled - strategy balance, maker wallet balance, allowance, read from the chain |
| `trdefi_quote` | A swap simulated on-chain against one position, at the current block |
| `trdefi_fusion_quote` | A 1inch Fusion quote for a pair - the path for positions that answer "Strategy is gated" |
| `trdefi_fusion_prepare` | The EIP-712 Fusion order to sign, with its typed data, extension and quote id |
| `trdefi_fusion_submit` | Forwards an already-signed Fusion order to 1inch's relayer |
| `trdefi_badge` | A shields.io payload, so a README can show live figures |

## How a position is bound, and how the swap completes

A position is not a deposit and not a pool you join. The maker signs a **bounded, revocable
allowance**: one token, one amount, one deadline. Nothing is deposited, nothing is locked, the
capital never leaves the maker's wallet, and the exit is a single transaction.

An agent does not manage a position and does not need to work out what the maker's two tokens are.
It asks for a position to trade against, gets a price for the swap it wants, and the engine settles
the swap on its own once the swap is signed.

A maker may gate a position on the caller holding an access licence token, and on most networks
most of them do: measured 2026-10-19 across 17 networks, 62 percent answered "Strategy is gated"
and 17 percent returned a price. On Ethereum, 58 of 60 sampled positions are gated. The credential
contract is only deployed on the 13 Aqua chains, so Arc, Monad, Cronos and HyperEVM carry no such
gate. `quote_ready` narrows to positions on a network with a quote route, but it does not guarantee
a price.

Pass `from` to `trdefi_quote` with the address that will sign the swap. The gate is checked on
`tx.origin`: without a caller the API answers as `address(0)` and returns the least specific
revert available, `0x89c62b64`. With one it returns `0x39c4052c`, which names the actual problem.
A caller holding the maker's credential gets a real price; anyone else gets an accurate "you do
not hold it".

For the live counts, read `trdefi_stats`. They refresh on a timer, so nothing here hardcodes them.

## What an agent can do with it

* Understand the liquidity landscape before proposing anything: which networks, which pairs, how deep
* Find a specific position and inspect it
* Price a swap, or learn why one cannot be priced — a maker may gate a position on the caller holding
  an access licence token, and those answer with that reason rather than a number

## What it deliberately does not do

**Creating a position and swapping are absent.** Both return *unsigned* transactions and need a
signing flow that has not been designed. Shipping a tool an agent cannot finish would only teach it
that we are broken.

The underlying API is **prepare-only** in any case: it never signs, never broadcasts and never holds a
key. Funds stay in the owner's wallet behind a bounded, revocable allowance.

## Connecting

```json
{
  "mcpServers": {
    "trdefi": {
      "type": "streamable-http",
      "url": "https://yield.trdefi.com/mcp"
    }
  }
}
```

## Links

* API reference — https://yield.trdefi.com/docs/api
* OpenAPI 3.1 — https://yield.trdefi.com/openapi.json
* Agent index — https://yield.trdefi.com/llms.txt
* Live statistics — https://yield.trdefi.com/stats.html

## Listings

* Official MCP Registry — `io.github.TRDEFI/liquidity`
* Glama — https://glama.ai/mcp/connectors/io.github.TRDEFI/liquidity
* Smithery — https://smithery.ai/server/@trdefi/liquidity
* mcpservers.org — https://mcpservers.org/servers/trdefi/mcp-server
* M8ven — https://m8ven.ai/mcp/trdefi-mcp-server-wq7ly5
* Coinbase x402 Bazaar — discoverable as a paid x402 service (the two prepare endpoints answer `402`)
* PayAPI Market — https://payapi.market/api/trdefi-liquidity
  (listing live; `POST /v1/positions` carries a settled, verified payment — 0.02 USDC on Base, block 52475755)
  Agents can search it at https://payapi.market/mcp
* Payforapi — https://payforapi.com/catalog/trdefi-liquidity-positions
  (both prepare endpoints listed; 402 verified live, 0.02 USDC on Base and Arc; cards in English and Russian)
  Machine-readable catalogue at https://payforapi.com/llms-full.txt

## Registry

Published to the official MCP Registry as `io.github.TRDEFI/liquidity`.

## License

All rights reserved.
