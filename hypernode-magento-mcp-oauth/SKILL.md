---
name: Set Up Magento MCP OAuth Discovery on Hypernode
description: Fix and configure OAuth 2.0 discovery (RFC 8414 / RFC 9728) for the magebitcom/magento2-mcp-module on a Hypernode-hosted Magento store, so Claude (or any MCP OAuth client) can connect. Use this skill whenever someone is installing the Magebit Magento MCP module on Hypernode, connecting Claude/Claude Desktop to a Magento MCP server, or debugging any of: /.well-known/oauth-authorization-server or /.well-known/oauth-protected-resource returning 404 or HTML instead of JSON, an OAuth connector falling back to a bare /authorize (404) instead of /mcp/oauth/authorize, an MCP OAuth connect flow failing with 429 Too Many Requests, or a Claude connector stuck/failing after nginx changes on Hypernode. Also use it for general Hypernode nginx questions about the split between the SSL-terminating vhost and the Varnish-origin vhost, or about the nginx_config_reloader daemon and /etc/nginx/app being overwritten.
---

# Set Up Magento MCP OAuth Discovery on Hypernode

You make the Magebit Magento MCP module's OAuth discovery endpoints work
correctly on a Hypernode instance, and connect a Claude MCP connector to it.
This was derived by debugging the exact failure end-to-end on a live
Hypernode instance — every step below fixes a real, previously-observed
failure mode, not a hypothetical one.

