# Bitcoin (XBT) Clock + Hybrid Mempool

<p align="center">
  <img src="assets/logo.png" alt="Bitcoin (XBT) Hybrid Mempool logo" width="120">
</p>

A live single-page dashboard for **Bitcoin BLAKE2b (XBT)** — the BIP-110 hardfork that split from Bitcoin mainnet at block 961,632 and has run BLAKE2b proof-of-work since block 961,640 (30 Aug 2026).

**Live:** https://xbtclock.online/

![status](https://img.shields.io/badge/status-live-brightgreen) ![type](https://img.shields.io/badge/type-single--file%20HTML-blue)

<p align="center">
  <img src="assets/screenshot.png" alt="Bitcoin (XBT) Hybrid Mempool screenshot" width="800">
</p>

---

## What it shows

**Latest Blocks** — a horizontal strip of recent blocks at the top of the page, flat cards showing height, fee range, block reward, tx count, age, and mining pool, with a pulsing "pending" card showing live mempool size and estimated time to the next block. Click any card to open a full block-detail modal — height, hash, size/weight, difficulty, pool, fee range, total fees, reward, merkle root — with a **Transactions** tab listing every tx in that block as its own card: timestamp, every input and output address (clickable, with amounts), the coinbase transaction called out distinctly with the block's mining pool name rather than blending in as a plain row, fee/fee-rate/USD, and total output value, with a "show all N remaining" expand for transactions with many inputs or outputs. From there you can drill all the way down: block → transaction → address (balance, total received/sent, tx count) — all off `mempool.guide`'s public REST API, no node required — and a **← Back** button in the modal header (present on every tab) steps back through that trail one level at a time.

- **Center ring** — chain height (large), subsidy, current transaction fee rate, and time since the last block (colored green/amber/red at the same 10/20-minute thresholds as the ring itself), plus the last block's mining pool and a "Next Adjust" line (difficulty % change with an up/down arrow, colored green/red, and the estimated retarget date). The ring itself plots recent blocks around a selectable time window (1h / 6h / 12h / 24h / 7d, defaults to 6h), colored by how their interval compared to the 600s protocol target — green (on target), amber (slow), red (very slow) — with a "now" dial and a scrub slider to rewind through loaded history. When a new block lands, the chain-height number flashes and thin ripples travel outward from center to the ring's edge.
- **Corners** — hashrate, current difficulty, coins mined _since the fork_ (as a % of what's left to mine post-fork), halving era, live prices from Neoxa (XBT/USDC) and NonKYC (XBT/USDT), market cap (full circulating supply since Bitcoin genesis × price), blocks to halving, actual vs. expected blocks mined in the last 24 hours, local time in a selectable time zone (searchable, grouped by region), and average block time / expected next block.
- **Sound** — an optional chime (or tick/blip/thump) plays when a new block lands, synthesized with WebAudio rather than shipped as audio files. Toggle/volume in the header, preference remembered across visits.
- **Since Last Block** — an elapsed timer plus a feed of the most recent blocks (up to 30, scrollable) with pool, size, and tx count.
- **Miner Propagation Report** — pool/coinbase distribution over the last 100 blocks, 1 day, or 1 week, with a proportional bar per miner.
- **Recent Transactions** — the latest mempool transactions with USD value, XBT amount, and fee rate; click any TXID to open its detail in the same modal the Latest Blocks uses.
- **Difficulty Adjustment** — progress bar toward the next retarget, current average block time, the current and previous % change, and the estimated retarget date.
- **Transaction Fees** — the four priority tiers (No/Low/Medium/High) in sat/vB with a USD estimate for a typical transaction.
- **Mempool** — minimum fee, memory usage, unconfirmed tx count, and a live-accumulated Incoming Transactions chart (no history endpoint — built from the page's own polling over time).
- **Recent Replacements** — RBF (replace-by-fee) chains, showing the previous vs. new fee rate and whether the replacement has been mined; TXIDs are clickable too.
- **XBT Network Nodes** — confirmed reachable node count, peers probed, and IPv6 share, plus a history chart tracking the confirmed count over time. Fed by a separate project, [xbt-crawler](../xbt-crawler) — a P2P network crawler seeded from a local XBT node's RPC that fingerprints peers by chain data (not user-agent, which isn't reliable for a fork sharing Bitcoin mainnet's magic bytes/port) — not a live API. See [Node data](#node-data) below.

On screens under ~820px wide, the header collapses behind a hamburger menu (status, time zone, sound, refresh) so it never overlaps or overflows on phones/tablets. Loaded block history is cached in `localStorage`, so a manual browser refresh doesn't force a full multi-page backfill again — it picks up where it left off and just fetches whatever's new. The node-count history chart is *not* `localStorage`-based — it's built from a published history file (see [Node data](#node-data)) so every visitor sees the same trend, not just whatever a given browser has locally accumulated.

No build step, no framework, no dependencies — it's one `.html` file. Open it locally or host it anywhere that serves static files.

## Data sources

| Source                                            | What it provides                                                                                                                                                                                                                                         | Auth |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| [mempool.guide](https://mempool.guide)            | Chain tip, blocks, difficulty adjustment, hashrate, mempool stats, fee estimates, recent transactions, RBF replacements, and (for the Latest Blocks modals) individual block/transaction/address lookups — all via the mempool.space-compatible REST API | None |
| [neoxa.exchange](https://neoxa.exchange/api-docs) | XBT/USDC ticker (their API's internal query parameter is `BTCB2_USDC` — a technical quirk, not the ticker)                                                                                                                                               | None |
| [api.nonkyc.io](https://api.nonkyc.io)            | XBT/USDT ticker (internal query parameter `BTCB2_USDT`)                                                                                                                                                                                                  | None |

All three sources are public/no-auth, but **none of them send permissive CORS headers**, so a browser can't call them directly from a page hosted on a different origin. This project routes every request through a small Cloudflare Worker that fetches server-side and re-adds `Access-Control-Allow-Origin: *`.

## Node data

The XBT Network Nodes panel doesn't call a live API — `mempool.guide` and friends only expose chain/mempool data, not a peer-count endpoint, and no such service exists for this chain (that's the reason the panel exists at all). Instead it fetches two same-origin static files, published manually alongside `index.html`:

| File                 | What it is                                                                       | Fetched by            |
| --------------------- | --------------------------------------------------------------------------------- | ---------------------- |
| `crawl-latest.json`   | The most recent crawl snapshot — current stats and the confirmed-peer list        | `fetchNodeLatest()`    |
| `crawl-history.json`  | A compact rolling history, one entry per crawl run, capped at 365 entries         | `fetchNodeHistory()`   |

Both are produced by [xbt-crawler](../xbt-crawler), a separate dependency-free Node.js project that:

1. Connects to a local XBT node's RPC to seed a peer list (`getpeerinfo` + `getnodeaddresses`)
2. Crawls the P2P network from there, speaking the Bitcoin wire protocol directly over raw sockets
3. Fingerprints each peer by requesting a known post-fork block via `getdata` — `block` reply confirms XBT, `notfound` means it's just a real Bitcoin/Knots node that happens to share the same magic bytes and port (necessary because BTCB2 inherited Bitcoin mainnet's network identity rather than defining its own — the `version` handshake alone can't tell the two apart)
4. Writes `crawl-latest.json` and appends one entry to `crawl-history.json`

Running `node publish.js` in that project does the crawl and pushes both files into this repo in one step. Neither file updating on any particular schedule is expected — the panel just shows whatever was last published, and stays in its default `—` state (with an empty history chart) until the files exist at all.

If you're not running the crawler, delete or leave the panel's `<div class="panel" id="nodePanel">` block out of `index.html` — it fails closed (silently) rather than showing broken/fake data.

## Setup

1. **Deploy the CORS proxy.** Create a Cloudflare Worker (see [Capacity](#capacity) below for which plan fits) with this code:

   ```js
   export default {
     async fetch(request) {
       const url = new URL(request.url).searchParams.get("url");
       if (!url) return new Response("Missing ?url=", { status: 400 });
       const upstream = await fetch(url, {
         headers: { "User-Agent": "Mozilla/5.0" },
       });
       const body = await upstream.arrayBuffer();
       return new Response(body, {
         status: upstream.status,
         headers: {
           "Access-Control-Allow-Origin": "*",
           "Content-Type":
             upstream.headers.get("content-type") || "application/json",
         },
       });
     },
   };
   ```

2. **Point the page at it.** In `index.html`, set:

   ```js
   const PROXY_PREFIX = "https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/?url=";
   ```

3. **Serve the HTML file.** Any static host works (Netlify, GitHub Pages, Cloudflare Pages, or just open the file locally) — there's nothing to build.

4. **(Optional) Publish node data.** For the XBT Network Nodes panel to show real numbers, run `node publish.js` from the [xbt-crawler](../xbt-crawler) project — it crawls the network and pushes `crawl-latest.json` + `crawl-history.json` into this repo. Skip this step entirely if you don't want that panel; it degrades gracefully with no data published. See [Node data](#node-data) below.

## Configuration

The important constants live in `CONFIG` near the top of the `<script>` block:

| Constant                                           | Meaning                                                                                                                |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `FORK_HEIGHT`                                      | 961640 — the clock's "zero point" (BLAKE2b activation), not true Bitcoin genesis                                       |
| `TARGET_BLOCK_SECONDS`                             | 600 — the inherited Bitcoin protocol target, used for the ring's on-target/slow/very-slow coloring                     |
| `SUPPLY_CAP`                                       | 21,000,000 — inherited issuance cap                                                                                    |
| `BLOCK_PAGES_ON_LOAD` / `BLOCK_PAGES_MAX_BACKFILL` | How much block history is fetched on load and how far it'll extend backward to fill longer ring windows (up to 7 days) |
| `REFRESH_CHAIN_MS` / `REFRESH_MARKET_MS`           | Polling intervals (5s / 5s by default)                                                                                 |

The Latest Blocks strip renders off the same `state.blocks` array the ring uses (capped to the most recent 150 cards) — it doesn't make any extra requests of its own. Opening a modal (block/transaction/address) does fetch on demand, only when you click something.

Two more constants, defined near `CONFIG` but outside it (same pattern as `PROXY_PREFIX`), control the node panel:

| Constant                | Meaning                                                                 |
| ------------------------ | ------------------------------------------------------------------------ |
| `NODE_CRAWL_URL`          | Path to `crawl-latest.json`, default `./crawl-latest.json`               |
| `NODE_HISTORY_URL`        | Path to `crawl-history.json`, default `./crawl-history.json`             |
| `NODE_CRAWL_REFRESH_MS`   | How often the page re-polls both files (5 min default — crawl data is slow-moving, no need to match the 5s chain refresh) |

## Capacity

At the default 5-second refresh intervals, one continuously-open tab makes roughly **104,000 requests/day** through the Worker — that's already past Cloudflare's free-plan cap of 100,000 requests/day with just one viewer. This project runs on **Workers Paid** ($5/mo, 10M requests included), which comfortably covers that. If you'd rather stay on the free plan, lengthen `REFRESH_CHAIN_MS`/`REFRESH_MARKET_MS` (10-15s still feels quite live and cuts the volume by half to two-thirds) instead of upgrading. Modal lookups (block/tx/address detail) are on-demand and add negligible extra load.

## Notes

- This chain does **not** reset to a fresh genesis like some other forks do — it inherits Bitcoin's full pre-fork history, so block height continues from real Bitcoin mainnet numbering (~970,000+) rather than starting over at 0.
- The "Mined post-fork" stat and market cap are deliberately different numbers: post-fork supply excludes everything inherited from before the fork, while market cap values the full circulating supply since true genesis, since every coin on the chain is spendable regardless of which side of the fork it was mined on.
- The Latest Blocks cards and modals get their aggregate data (fee ranges, totals, pool attribution) from fields `mempool.guide` already computes server-side (`extras.feeRange`, `extras.totalFees`, `extras.pool`, etc.) — nothing is fetched per-transaction just to populate the strip itself.
- The coinbase transaction in each block's Transactions tab is detected via `vin[0].is_coinbase` and shown with the block's own pool attribution rather than an input list (it has no real inputs to show).
- Block-level extras (pool name, fee range) require the `/v1/` prefixed endpoint specifically — the plain `/block/:hash` endpoint doesn't include them.
- API endpoint field names for NonKYC were reverse-engineered against their public v2 market endpoint rather than sourced from published docs — if their schema changes, that fetch degrades gracefully to `—` rather than breaking the page.
- The node count shown is a **lower bound**, not a guaranteed total — it's whatever the crawler's last run could actually reach and confirm. Since BTCB2 shares Bitcoin mainnet's magic bytes and port (see above), most alive peers a crawl encounters are real Bitcoin/Knots nodes, not XBT ones — this is expected, not a bug in the confirmed count.

## License

[MIT with Attribution](LICENSE) — free to use, modify, and deploy, but any fork or deployment must visibly credit the original author and link back to this repo.
