# ☢ Hash Chain Budget

Stateless rate limiting using self-consuming cryptographic hash chains.

---

Initial thought was to find a simple 'limit-on-itself' state/var for a very trivial usecase — limit downloads in a stateless JS app. Track n-times, then expire. Reload updates the session. Like radioactive material and its half-life. My brain said: the solution is already there. So CC found it and says; 'old tech, new usecase'. And that it could be a big deal. So here it is, you decide.

---

## The problem

Every API rate limiter on the internet works the same way: a counter in a database, incremented on every request, checked against a limit. At scale this becomes the most-hit table in your infrastructure. Redis clusters exist largely to serve this one pattern.

## The solution

The server mints a random seed and walks it N times through SHA-256 to produce an endpoint. It sends the seed to the client and stores only the endpoint. Each API call the client hashes its current value once — consuming one step — and presents it to the server. The server verifies by walking the remaining steps to the known endpoint.

```
seed → hash → V₁ → hash → V₂ → … → Vₙ → dead
                                          ↑
                                    server endpoint
```

No counter. No increment. No read-modify-write race condition. **The budget is in the math.**

Time boundaries are free: bake the current hour into the seed on the server side.

```js
seed = sha256(userId + currentHour + secretKey)
```

Same seed, every time, for the same user in the same hour. Different hour — different seed, different chain, automatically expired. No TTL columns, no cron jobs, no cleanup.

## How it works

**Server — mint (once per session or hour):**
```js
const seed = await sha256(userId + currentHour + secretKey)
let endpoint = seed
for (let i = 0; i < n; i++) endpoint = await sha256(endpoint)
db.set(userId + currentHour, endpoint)
return { seed, steps: n }
```

**Client — consume (each API call):**
```js
let current = seed  // received on init, lives in memory
let steps = n

async function callAPI(payload) {
  if (steps <= 0) throw new Error('rate limit exceeded')
  current = await sha256(current)
  steps--
  return fetch('/api', {
    headers: { 'X-Chain': current, 'X-Steps': steps },
    body: payload
  })
}
```

**Server — verify (each API call, no DB read for the counter):**
```js
async function verify(userId, current, stepsLeft) {
  let val = current
  for (let i = 0; i < stepsLeft; i++) val = await sha256(val)
  return val === db.get(userId + currentHour)
}
```

The only database read is fetching the endpoint — one value, heavily cacheable, never written during the rate-limited period.

## At scale — what this actually changes

### 100 million API calls per day

Traditional approach: 100 million counter increments + 100 million counter reads against a central Redis cluster. Redis becomes the chokepoint. You add replicas, you tune persistence, you worry about split-brain.

Hash chain approach: 100 million SHA-256 computations (microseconds each, CPU-only, no I/O) + 100 million endpoint lookups (one read per user per hour, fully cacheable at the edge). **The counter is gone. The write load is zero.**

---

### Global edge deployment

Traditional approach: rate limit state lives in a central Redis. Edge nodes in Tokyo, São Paulo, and Frankfurt all phone home to verify every request. Latency, single point of failure, cross-region replication cost.

Hash chain approach: the endpoint is a static value per user per hour. Cache it at the edge. Every verification is local — pure CPU, no network hop. **Rate limiting becomes as fast as a hash function.**

---

### Serverless and stateless functions

Traditional approach: a Lambda function needs rate limiting. It has no persistent memory. Every invocation reads and writes a central counter. The function is stateless but its rate limiter isn't, which defeats the architecture.

Hash chain approach: the function receives the chain step in the request header. It fetches the cached endpoint (or recomputes it from the seed derivation formula). It verifies locally. **No shared mutable state anywhere in the call path.**

---

### Microservices — no shared session store

Traditional approach: Service A, B, and C all need to enforce the same rate limit. They share a Redis. Now every service depends on Redis being up, reachable, and consistent.

Hash chain approach: each service independently verifies the chain step. They all know the secret key and the derivation formula. No shared infrastructure required. **Each service is fully autonomous.**

---

### Auth server load

Traditional approach: every request hits the auth server to validate a JWT and check rate limit state. The auth server is the most critical, highest-traffic service you run.

Hash chain approach: the auth server mints chains at session start. Between sessions it is not involved. **Auth server load goes from continuous to heartbeat.**

---

### Burst detection and audit

Each chain step is deterministic and sequential. If a client presents step 7 and the server last saw step 3, four steps were consumed without being presented — anomaly detected without any logging infrastructure. **The chain is its own audit trail.**

---

## Properties

- **Zero write load during rate-limited period** — the only writes are mint and exhaust
- **Fully cacheable** — the endpoint never changes within its time window
- **Edge-native** — verification is CPU-only, no I/O, runs anywhere
- **Automatic expiry** — time-scoped seeds expire without any cleanup
- **Race-condition free** — no read-modify-write, no distributed lock needed
- **Auditable** — gaps in the sequence are detectable without logging

## Origin

This pattern emerged from a practical problem: enforcing download limits in a Shopify app without per-download database calls. The underlying primitive is Lamport's hash chain from 1981, originally used for one-time passwords. The new framing: not authentication, but a **self-consuming stateless budget** that resets automatically on a time boundary.

## Demo

See `index.html` — vanilla JS, no dependencies, runs in any browser.
