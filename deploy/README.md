# VPS deployment configuration

Configuration captured from the Debian VPS at `ip-172-26-4-213` on 2026-11-27.
It is the source of truth for the nginx layer that fronts the IPFS site.

## How the site is served

`lochner.tech`, `www.lochner.tech`, and (after this change) `news.lochner.tech`
share one nginx server block. nginx terminates TLS and proxies every request to
the local Kubo gateway:

```
browser -> nginx :443 -> http://127.0.0.1:8080 (Kubo gateway)
```

The important line is:

```nginx
proxy_set_header Host lochner.tech;
```

Kubo's path gateway resolves `/` from the `Host` header through DNSLink
(`_dnslink.lochner.tech`). Because nginx always sends `Host: lochner.tech`, every
name in this block serves the same IPNS name:

```
k2k4r8jw4dtnalpkgklrqeflhsgderg6a8wn5lix7bww1yjemm0rx7ye
```

This is why `news.lochner.tech` does **not** need its own `_dnslink.news` TXT
record or a separate `Gateway.PublicGateways` entry. It reuses the existing
block and the existing certificate lineage.

## Files in this directory

| Path | Notes |
| --- | --- |
| `nginx/nginx.conf` | Verbatim copy of `/etc/nginx/nginx.conf`. |
| `nginx/sites-available/lochner.tech` | Target config. Differs from the VPS copy only by the `news.lochner.tech` additions documented below. |
| `nginx/sites-available/test-port.conf` | Verbatim copy. Enabled site for `ipfs.lochner.tech` on port `8765`, serving `/var/www/test-static`. This is a static test directory, not the IPFS gateway. |

`/etc/nginx/sites-enabled` is entirely symlinks into `sites-available`:

```
alfmir.net             -> ../sites-available/alfmir.net
chat-api.alfmir.net    -> ../sites-available/chat-api.alfmir.net
chat.alfmir.net        -> ../sites-available/chat.alfmir.net
default                -> ../sites-available/default
lemmy.lochner.tech.conf-> ../sites-available/lemmy.lochner.tech.conf
lochner.tech           -> ../sites-available/lochner.tech
test-port.conf         -> ../sites-available/test-port.conf
```

## Adding `news.lochner.tech`

Already reflected in `nginx/sites-available/lochner.tech`:

1. `server_name` includes `news.lochner.tech` in both the port 443 and port 80 blocks.
2. The port 80 block redirects `news.lochner.tech` to HTTPS before the final `return 404`.
3. The `lochner.tech` certificate lineage must include the new name.

### DNS

Already in place. `news.lochner.tech`, `lochner.tech`, and `ipfs.lochner.tech`
all resolve to the same VPS public address, `54.186.247.15`. No DNS change is
needed.

No `_dnslink.news.lochner.tech` record is needed either, because nginx forces
`Host: lochner.tech` and Kubo resolves `_dnslink.lochner.tech`.

The domain is also a Bluesky custom handle, and that verification is DNS-based:

```text
_atproto.news.lochner.tech. TXT "did=did:plc:ikc5s5ebbhbxqu5jmulny7wp"
```

Serving the IPFS site at the root of the domain does not interfere with it.

### Current symptom

`https://news.lochner.tech/` currently fails at TLS (`HTTP 526` from this
environment's inspecting proxy; a browser shows a certificate-name error)
because nginx has no matching `server_name` and the `lochner.tech` certificate
does not include the name. The IPFS content itself is reachable through the
existing block.

### Certificate and reload

```bash
sudo certbot --nginx --expand \
  -d lochner.tech -d www.lochner.tech -d news.lochner.tech
sudo nginx -t && sudo systemctl reload nginx
```

`certbot --nginx` edits the site file in place and may reformat it. The copy in
this directory is the intended target state; reconcile any formatting changes
back into the repo after running it.

### If certbot adds the name to `default`

`certbot --nginx` installs the certificate into the server block it finds for
the name. If `news.lochner.tech` is not already present in the live
`lochner.tech` block, certbot can add a new `server` block to
`sites-available/default` instead. Because `sites-enabled` loads
alphabetically, `default` can win the duplicate `server_name` and serve the
wrong content.

Apply `sites-available/lochner.tech` from this repository first, then remove
the certbot-generated `news.lochner.tech` server block from `default` (or
disable `default` if it is only the stock welcome page):

```bash
sudo cp /etc/nginx/sites-available/default "/root/default.bak.$(date +%s)"
sudo grep -n "news.lochner.tech" /etc/nginx/sites-available/default
# delete the server block(s) that contain that server_name, then:
sudo nginx -t && sudo systemctl reload nginx
```

### If nginx will not start after certbot

`bind() ... Address already in use` right after a certbot run means certbot
started its own nginx master and left it running while systemd still considers
the unit stopped. Stop the orphan and start through systemd:

```bash
sudo nginx -s stop
# if the process remains:
sudo pkill -9 nginx
sudo systemctl start nginx
sudo systemctl status nginx --no-pager
```

### Verify

```bash
dig +short A    news.lochner.tech
dig +short AAAA news.lochner.tech

curl -I http://news.lochner.tech/
curl -I https://news.lochner.tech/
curl -s https://news.lochner.tech/ipfs-version.json | head

openssl s_client -connect news.lochner.tech:443 \
  -servername news.lochner.tech </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName

curl -s -o /dev/null -w '%{http_code}\n' \
  http://127.0.0.1:8080/ipns/k2k4r8jw4dtnalpkgklrqeflhsgderg6a8wn5lix7bww1yjemm0rx7ye/
```

`subjectAltName` must list `news.lochner.tech`. If it does not, the old
certificate is still being served.

## Still to capture

The VPS also has these enabled site files. They were not copied here because
their contents are unknown or may contain secrets. Send sanitized copies:

- `sites-available/alfmir.net`
- `sites-available/chat.alfmir.net`
- `sites-available/chat-api.alfmir.net`
- `sites-available/lemmy.lochner.tech.conf`
- `sites-available/default`

Useful non-secret outputs to record alongside them:

```bash
sudo nginx -T
sudo certbot certificates
sudo ss -tlnp | grep -E ':80|:443|:8765|:8080|:4001|:4002|:3000'
screen -ls
ipfs config Addresses
ipfs config Gateway
ipfs config --json Gateway.PublicGateways
ipfs key list -l
```

## Do not commit

- `/etc/letsencrypt/live/**`, `/etc/letsencrypt/archive/**`, `privkey.pem`
- `~/.ipfs/config`, `~/.ipfs/keystore/**`, `~/.ipfs/swarm.key`
- `.env` files, API tokens, database passwords, or `Authorization` headers

The copied configs above contain paths and hostnames only. Redact anything
sensitive before adding the remaining site files.

## Operational note

The IPFS daemon is started manually inside `screen -S ipfs` and is not
supervised. After a reboot it must be restarted before the site resolves. A
systemd unit would remove that manual step but is outside the scope of this
config capture.
