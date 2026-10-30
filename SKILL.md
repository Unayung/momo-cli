---
name: momo
description: Use when searching momo購物網 (www.momoshop.com.tw) products, checking prices/stock/specs, monitoring prices, or viewing/adding/removing items in the user's momo cart (購物車). No checkout.
---

# momo — momo購物網 from a shell (no checkout)

`~/.local/bin/momo` → `~/Projects/momo-cli/momo`. Stdlib Python 3.8+, plain HTTPS; Edge on
CDP 9222 is a read fallback and the opt-in `momo login --from-browser`. Run `momo doctor` first.

Read (no login):
- `momo search <q> [--limit N] [--sort relevance|price|price-desc|sales|newest] [--min/--max TWD] [--page N] [--json]`
  → id, title, price (TWD), orig_price, discount, rating, reviews, stock, url
- `momo product <url|id> [--json]` (alias `item`) → title, brand, price, promo, in_stock,
  shipping, rating, reviews, `specs[]` with variant id / name / price / stock, url
- `momo price <id> <id> … [--json]` → batch price/stock check for price monitoring
- Product IDs are digit strings (e.g. `7252730`); full URLs (`GoodsDetail.jsp?i_code=...`) work too.
- `--sort price`/`price-desc` are sorted client-side from relevance-pool results: momo's own
  price sort mixes in a pinned sponsor block (same items up front regardless of sort) and
  `/TP/...` marketplace ads with decoy NT$1-11 prices, confirmed on the live site itself — the
  CLI drops those and sorts real listings itself, so results may rank only among
  relevance-discovered items rather than momo's full catalog.
- All sorts (including default `relevance`) dedupe by product id across the backend pages
  fetched per request: momo's pinned block can repeat a second time later in the *same* page,
  and leaks across pages (page 2 can replay a page-1 item). Fetches accumulate backend pages
  until `--limit` unique ids are collected, so a page is always unique and never repeats the
  previous page.

Login (never ask for or type passwords/OTP, and never ask the user to paste cookies into chat):
- Tell the user: DevTools > Network > click any www.momoshop.com.tw request > copy the `Cookie:`
  request header to the clipboard, then run `momo login` themselves (or pipe it in on stdin).
  No usable Cookie header = exit 2 with those steps.
- `momo whoami` verifies the saved session over plain HTTP; exit 3 = not signed in → run `momo login`.
- `login --from-browser` is opt-in only (Edge on CDP 9222, `MOMO_CDP`; waits up to `--wait 300`s for
  the USER to sign in). It is also the only way to capture the cart tokens (STORAGEINFO/VERSION)
  for a new account. `--from-clipboard` / `--cookie-stdin` also work.

Cart (writes need explicit user approval):
1. `momo cart list [--json]` — lines with line #, id, name, spec, qty, price, subtotal, total
2. `momo cart add <id> [--qty N] [--spec TEXT]` (dry run) → show the user → add `--yes`
3. `momo cart qty <line#|id> <N> --yes`, `momo cart rm <line#|id> --yes`
Every write re-reads the cart and verifies the change landed; never report success from POST reply alone.

Pitfalls:
- Cart mutators require valid `STORAGEINFO` and `VERSION` tokens from the cart page, saved by `momo login --from-browser` (a cookie login keeps the saved ones for the same account).
- Specs: `cart add` on multi-variant products needs `--spec <id|name>`; `product` lists all available choices.
- No checkout / payment / order placement endpoints exist in the CLI by design; never attempt order placement.
