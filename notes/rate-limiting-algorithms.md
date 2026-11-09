# Five rate-limiting algorithms, and when each one is the right call

Most rate limiters are built on one of five algorithms. They differ in three ways
that actually matter in production: how they treat bursts, how much state they keep
per key, and how accurate they are at window edges.

## TL;DR

| Algorithm | State per key | Bursts | Accuracy | Good default for |
|---|---|---|---|---|
| Fixed window counter | 1 counter | Up to 2x the limit at a window edge | Low | Quotas ("1000/day"), billing |
| Sliding window log | Every timestamp in the window | None beyond the limit | Exact | Low-volume, high-value actions (logins, OTPs) |
| Sliding window counter | 2 counters | Smoothed | Approximate, very close | General API limits at scale |
| Token bucket | Token count + timestamp | Allowed, up to bucket size | Exact for its model | Public APIs that should tolerate bursts |
| Leaky bucket / GCRA | 1 timestamp (GCRA) | Controlled by a tolerance value | Exact for its model | Smooth, evenly spaced traffic |

If you only remember one thing: **token bucket for APIs, fixed window for quotas,
sliding log only when you need exactness and volume is low.**

---

## 1. Fixed window counter

Split time into windows (e.g. each minute). Keep one counter per key per window.
`INCR`, set a TTL, reject when the counter exceeds the limit.

```
key = "rl:{user}:{floor(now / 60)}"
count = INCR key; if count == 1: EXPIRE key 60
allow = count <= limit
```

**The catch:** the edge. With a limit of 100/min, a client can send 100 requests at
12:00:59 and another 100 at 12:01:00. That's 200 requests in two seconds, and both
windows look fine.

Use it when the limit is a *quota* rather than protection: "10,000 calls per month on
the free plan". Nobody cares about a burst at the month boundary.

## 2. Sliding window log

Store a timestamp for every accepted request (a Redis sorted set works well). On each
request, drop timestamps older than the window, count what's left, and decide.

```
ZREMRANGEBYSCORE key 0 (now - window)
if ZCARD key < limit: ZADD key now <unique-id>; allow
else: reject
```

It's exact. It's also O(limit) memory per key, and a busy key with a 10k/hour limit
holds 10k entries. Fine for login attempts, expensive for every API call.

Two details people miss:
- Use a unique member (`now + random suffix`); two requests in the same millisecond
  with the same member collapse into one entry.
- Decide whether *rejected* requests are logged. If they are, a client that keeps
  hammering stays locked out forever. Usually you only log accepted ones.

## 3. Sliding window counter

A cheap approximation of the sliding log. Keep the fixed-window counters for the
current and previous window, and weight the previous one by how much of it still
overlaps the sliding window:

```
elapsed  = (now % window) / window          # 0.0 .. 1.0 into the current window
estimate = curr_count + prev_count * (1 - elapsed)
allow    = estimate < limit
```

It assumes requests in the previous window were spread evenly, which is rarely true,
but the error is small in practice. Cloudflare measured this approach on 400 million
requests from 270,000 sources and found 0.003% of requests were wrongly allowed or
limited ([source](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)).

Two counters per key, no edge-burst problem, close to exact. A strong default.

## 4. Token bucket

A bucket holds up to `capacity` tokens and refills at `rate` tokens/second. Each
request takes a token (or `cost` tokens); no token, no request.

The two parameters mean different things, and that's the point:
- `rate` is the long-run average you allow.
- `capacity` is how big a burst you tolerate.

A client that's been idle can spend its whole bucket at once, then settles down to
`rate`. That matches how real clients behave (a page load fires 20 calls, then
nothing for a while), which is why it's common for public APIs. AWS API Gateway
throttles with a token bucket
([docs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)),
and Stripe described their Redis-based token bucket limiter in
[Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters).

You don't need a timer to refill. Store `tokens` and `last_refill`, and compute the
refill lazily on each request. In Redis, do it in one Lua script so the
read-modify-write is atomic:

