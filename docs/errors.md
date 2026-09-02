# Errors

Endpoint errors come back as RFC 9457 problem+json, with the keys in
alphabetical order:

```json
{
  "detail": "unknown model 'nope' — choose one of ['boltz2', 'esmfold2', 'esmfold2-fast', 'openbind', 'opendde', 'opendde-abag', 'openfold3', 'protenix-v2', 'rf3'].",
  "instance": "/v1/predictions",
  "status": 400,
  "title": "Invalid request",
  "type": "https://japanfold.aiand.com/errors/invalid-input"
}
```

- `detail`: specific explanation of *this* failure. Read this first.
- `instance`: the request path that failed.
- `status`: the HTTP status, mirrored in the body.
- `title`: short, human-readable summary.
- `type`: a URI categorizing the error (may be `about:blank`).

## Status codes

| Code | Meaning | What to do |
|---|---|---|
| `400` | Invalid request: bad params, malformed input, or over a size cap (residues, chains, designs, …). | Fix the request; read `detail`. See [Models & limits](models-and-limits.md). |
| `401` | Missing/invalid credentials for an authenticated action. | Check your `Authorization: Bearer` key. |
| `403` | Forbidden. | See the Cloudflare note below if `detail` mentions error `1010`. |
| `404` | No such job, or a job you don't own. | Check the id; jobs are scoped to their owner. |
| `413` | Request body over 8 MiB. Served as Flask's HTML page, the one response that is not problem+json. | Shrink the input; you're almost certainly over `max_content_chars` anyway (see [Models & limits](models-and-limits.md)). |
| `429` | At capacity, over a rate limit, or already at 3 active jobs. | **Honor the `Retry-After` header**, then retry. See the limits page. |
| `503` | The accelerators are offline for maintenance, so no job can start. | **Honor the `Retry-After` header** (5 minutes) and retry. Nothing is wrong with your request. |

## Handling 429 (at capacity)

When the service is busy, or you exceed a submit or active-job quota, you get
`429` with a `Retry-After` header in seconds. Wait that long and retry, backing
off if it repeats.

```bash
resp=$(curl -s -D /tmp/h -o /tmp/b -w '%{http_code}' -X POST \
  https://api.japanfold.aiand.com/v1/predictions -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}')
if [ "$resp" = "429" ]; then
  sleep "$(grep -i '^retry-after:' /tmp/h | tr -dc 0-9)"
  # retry...
fi
```

## Handling 503 (maintenance)

`429` and `503` mean different things. `429` is a queue: the hardware is running
and your turn is coming. `503` means there is no hardware to run on at that
moment, so submitting again immediately cannot succeed:

```json
{
  "type":   "https://japanfold.aiand.com/errors/unavailable",
  "title":  "Service Unavailable",
  "status": 503,
  "detail": "JapanFold's accelerators are offline for maintenance right now, so no jobs can start. Please try again in a few minutes.",
  "instance": "/v1/predictions"
}
```

Only job creation (`/v1/predictions`, `/v1/designs`, `/v1/embeddings`) returns
it. Reading jobs and downloading results keep working, so anything you submitted
earlier stays available.

Three responses come from the framework rather than the API, so they are not
problem+json: a `413`, a `405` from the wrong HTTP method (both HTML), and a
`404` on a path that does not exist (`{"error":"not found"}`). Every error from
an endpoint that exists is problem+json.

## Large bodies

The body cap is 8 MiB, and it sits above the input caps, so an oversized input
usually fails validation with a `400` first: an 8.2 MB request comes back as a
problem+json `400` naming `max_content_chars`. Only past 8 MiB do you get the
`413`, and a body far past it (9 MB) is dropped by the edge with a `502` before
the API sees it at all.

## Cloudflare 403 / error 1010

A `403` whose body references Cloudflare **error 1010** is edge-level bot
filtering of your HTTP client, not an API error. Retry the identical request with
a browser-like `User-Agent`:

```bash
curl -s -A 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36' \
  https://api.japanfold.aiand.com/v1/models
```
