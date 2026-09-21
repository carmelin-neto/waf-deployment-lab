# WAF Deployment Lab — Detection, Enforcement, and a Real Cloud Constraint

Type: Web Application Firewall deployment and testing, in two parts — (1) a self-hosted ModSecurity/OWASP CRS deployment in front of a deliberately vulnerable app, tested through both detection-only and blocking modes, and (2) an attempted cloud extension using Cloudflare, which surfaced a genuine infrastructure constraint rather than completing as originally planned.

Why this project exists: Every other project in this portfolio deals with network-layer and OT-specific security. This one closes a different gap — web application security, the layer most cloud-hosted business systems actually run on, and a skill explicitly requested across several roles I applied to.
The same discipline of naming real infrastructure constraints honestly — rather than hiding them — is applied again in [home-soc-ai-triage](https://github.com/carmelin-neto/home-soc-ai-triage), which documents a Wazuh ingestion gotcha and a billing-system pivot to a local AI model.

---

 Part 1: Self-Hosted WAF (ModSecurity + OWASP CRS)

Target: [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/), a deliberately vulnerable web application, run locally in Docker.

WAF: nginx + ModSecurity 3.0.16 + OWASP Core Rule Set 3.3.10, also in Docker, configured at Paranoia Level 1 with an inbound anomaly threshold of 5.

Baseline: No WAF (Port 3000, Direct)

A UNION-based SQL injection payload sent directly to the unprotected application:

```
curl -i "http://localhost:3000/rest/products/search?q=test'))%20UNION%20SELECT%20*%20FROM%20users--"
```

**Result:** `500 Internal Server Error`, with the raw SQLite error returned directly in the response body:

```
SQLITE_ERROR: SELECTs to the left and right of UNION do not have the same number of result columns
```

This confirms two separate problems, not one: the application is vulnerable to SQL injection at this endpoint (the payload reached the database layer unsanitized), and it practices verbose error disclosure — the raw database error, including the exact SQL structure and backend technology (SQLite, Express ^4.22.1), is leaked straight to the client. In a real attack, this error message is itself useful reconnaissance: it tells an attacker exactly how to adjust a `UNION SELECT` payload to succeed, by revealing the column-count mismatch that caused the failure.

 Detection Without Enforcement (Port 8080, `MODSEC_RULE_ENGINE=DetectionOnly`)

The identical payload, now passing through the WAF in detection-only mode:

```
curl -i "http://localhost:8080/rest/products/search?q=test'))%20UNION%20SELECT%20*%20FROM%20users--"
```

Result: the attack still succeeds — same `500` error, same leaked SQLite message. But the WAF's own logs show it was seen and correctly evaluated:

- Two independent OWASP CRS rules matched the payload: rule 942190 ("Detects MSSQL code execution and information gathering attempts") and rule 942270 ("Looking for basic sql injection")
- The resulting **inbound anomaly score was 10** — double the configured blocking threshold of 5

This is the expected, correct behavior of detection-only mode: full visibility into what *would* be blocked, with zero actual enforcement. It's the equivalent of a smoke detector that sounds an alarm log but doesn't trigger the sprinklers — useful for tuning a rule set against real traffic before ever risking a false-positive outage in production.

### Enforcement (Port 8080, `MODSEC_RULE_ENGINE=On`)

Same payload, same endpoint, WAF now switched to blocking mode:

```
curl -i "http://localhost:8080/rest/products/search?q=test'))%20UNION%20SELECT%20*%20FROM%20users--"
```

Result: `403 Forbidden`. The request never reaches the vulnerable application at all.

False-Positive Check

Before calling the blocking configuration good, I tested it against **legitimate** traffic — a normal product search:

```
curl -i "http://localhost:8080/rest/products/search?q=apple"
```

Result: clean `200 OK`, full product results returned normally. The WAF is not over-blocking ordinary use.

Part 1 Summary

| Stage | Target | Result |
|---|---|---|
| No WAF | Port 3000 | SQLi succeeds; raw SQLite error and backend stack leaked to client |
| WAF, Detection-Only | Port 8080 | SQLi still succeeds (same leak); WAF correctly logs and scores it (anomaly score 10 vs. threshold 5) |
| WAF, Blocking | Port 8080 | SQLi blocked outright — `403 Forbidden`, request never reaches the app |
| WAF, Blocking, legitimate traffic | Port 8080 | Normal search returns `200 OK` — no false positive |

This is a complete, provable before/after/verified chain: the vulnerability is real and demonstrated, the WAF's detection accuracy is proven independently of its blocking behavior, the blocking behavior is confirmed to work, and the blocking configuration is confirmed not to break legitimate use.

---

Part 2: Attempted Cloud Extension (Cloudflare) — A Real Infrastructure Constraint

Goal: repeat the same attack/detection/block sequence against a cloud-based WAF (Cloudflare's free tier), to contrast a self-hosted deployment against a managed cloud service.

What happened: I successfully established a Cloudflare Tunnel exposing the local Juice Shop instance to the internet, and verified the tunnel mechanism worked end-to-end (confirmed via a live public URL returning the application's normal response). However, applying Cloudflare's WAF rules requires the traffic to route through a zone — a domain actually registered in a Cloudflare account. A quick, ephemeral tunnel URL (`trycloudflare.com`) is not tied to any zone in the account and cannot have WAF rules attached to it, regardless of tunnel configuration.

Setting up a proper named tunnel confirmed this directly: Cloudflare's own tunnel-authorization flow prompted for a zone selection, and the list was genuinely empty — no domain had been added to the account, and none is provided for free. Cloudflare's free tier removes the *cost* of WAF rules and managed rulesets, but it does not remove the requirement to own a registered domain in the first place.

Why I'm documenting this as a finding rather than omitting it: this is a real, useful piece of knowledge for anyone evaluating cloud WAF options on a budget — "free tier" does not mean "zero cost to get started," and the actual floor cost of a genuine cloud WAF deployment is the price of a domain (roughly $8-12/year), not the WAF service itself. That distinction matters for real budget planning, and it's a more honest, more useful finding than a clean deployment with no obstacles would have been.

What a completed Phase 2 would require: registering an actual domain, pointing its nameservers to Cloudflare, routing the named tunnel through it, then repeating the identical attack/false-positive test sequence from Part 1 against the Cloudflare-protected domain. The self-hosted results in Part 1 already establish the baseline methodology this would extend.

---

What This Project Demonstrates

The core skill here isn't "deployed a WAF" — it's verifying a security control's behavior in stages rather than trusting a configuration once it's turned on: proving the vulnerability exists, proving the control sees it accurately before trusting it to act, proving the control blocks it, and proving the control doesn't break legitimate traffic in the process. Part 2's incomplete status is itself handled the same way the rest of this portfolio treats limitations — named directly, with the real reason and the real cost to close it, rather than glossed over.

 Portfolio Connections

| This project | Relationship |
|---|---|
| [ot-active-scanning-risk](https://github.com/carmelin-neto/ot-active-scanning-risk) | Same underlying discipline — verify before you trust a tool's behavior, whether it's a scanner or a WAF |
| [ot-vulnerability-assessment](https://github.com/carmelin-neto/ot-vulnerability-assessment) | Same finding-then-fix structure, applied to web application security instead of network/OT security |
