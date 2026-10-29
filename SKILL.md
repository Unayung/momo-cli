---
name: momo
description: Use when searching momo購物網 (www.momoshop.com.tw) products, checking prices/stock/specs, monitoring prices, or viewing/adding/removing items in the user's momo cart (購物車). No checkout.
---

# momo — momo購物網 from a shell (no checkout)

`~/.local/bin/momo` → `~/Projects/momo-cli/momo`. Stdlib Python 3.8+, plain HTTPS; Edge on
CDP 9222 is a fallback and source for `momo login`. Run `momo doctor` first.

Read (no login):
- `momo search <q> [--limit N] [--sort relevance|price|price-desc|sales|newest] [--min/--max TWD] [--page N] [--json]`
  → id, title, price (TWD), orig_price, discount, rating, reviews, stock, url
- `momo product <url|id> [--json]` (alias `item`) → title, brand, price, promo, in_stock,
  shipping, rating, reviews, `specs[]` with variant id / name / price / stock, url
- `momo price <id> <id> … [--json]` → batch price/stock check for price monitoring
- Product IDs are digit strings (e.g. `7252730`); full URLs (`GoodsDetail.jsp?i_code=...`) work too.

Login (never ask for or type passwords/OTP):
- `momo login` copies the session from Edge on CDP 9222 (`MOMO_CDP`). If Edge is not signed in,
  it opens the login page in a background tab and waits up to `--wait 300`s while the USER signs in.
- `momo whoami` verifies the saved session over plain HTTP; exit 3 = not signed in → run `momo login`.
- Fallbacks: `momo login --from-clipboard` / `--cookie-stdin` (DevTools Cookie header).

Cart (writes need explicit user approval):
1. `momo cart list [--json]` — lines with line #, id, name, spec, qty, price, subtotal, total
2. `momo cart add <id> [--qty N] [--spec TEXT]` (dry run) → show the user → add `--yes`
3. `momo cart qty <line#|id> <N> --yes`, `momo cart rm <line#|id> --yes`
Every write re-reads the cart and verifies the change landed; never report success from POST reply alone.

Pitfalls:
- Cart mutators require valid `STORAGEINFO` and `VERSION` tokens from the cart page, saved automatically on `momo login`.
- Specs: `cart add` on multi-variant products needs `--spec <id|name>`; `product` lists all available choices.
- No checkout / payment / order placement endpoints exist in the CLI by design; never attempt order placement.
