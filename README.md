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

## Login (no password)

momo uses password + CAPTCHA / OTP, so the CLI never handles credentials.

```bash
momo login                    # = --from-browser: copy the session from Edge on CDP 9222.
                              #   Not signed in? opens the login page in a background tab
                              #   and waits (--wait 300) until you sign in yourself
momo login --from-clipboard   # copy DevTools' "Cookie" request header first
momo login --cookie-stdin     # or paste / pipe it
momo whoami                   # verifies the SAVED session over plain HTTP
momo logout
```

The session is verified with `api/shoppingcart/query/getEachCartPICount` before saving to `~/.config/momo/state.json` (0600, moved by `MOMO_HOME`) or the OS keyring with `--keyring`. `MOMO_COOKIE` overrides both.

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