Package: [`magebitcom/magento2-mcp-module`](https://github.com/magebitcom)
(plus its sibling tool packages: `-catalog-tools`, `-cms-tools`,
`-customer-tools`, `-db-tools`, `-marketing-tools`, `-order-tools`,
`-report-tools`, `-tax-tools`). It exposes Magento over MCP at `/mcp` and
implements OAuth 2.0 with discovery per RFC 8414 (authorization server
metadata) and RFC 9728 (protected resource metadata).

Throughout, `example.com` stands for the real domain and `<vhost>` for the
real vhost directory name under `/data/web/nginx/` (usually the domain
itself). All commands assume SSH access to the Hypernode as the `app` user.

## Step 0 — confirm the module itself works before touching nginx

```bash
curl -s https://example.com/mcp/oauth/authorizationservermetadata | head -c 300
curl -s https://example.com/mcp/oauth/protectedresourcemetadata | head -c 300
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/mcp/oauth/authorize   # expect 400 (missing params), not 404
```
If these don't return correct JSON / 400, the problem is in the module or
Magento config, not nginx — stop here and fix that first. Everything below
assumes these already work and only `/.well-known/*` is broken.

## Step 1 — understand the Hypernode nginx architecture (don't skip this)

Every mistake in the original debugging session traced back to not knowing
this up front:

- `/etc/nginx/app/` and `/etc/nginx/app_bak/` are **generated, root-owned**
  output. A daemon called `nginx_config_reloader` watches the user-writable
  `/data/web/nginx/` and re-renders `/etc/nginx/app/` from it on every
  change, then reloads nginx automatically. **Never edit anything under
  `/etc/nginx/app/` directly** — the next sync silently overwrites it and
  rotates the edit into `/etc/nginx/app_bak/`, with no error to warn you.
  Always edit under `/data/web/nginx/<vhost>/` (or `/data/web/nginx/` root
  for platform-wide files like `http.ratelimit`, see Step 4).
- After every change, confirm it actually synced:
  ```bash
  journalctl -u nginx-config-reloader -n 10 --no-pager   # look for "MODIFIED ... Applying new config"
  diff /data/web/nginx/<vhost>/yourfile.conf /etc/nginx/app/<vhost>/yourfile.conf   # should be identical
  ```
- **There are two separate nginx server blocks per vhost, and PHP only runs
  in one of them:**
  1. `/etc/nginx/sites/https.<vhost>.conf` (ports 443/8443) terminates TLS
     and does **not** run PHP. Its `location /` (from
     `/data/web/nginx/<vhost>/public.magento2.conf`) just `proxy_pass`es
     everything to Varnish on `127.0.0.1:6081`. It includes
     `/etc/nginx/app/<vhost>/server.*` and `.../public.*`.
  2. `/etc/nginx/sites/varnish.<vhost>.conf` (`127.0.0.1:8080`) is the actual
     PHP-serving origin Varnish forwards to. It includes
     `/etc/nginx/app/<vhost>/varnish.*`; `varnish.webroot.conf` in there is
     what has `location ~ \.php$ { echo_exec @phpfpm; }`.

  A file matching `server.*` in `/data/web/nginx/<vhost>/` loads **only** in
  vhost (1); a file matching `varnish.*` loads **only** in vhost (2). If you
  put a `rewrite ... /index.php last;` in a `server.*` file, it re-matches
  locations *within vhost (1)*, finds no PHP handler, falls through to the
  generic `proxy_pass`, and forwards the literal string `/index.php` (the
  real path is lost) to Varnish — which Magento renders as a real,
  200-status **homepage**, with no redirect or error to reveal what
  happened. This is the single most time-consuming failure mode to debug
  blind; the fix in Step 3 avoids it by construction.

## Step 2 — understand why `/.well-known/*` 404s by default

`/etc/nginx/security_locations.conf` (root-owned, shared platform-wide, do
not edit) has:
```nginx
location ~ ^/\.well-known {
}
```
An empty regex location, there so `.well-known/acme-challenge` and
`.well-known/pki-validation` (handled by more specific `^~` locations
elsewhere) don't fall through to a "block dotfiles" 403. Any other
`.well-known/*` path matches this regex and gets nginx's bare 404, never
reaching PHP. `.well-known/acme-challenge` must keep working — the fix below
only targets the two OAuth-specific paths.

## Step 3 — deploy the fix: two files, one per vhost

Both locations use `^~` to take priority over the empty regex from Step 2,
scoped to exactly these two paths.

**`/data/web/nginx/<vhost>/server.mcp-oauth.conf`** (SSL vhost — proxy
through to Varnish, no rewrite):
```nginx
location ^~ /.well-known/oauth-authorization-server {
    set $log_handler varnish;
    proxy_pass http://127.0.0.1:6081;
    proxy_read_timeout 900s;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $real_scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header Host $http_host;
}
location ^~ /.well-known/oauth-protected-resource {
    set $log_handler varnish;
    proxy_pass http://127.0.0.1:6081;
    proxy_read_timeout 900s;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $real_scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header Host $http_host;
}
```
Before deploying, `cat /data/web/nginx/<vhost>/public.magento2.conf` on the
target instance and make sure the `proxy_set_header` lines above match it
exactly — copy any extra/different headers that vhost uses.

**`/data/web/nginx/<vhost>/varnish.mcp-oauth.conf`** (origin vhost — hand off
to PHP; this is the one that's safe to `rewrite` in, because
`varnish.webroot.conf` in this same vhost has the `\.php$` handler):
```nginx
location ^~ /.well-known/oauth-authorization-server {
    rewrite ^ /index.php last;
}
location ^~ /.well-known/oauth-protected-resource {
    rewrite ^ /index.php last;
}
```

Deploy both, then confirm sync per Step 1:
```bash
scp server.mcp-oauth.conf  app@HOST:/data/web/nginx/<vhost>/server.mcp-oauth.conf
scp varnish.mcp-oauth.conf app@HOST:/data/web/nginx/<vhost>/varnish.mcp-oauth.conf
ssh app@HOST 'journalctl -u nginx-config-reloader -n 10 --no-pager'
```

If anything was hit successfully-but-wrong before this fix (e.g. served the
homepage), purge the stale cached copy:
```bash
ssh app@HOST "varnishadm ban 'req.url ~ ^/\.well-known/'"
ssh app@HOST "cd /data/web/current && php bin/magento cache:clean full_page config"
```

Verify:
```bash
for p in oauth-authorization-server oauth-authorization-server/mcp \
         oauth-protected-resource oauth-protected-resource/mcp; do
  echo "== $p =="; curl -s -D - -o /tmp/r.txt -H 'Cache-Control: no-cache' \
    "https://example.com/.well-known/$p?cbust=$(date +%s%N)" | grep -iE '^HTTP|content-type'
  head -c 300 /tmp/r.txt; echo; echo
done
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/                              # 200
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/.well-known/acme-challenge/x  # 404, untouched
```
Every `.well-known` variant must return `HTTP 200`, `content-type:
application/json`, real JSON (not HTML), and the authorization-server
response's `authorization_endpoint` must equal
`https://example.com/mcp/oauth/authorize`.

## Step 4 — the rate-limiter gotcha (do this even if Step 3's curl tests pass)

`curl` is invisible to Hypernode's bot rate limiter, but **Claude's actual
OAuth client is not**. Anthropic's MCP connector sends `User-Agent:
python-httpx/<version>`. Hypernode's default bot-denylist regex
(`~*(http|crawler|spider|bot|search|...)`) matches the substring `http`
inside `httpx`, so real discovery requests from Claude get rate-limited to 1
request/second — and the connect flow bursts several in a row (protected
resource + authorization server, each renegotiated over HTTP/1.1 and
HTTP/2), tripping the limit immediately.

**Symptom:** your own curl tests from Step 3 all pass, but Claude's actual
connection attempt still fails — either its connector debug log shows `429`,
or it silently falls back to guessing a default (wrong, 404ing) `/authorize`
path instead of using the discovered `/mcp/oauth/authorize`.

**Diagnose:**
```bash
ssh app@HOST 'grep well-known /var/log/nginx/access.log | tail -30'          # status 429, user_agent containing httpx/python
ssh app@HOST "grep -E 'limiting (requests|connections)' /var/log/nginx/error.log | tail -20"  # confirms zone "bots"
```

**Fix** — Hypernode's own documented override mechanism (see
https://docs.hypernode.com/hypernode-platform/nginx/how-to-resolve-rate-limited-requests-429-too-many-requests.html).
Edit `/data/web/nginx/http.ratelimit` (user-writable; lives at the nginx
root, **not** per-vhost — don't duplicate it into `<vhost>/`). Read the
current content first — it may already carry other customizations (e.g.
per-IP overrides) that must be preserved, not overwritten:
```bash
ssh app@HOST cat /data/web/nginx/http.ratelimit
```
Add (or extend) a `$limit_bots` map with `python-httpx` in the allowlist
alternation, keeping every existing entry — check
`/etc/nginx/conf.d/web.conf` for the live default list if `http.ratelimit`
doesn't already override it:
```nginx
map $http_user_agent $limit_bots {
    default '';
    ~*(google|bing|heartbeat|uptimerobot|shoppimon|monitis.com|Zend_Http_Client|magereport.com|SendCloud/|Adyen|ForusP|contentkingapp|node-fetch|Hipex|Hypernode|xCore|Mollie|python-httpx) '';
    ~*(http|crawler|spider|bot|search|Wget/|Python-urllib|PHPCrawl|bGenius|MauiBot|aspiegel|facebookexternal) 'bot';
}
```
(Adjust both lists to whatever is actually live on the target instance —
only *add* `python-httpx` to the allowlist, don't replace the lists
wholesale.)

Back up, deploy, confirm sync, then burst-test with the real User-Agent:
```bash
ssh app@HOST cp /data/web/nginx/http.ratelimit /data/web/nginx/http.ratelimit.bak-$(date +%F)
scp http.ratelimit app@HOST:/data/web/nginx/http.ratelimit
ssh app@HOST 'journalctl -u nginx-config-reloader -n 6 --no-pager'

for i in 1 2 3 4 5; do
  curl -s -o /dev/null -w "req$i: HTTP %{http_code}\n" -A "python-httpx/0.28.1" \
    "https://example.com/.well-known/oauth-authorization-server?b=$i"
done   # all 5 should be 200, not 429
```

This exemption is **site-wide and keyed on user-agent**, not scoped to a
path — intentional, since the same client will also hit
`/mcp/oauth/authorize` and `/mcp/oauth/token` later in the same flow and
needs the same exemption there too. If a narrower, path-only exemption is
preferred instead, bypass `@fastcgi_backend`'s `limit_req` by inlining the
fastcgi_pass config directly into the two `varnish.mcp-oauth.conf` locations
from Step 3 (copy the relevant lines from `/etc/nginx/handlers.conf`'s
`@fastcgi_backend`, omitting only `limit_req zone=bots;`) — but the
allowlist approach is Hypernode's supported mechanism and is what this skill
defaults to.

## Step 5 — connect from Claude

1. In Magento admin, under the MCP module's OAuth Clients grid, create a
   client and note the client ID/secret (or use dynamic client registration
   if supported).
2. In Claude Desktop/claude.ai: Settings → Connectors → Add custom
   connector. URL: `https://example.com/mcp`.
3. Click Connect. Expect a `claude.ai/login/continue...` redirect, then a
   redirect to `https://example.com/mcp/oauth/authorize?...` (**not** a bare
   `/authorize`) with a consent/login prompt, then back to
   `claude.ai/api/mcp/auth_callback`.
4. If a previous attempt against the same URL failed, **remove that
   connector entry entirely and fully quit/reopen Claude Desktop** before
   adding it again — discovery results may be cached client- or
   backend-side, keyed by the MCP URL or client ID rather than by the
   individual connection instance, so simply retrying inside the same
   session/connector entry can reuse the old broken result.

## Troubleshooting quick-reference

| Symptom | Cause | Step |
|---|---|---|
| `.well-known/*` → 404, plain nginx page, never reaches PHP | Empty regex block in `security_locations.conf`, no override yet | 2, 3 |
| `.well-known/*` → 200 but body is the **homepage** | Fix applied to the SSL vhost (`server.*`) with a URI rewrite, which falls through to the generic Varnish proxy and loses the path | 1, 3 — use the non-rewriting proxy_pass version in the SSL vhost |
| curl against `.well-known/*` works, but Claude's own connect attempt still fails/404s on `/authorize` | Bot rate limiter (429) silently breaking Claude's discovery requests | 4 |
| Everything above is fixed, but a *new* connector attempt still shows old broken behavior | Client- or backend-side discovery cache keyed by URL/client_id | 5.4 |
| A change to `/data/web/nginx/...` doesn't seem to take effect | Checked `/etc/nginx/app/...` before confirming the reloader synced, or edited `/etc/nginx/app/...` directly and it got reverted | 1 |
