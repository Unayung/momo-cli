# momo-cli

Agent-native CLI for momo購物網 (www.momoshop.com.tw, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, product details with specs/variants, batch price checks, and your 購物車 (list / add / rm / qty). **No checkout.**

Portable: every call is plain HTTPS to the same Next.js SSR / JSON endpoints the web app uses (measured 2026-10-29). If a read is ever bot-walled it can fall back to a background tab in a CDP browser (`MOMO_CDP`, default `127.0.0.1:9222`); `--via auto|http|browser` controls transport.

```bash
momo doctor                                  # APIs, browser, login, cart state
momo search 衛生紙 [--limit 20] [--sort relevance|price|price-desc|sales|newest]
                   [--min 500 --max 1000] [--page 2]
momo product <url|id>                        # alias: item. title, brand, price,
                                             # in_stock, shipping, promo, specs
momo price <url|id> [<url|id> ...]           # batch price/stock check (monitoring)
momo raw 'api/shoppingcart/query/getEachCartPICount'   # read-only GET passthrough
```

`--sort price` / `price-desc` are sorted client-side from clean, ad-free results:
momo's own server-side price sort was found to inject a pinned sponsor block
(same ~6 items up front regardless of sort) and `/TP/...` marketplace
listings with decoy NT$1-11 teaser prices (confirmed on the live site, not
just the scrape), so the CLI instead gathers candidates from relevance pages
(measured ad-free), drops any listing without a real product id, dedupes,
and sorts/paginates itself — giving a provably sorted, bogus-price-free
result at the cost of ranking only among relevance-discovered items rather
than momo's whole catalog (which momo's own sort can't reliably do either
for this kind of query). `relevance|sales|newest` still use momo's own
server-side order directly, with the same ad-listing exclusion applied.

All sorts, including the default `relevance`, dedupe by product id across
the backend pages fetched for a request: momo's pinned/sponsored block can
repeat a second time later within the *same* page's own listing, and leaks
across pages (a page-2 request can replay a page-1 pinned item). The CLI
fetches backend pages sequentially, keeps the first occurrence of each id,
and backfills with more backend pages until `--limit` is satisfied, so
results are guaranteed unique within a page and never repeat across
consecutive pages.

## Login (no password)

momo uses password + CAPTCHA / OTP, so the CLI never handles credentials. It logs in with the
`Cookie` request header of a browser session you already have:

1. In your browser: DevTools > Network > click any request to `www.momoshop.com.tw` > Request Headers.
2. Copy the value of `Cookie:` (with or without the leading `Cookie: `).
3. Run `momo login`. It reads the clipboard, or stdin when piped.

```bash
momo login                    # Cookie header from stdin (piped) or the clipboard;
                              #   nothing usable there = exit 2 with these steps
momo login --from-clipboard   # clipboard only
momo login --cookie-stdin     # paste or pipe it
momo login --keyring          # store it in the OS keyring instead of the file
momo login --from-browser     # opt-in: copy the session from Edge on CDP 9222; if not signed in,
                              #   opens the login page in a background tab (--wait 300)
momo whoami                   # verifies the SAVED session over plain HTTP
momo logout
```

The session is verified with `api/shoppingcart/query/getEachCartPICount` before saving to
`~/.config/momo/state.json` (0600, moved by `MOMO_HOME`) or the OS keyring with `--keyring`.
`MOMO_COOKIE` overrides both. Exit 3 means there is no valid session (or it expired): log in again.

The cart write API also sends `STORAGEINFO` and `VERSION` from the browser's localStorage, which a
Cookie header does not contain. `--from-browser` captures them; a cookie login for the same
account keeps the ones already saved.

Edge on CDP 9222 is needed only for `login --from-browser` and as the fallback when a read is
bot-walled. Setup: [Browser on CDP 9222](#browser-on-cdp-9222-setup).

## Browser on CDP 9222 (setup)

The commands listed above as needing Edge drive a background tab in your own browser through the Chrome DevTools
Protocol, so their requests carry a real browser's fingerprint and sign-in. The CLI never launches a browser;
start one yourself with remote debugging on:

```bash
# Linux (Edge; google-chrome / chromium take the same flags)
microsoft-edge-stable --remote-debugging-port=9222 --user-data-dir="$HOME/.config/microsoft-edge-debug"

# macOS
"/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Microsoft Edge Debug"
```

- `--user-data-dir` must point at a separate profile: Chromium-based browsers ignore `--remote-debugging-port` on
  the default profile (Chrome 136+). Sign in to momoshop.com.tw once inside this profile; it stays signed in across
  restarts. If that profile is already open without the flag, quit the browser first: launching again only opens
  a window in the running instance.
- Check it: `curl -s http://127.0.0.1:9222/json/version` prints the browser version, and `momo doctor` shows
  `cdp ok`. Another port or host: `--cdp http://127.0.0.1:9333` or `MOMO_CDP=http://127.0.0.1:9333`.
- `--remote-allow-origins` is not needed: the CLI connects without an `Origin` header (verified 2026-10-30), and
  leaving it out keeps web pages from connecting. The port listens on 127.0.0.1 only; keep it that way, because
  anything that can reach it controls the browser and every session in it.
- The CLI only creates background tabs (`Target.createTarget(background=true)`), never focuses or touches yours,
  closes its own when done, and blanks rather than closes the last tab so the browser does not quit. Run one
  command at a time.

## Cart (購物車)

```bash
momo cart list                                  # lines, spec, qty, price, subtotal, total
momo cart add <url|id> [--qty 2] [--spec 001] --yes
momo cart qty <line#|id> 3 --yes
momo cart rm  <line#|id> --yes
```

Without `--yes`, `cart add/qty/rm` perform a dry run and output what would be executed. After every write, the cart is re-read and confirmed; a reply without a visible change raises an error. Products with multiple specs (size/color) need `--spec <text or id>`; `momo product` lists available choices. Cart commands always use the saved session.

Every command takes `--json` → `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

No checkout, payment or order placement by design. Unofficial private API — may break anytime.
