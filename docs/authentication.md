# Authentication

The API is public and keyless. Submitting work, polling it, downloading results
and deleting a job all take no credentials, so just send the request. This is
the same access the web app at
[workbench.japanfold.aiand.com](https://workbench.japanfold.aiand.com) uses,
with the same limits. The one exception is listing your jobs.

```bash
# No auth needed:
curl -s https://api.japanfold.aiand.com/v1/models
```

## Optional API key: ownership

A key does two things: it moves job ownership off your IP address, and it
unlocks `GET /v1/jobs`, the one endpoint that will not answer keyless. It does
not raise any limit. Send it as `Authorization: Bearer` or `X-API-Key`:

```bash
curl -s https://api.japanfold.aiand.com/v1/predictions \
  -H 'Authorization: Bearer jf_live_your_key_here' \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}'
```

A key that's present but invalid is rejected with `401`. Omit the header
entirely and requests are scoped to your IP/session instead. Keys are issued
by the JapanFold team; there's no self-serve signup yet.

## Ownership and quotas

- **Keyless requests** are grouped and rate-limited per client IP, and per
  session for the web app. Quotas are the same for everyone, key or no key.
- **Keyed requests** are owned by your key instead. Either way you reach only
  your own jobs; someone else's id returns `404`. Polling, cancelling and
  deleting stay keyless, by the job id. Listing does not: without a key,
  everyone behind one office IP would share a job list, so
  [`GET /v1/jobs`](jobs.md#list-your-jobs) returns `401`.
- The concrete numbers are on [Models & limits](models-and-limits.md) and, live,
  at `GET /v1/models`.

Over a cap you get `400`; at capacity you get `429` with a `Retry-After` header
(see [Errors](errors.md)).
