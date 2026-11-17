# Antigravity network routing

`User location is not supported for the API use` is an upstream generation
failure. A successful login, a paid subscription, available model quota, or a
successful Claude request does not establish that Gemini generation works on the
same route.

## Diagnose the route

1. Check the affected credential's model list using the management API. Use the
   exact model ID; for example, `gemini-4-carbon-high` includes its suffix.
2. Inspect the application container's configured DNS servers with
   `docker inspect`. Docker's `127.0.0.11` resolver can forward queries to a
   separately configured resolver; its presence alone does not indicate a leak.
3. Resolve the actual upstream host from the application's network namespace.
   Antigravity generation uses `daily-cloudcode-pa.googleapis.com`, while account
   discovery can use `cloudcode-pa.googleapis.com`.
4. Verify the peer address during an actual request. A changed DNS answer proves
   that a new relay was selected, but not that Google accepts that relay's exit.
5. Compare the same credential, model, and minimal request on another existing
   route. Keep the credential and request contents out of diagnostic output.

Keep custom DNS settings on the intended application services when only those
services should use DNS routing. Changing the host resolver is unnecessary for
this configuration.

## Set a proxy for one credential

CPA supports the `proxy_url` metadata field on an OAuth credential. Use the
management API's credential-fields endpoint to update the running credential and
persist the setting. The deployed v8 service exposes
`PATCH /v8/management/credentials/fields`; check the running version's routes
before applying it to another deployment.

Request body, with placeholders only:

```json
{
  "name": "<credential-file-name>",
  "proxy_url": "socks5h://<encoded-user>:<encoded-password>@<proxy-host>:<proxy-port>"
}
```

Use an existing trusted proxy that the server can reach. Percent-encode the
username and password when constructing the URL. Keep the real URL in the private
credential store; do not commit authentication files or management keys.

Setting this field on one credential limits the change to that credential. It
does not configure another service or change the global routing strategy. Remove
the override by updating `proxy_url` to an empty string through the same endpoint.

## Route all CPA providers through one proxy

For a deployment that should use the same proxy for all provider requests, set
the v8 global request proxy in the private `config.yaml`:

```yaml
requests:
  proxy-url: "socks5h://<encoded-user>:<encoded-password>@<proxy-host>:<proxy-port>"
```

Audit existing credential-level `proxy_url` values and configured provider
overrides as well. They can override the global setting. In the affected
deployment, the same proxy was explicitly applied to all 45 existing credential
files, including credentials that remain disabled, so credential refresh and
model discovery also have an explicit route. Empty provider-level proxy values
retain the global fallback. No account was enabled by this change.

Persist the configuration and restart CPA to replace existing upstream
connections. Confirm that live provider connections reach the SOCKS5 endpoint.
The HTTP listener can become ready before account-specific models finish
registering. Wait until the requested model appears in `/v1/models` before testing
generation after a restart. If it remains absent, compare the stored
`antigravity_capability_hints.model_ids` against an authenticated upstream
`fetchAvailableModels` response. Use only actual upstream model IDs when
refreshing capabilities; subscription status alone does not establish access.

## Interpret 429 responses separately from remaining quota

A credential's positive `quotaInfo.remainingFraction` does not guarantee that
every generation request will be accepted immediately. A generic upstream
`429 RESOURCE_EXHAUSTED` does not identify the exact limiting resource. CPA's
`reason=quota` cooldown label describes its routing state, not a fresh measurement
that the account has zero remaining quota.

On 2026-11-17, the two affected Argon credentials reported approximately 99.99%
and 99.22% remaining while their generation requests had produced 429 responses.
Minimal requests subsequently succeeded directly upstream, including a test with
the original 64,000 output-token limit. This did not establish the output limit
as the cause of the earlier failures.

The deployment's routing strategy was changed from `fill-first` to `round-robin`
to distribute requests across eligible credentials instead of concentrating new
requests on the first eligible credential. Cooling and credential failover remain
enabled. After direct generation was confirmed to work, the affected credential's
cooldown was reset once using the management API. A native Gemini request through
CPA then returned HTTP 200 and `OK` in approximately five seconds.

Do not repeatedly clear cooldowns or disable cooling to force traffic through
an upstream that continues returning 429. Successful minimal requests do not
prove that a previous long conversation will succeed under concurrent load.
Google documents capacity-dependent limits in its
[Antigravity plans documentation](https://antigravity.google/docs/plans).

## Verify generation

Verify the persisted setting, then test through CPA itself, rather than relying
only on a direct upstream call. Use a short instruction such as
`Reply with NETWORK_OK only.` and allow sufficient output tokens for a reasoning
model. Confirm HTTP 200, the expected text, and `finish_reason: stop`. For the
public streaming endpoint, also confirm the final `data: [DONE]` marker.

Check the reverse proxy's API prefix before testing a public endpoint. In the
affected deployment, CPA is mounted at `/cpa/`, so its chat endpoint is
`/cpa/v1/chat/completions`. The root `/v1/` belongs to another service. A
Cloudflare `1010` response is an edge client-signature rejection, separate from
Google's JSON location error; use the application's normal client headers when
checking that public route.

On 2026-11-17, the affected deployment returned the location error through both
its selected JP and US DNS relays. The same credential generated successfully
over another network. An existing server-accessible SOCKS5 proxy also returned
HTTP 200 for both `gemini-4-carbon-high` and `gemini-4-hydrogen-xhigh`. Applying
that proxy to the affected credential through the management API restored both
models through CPA, with `NETWORK_OK` and `finish_reason: stop`.
The public `/cpa/v1/chat/completions` streaming route was also verified with
HTTP 200, `NETWORK_OK`, and the final `[DONE]` marker.

This is a deployment configuration fix; no executor or retry-policy changes are
required. DNS relay availability and proxy availability can change independently,
so repeat a generation check when changing the route.
