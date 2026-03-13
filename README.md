# ☢ Hash Chain Budget

Stateless rate limiting using self-consuming cryptographic hash chains.

---

Initial thought was to find a simple 'limit-on-itself' state/var for a very trivial usecase — limit downloads in a stateless JS app. Track n-times, then expire. Reload updates the session. Like radioactive material and its half-life. My brain said: the solution is already there. So CC found it and says; 'old tech, new usecase'. And that it could be a big deal. So here it is, you decide.

---

## Origin

This pattern emerged from a practical problem: enforcing download limits in a Shopify app without per-download database calls. The underlying primitive is Lamport's hash chain from 1981, originally used for one-time passwords. The new framing: not authentication, but a **self-consuming stateless budget** that resets automatically on a time boundary.

## Demo

See `index.html` — vanilla JS, no dependencies, runs in any browser.
