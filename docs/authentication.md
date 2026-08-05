# Authentication

The API is public and keyless. Every endpoint works with no credentials, so just
send the request. This is the same access the web app at
[demo.japanfold.com](https://demo.japanfold.com) uses, with the same limits.

```bash
# No auth needed:
curl -s https://api.japanfold.com/v1/models
```

## Optional API key: ownership

Send a key of the form `jf_live_…` as `Authorization: Bearer` or `X-API-Key`
to own your jobs under the key instead of your IP/session:

```bash
curl -s https://api.japanfold.com/v1/predictions \
  -H 'Authorization: Bearer jf_live_your_key_here' \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","sequence":"MKTAYIAK..."}'
```

A key that's present but invalid is rejected with `401`. Omit the header
entirely and requests are scoped to your IP/session instead. Keys are issued on
request by the JapanFold team.

## Ownership and quotas

- **Keyless requests** are grouped and rate-limited per client IP, and per
  session for the web app. Quotas are the same for everyone, key or no key.
- **Keyed requests** are owned by your key instead. Either way you can only list,
  poll, cancel or delete your own jobs; someone else's returns `404`.
- The concrete numbers are on [Models & limits](models-and-limits.md) and, live,
  at `GET /v1/models`.

Over a cap you get `400`; at capacity you get `429` with a `Retry-After` header
(see [Errors](errors.md)).
