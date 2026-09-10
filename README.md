# XBT Clock

A live single-page dashboard for **Bitcoin BLAKE2b (XBT)** — the BIP-110 hardfork that split from Bitcoin mainnet at block 961,632 and has run BLAKE2b proof-of-work since block 961,640 (30 Aug 2026).

**Live:** https://xbtclock.online/

![status](https://img.shields.io/badge/status-live-brightgreen) ![type](https://img.shields.io/badge/type-single--file%20HTML-blue)

---

## What it shows

- **Center ring** — chain height (large) with blocks-since-fork underneath, subsidy, difficulty adjustment, and time since the last block. The ring itself plots recent blocks around a selectable time window (1h / 6h / 12h / 24h / 7d), colored by how their interval compared to the 600s protocol target — green (on target), amber (slow), red (very slow) — with a "now" dial and a scrub slider to rewind through loaded history.
- **Corners** — hashrate, coins mined *since the fork* (as a % of what's left to mine post-fork), halving era, live prices from Neoxa (XBT/USDC) and NonKYC (XBT/USDT), market cap (full circulating supply since Bitcoin genesis × price), blocks to halving, the last block's mining pool, local time in a selectable time zone, and average block time / expected next block.
- **Since Last Block** — an elapsed timer plus a feed of the most recent blocks with pool, size, and tx count.
- **Miner Propagation Report** — pool/coinbase distribution over the last 100 blocks, 1 day, or 1 week, with a proportional bar per miner.

No build step, no framework, no dependencies — it's one `.html` file. Open it locally or host it anywhere that serves static files.

## Data sources

| Source | What it provides | Auth |
|---|---|---|
| [mempool.guide](https://mempool.guide) | Chain tip, blocks, difficulty adjustment, hashrate (mempool.space-compatible REST API) | None |
| [neoxa.exchange](https://neoxa.exchange/api-docs) | XBT/USDC ticker (their API's internal query parameter is `BTCB2_USDC` — a technical quirk, not the ticker) | None |
| [api.nonkyc.io](https://api.nonkyc.io) | XBT/USDT ticker (internal query parameter `BTCB2_USDT`) | None |
| [mempool.space](https://mempool.space) | Reference BTC/USD price (optional, currently unused in the UI) | None |

All three primary sources are public/no-auth, but **none of them send permissive CORS headers**, so a browser can't call them directly from a page hosted on a different origin. This project routes every request through a small Cloudflare Worker that fetches server-side and re-adds `Access-Control-Allow-Origin: *`.

## Setup

1. **Deploy the CORS proxy.** Create a Cloudflare Worker (free tier is plenty — see [Capacity](#capacity-free-tier) below) with this code:

   ```js
   export default {
     async fetch(request) {
       const url = new URL(request.url).searchParams.get('url');
       if (!url) return new Response('Missing ?url=', { status: 400 });
       const upstream = await fetch(url, { headers: { 'User-Agent': 'Mozilla/5.0' } });
       const body = await upstream.arrayBuffer();
       return new Response(body, {
         status: upstream.status,
         headers: {
           'Access-Control-Allow-Origin': '*',
           'Content-Type': upstream.headers.get('content-type') || 'application/json',
         },
       });
     },
   };
   ```

2. **Point the clock at it.** In `index.html`, set:

   ```js
   const PROXY_PREFIX = 'https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/?url=';
   ```

3. **Serve the HTML file.** Any static host works (Netlify, GitHub Pages, Cloudflare Pages, or just open the file locally) — there's nothing to build.

## Configuration

The important constants live in `CONFIG` near the top of the `<script>` block:

| Constant | Meaning |
|---|---|
| `FORK_HEIGHT` | 961640 — the clock's "zero point" (BLAKE2b activation), not true Bitcoin genesis |
| `TARGET_BLOCK_SECONDS` | 600 — the inherited Bitcoin protocol target, used for the ring's on-target/slow/very-slow coloring |
| `SUPPLY_CAP` | 21,000,000 — inherited issuance cap |
| `BLOCK_PAGES_ON_LOAD` / `BLOCK_PAGES_MAX_BACKFILL` | How much block history is fetched on load and how far it'll extend backward to fill longer ring windows (up to 7 days) |
| `REFRESH_CHAIN_MS` / `REFRESH_MARKET_MS` | Polling intervals (30s / 25s by default) |

## Capacity (free tier)

At the default refresh intervals, one continuously-open tab makes roughly **19,400 requests/day** through the Worker. Cloudflare's free plan caps at 100,000 requests/day, so this comfortably supports a handful of people leaving the page open around the clock — beyond roughly 5 concurrent 24/7 viewers, either lengthen `REFRESH_CHAIN_MS`/`REFRESH_MARKET_MS`, or upgrade to Workers Paid ($5/mo, 10M requests included).

## Notes

- This chain does **not** reset to a fresh genesis like some other forks do — it inherits Bitcoin's full pre-fork history, so block height continues from real Bitcoin mainnet numbering (~970,000+) rather than starting over at 0.
- The "Mined post-fork" stat and market cap are deliberately different numbers: post-fork supply excludes everything inherited from before the fork, while market cap values the full circulating supply since true genesis, since every coin on the chain is spendable regardless of which side of the fork it was mined on.
- API endpoint field names for NonKYC were reverse-engineered against their public v2 market endpoint rather than sourced from published docs — if their schema changes, that fetch degrades gracefully to `—` rather than breaking the page.

## License

[MIT with Attribution](LICENSE) — free to use, modify, and deploy, but any fork or deployment must visibly credit the original author and link back to this repo.