```lua
-- KEYS[1] = bucket key
-- ARGV: capacity, refill rate (tokens/sec), now (ms), cost
local capacity = tonumber(ARGV[1])
local rate     = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local cost     = tonumber(ARGV[4])

local state  = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(state[1]) or capacity
local ts     = tonumber(state[2]) or now

-- lazy refill, capped at capacity
tokens = math.min(capacity, tokens + math.max(0, now - ts) / 1000 * rate)

local allowed = tokens >= cost
if allowed then tokens = tokens - cost end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
-- a full bucket is the same as no key, so let idle keys expire
redis.call('PEXPIRE', KEYS[1], math.ceil(capacity / rate * 1000))

-- Lua false becomes nil in a Redis reply, and floats get truncated: be explicit
return { allowed and 1 or 0, tostring(tokens) }
```

`now` comes from the caller here. If your app servers' clocks drift, read
`redis.call('TIME')` inside the script instead, so every instance shares one clock.

## 5. Leaky bucket and GCRA

The leaky bucket has two readings that people mix up:

- **As a queue:** requests go into a FIFO and drain at a fixed rate. Output is
  perfectly smooth, but requests wait, and a full queue means rejection. This is
  traffic *shaping*. NGINX's `limit_req` is built on it, and its `burst` / `nodelay`
  options control whether excess requests are queued or passed through
  ([docs](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)).
- **As a meter:** no queue, just "would this request overflow the bucket?". This is
  equivalent to a token bucket viewed from the other side.

**GCRA** (Generic Cell Rate Algorithm, from ATM networks) implements the meter with a
single stored timestamp, the *theoretical arrival time* (TAT):

```
T   = period / limit          # emission interval, e.g. 60s / 100 = 0.6s
tau = T * (burst - 1)         # how far ahead of schedule a client may run

tat = stored_tat or now
if now < tat - tau: reject, retry after (tat - tau - now)
else: store max(tat, now) + T; allow
```

One number per key, no refill math, and `retry_after` falls out for free. Brandur
Leach's [write-up](https://brandur.org/rate-limiting) is the clearest explanation
of it I know of.

---

## Things that bite in production

**Races.** `GET`, compare, then `SET` from application code will let concurrent
requests through. Do the whole decision atomically: a Lua script, `INCR` (which
returns the new value), or a single conditional write.

**Distributed limits.** One Redis is one round trip per request and one point of
failure. Common compromises: shard keys across nodes; or keep local per-instance
buckets with `limit / N` each and accept some imprecision; or count locally and sync
to a central store every few hundred ms. All trade exactness for latency.

**Fail open or fail closed?** If the limiter's store is down, do you block all
traffic or allow all of it? For abuse protection on a login endpoint, fail closed.
For a general API, failing open is usually the less bad outage. Decide on purpose
and write it down.

**What you key on.** Per-IP limits punish everyone behind a corporate NAT or
mobile carrier and are trivial to dodge with a botnet. Prefer API key or user ID,
and keep a looser IP limit as a backstop for unauthenticated routes.

**Tell the client.** Reject with `429 Too Many Requests`
([RFC 6585](https://www.rfc-editor.org/rfc/rfc6585)) and send `Retry-After`. Many
APIs also expose `X-RateLimit-Limit` / `-Remaining` / `-Reset`; the IETF is
standardising this as `RateLimit` and `RateLimit-Policy` headers
([draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)).
Well-behaved clients will back off on their own if you tell them how.

## Picking one

- Protecting a public API, clients are bursty → **token bucket**.
- You want evenly spaced requests to a fragile downstream → **leaky bucket (queue)**.
- Same as token bucket but you want minimal state → **GCRA**.
- Lots of keys, want close-to-exact sliding limits cheaply → **sliding window counter**.
- Small number of sensitive actions, exactness matters → **sliding window log**.
- Monthly or daily plan quotas → **fixed window**.

## References

- Cloudflare, [Counting things, a lot of different things](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) (sliding window counter accuracy)
- Stripe, [Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- Brandur Leach, [Rate limiting, cells, and GCRA](https://brandur.org/rate-limiting)
- NGINX, [ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- AWS, [Throttle requests to your REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- IETF, [RateLimit header fields for HTTP (draft)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- RFC 6585, [Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585) (429)
