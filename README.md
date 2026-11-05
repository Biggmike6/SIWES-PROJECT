# LinkGuard — SIWES website scam-warning detector

React + Vite frontend, Node/Express API, PostgreSQL persistence. Use Node.js 24 LTS (or 22.19+) and a local PostgreSQL installation. The backend uses the system certificate store as well as Node's normal trust store; certificate validation remains enabled.

## First run
1. Create a PostgreSQL database named `linkguard` (using pgAdmin or `CREATE DATABASE linkguard;`).
2. Keep the existing `Backend/.env`, and set `DATABASE_URL` to the connection string for your database. See `.env.example`. URL-encode special characters in the password. Never commit this file.
3. Optionally add `GOOGLE_SAFE_BROWSING_KEY` in that file. Without a key the reputation check is explicitly unavailable.
4. In a terminal inside Backend: `npm install`, then `npm run db:setup`, then `npm run dev`.
5. In a separate terminal inside Frontend: `npm install`, then `npm run dev`.
6. Open http://localhost:5173. Vite forwards `/api` requests to Express on port 5000.

## Commands
Backend: `npm test` runs deterministic scoring/security checks. `npm start` starts the API. Frontend: `npm run build` creates the production bundle. Production hosting needs its own same-origin `/api` reverse proxy or frontend API configuration.

## API
- POST `/api/scan` with JSON `{ "url": "https://example.com" }` returns the saved report (201).
- GET `/api/scans?page=1` returns 20 saved summaries with pagination.
- GET `/api/scans/:id` returns one full report.
- GET `/api/health` checks database connectivity.

## Content inspection and scoring version 2.0
The server downloads the submitted public page, follows up to four redirects per page, and reads up to two relevant same-origin links (plans, investments, FAQs, or similar). It extracts text without executing scripts and records short previews and quoted evidence. Scans are not a full-site audit: login-only content, images, JavaScript-rendered text, and unlinked offers are not inspected. Unreadable or blocked content produces an Inconclusive assessment when no stronger warning is present.

Content warning weights: very high advertised returns 60; payment to unlock a withdrawal 60; guaranteed/risk-free investment claims 35; recruitment rewards in an investment context 20; recurring daily-profit promise without a measurable high rate 15; pressure to invest quickly 10. Each rule contributes once across inspected pages. High-return thresholds are provisional: 1% daily, 5% weekly, or 20% monthly. These are project heuristics, not legal definitions or calibrated fraud probabilities.

Regression tests include the described offer of investing ₦10,000 and receiving ₦500 daily (5% daily), plus investing ₦100,000 and withdrawing ₦300,000 monthly (200% monthly profit including returned principal). They are synthetic examples, not findings about any named live company. Common anti-scam warnings and negated guarantees are excluded by simple contextual rules. Language variants and layout differences can still be missed; evaluate a labelled dataset before claiming accuracy.

The report contains `assessment`, `recommendation`, `content`, evidence, and `scoring_version: "2.0"`. API `verdict` displays the assessment: Dangerous, Suspicious, Inconclusive, No major warning found, or URL-only report for older history. The existing database verdict column retains its numerical bucket for compatibility; the API exposes that separately as `risk_bucket`. No major warning found requires readable content and completed checks; it never verifies the operator or clears the investment. Older reports are not retroactively rescanned.

### Supporting URL checks
Known Google threat: 80 points. Brand impersonation/curated pattern: 35. Domain under 30 days: 20, under 90 days: 10. Invalid certificate or HTTP: 15. IP URL: 15. Embedded credentials: 15. More than two subdomain levels: 10. Known shortener: 10. Sum capped at 100; 0–24 Low risk, 25–59 Suspicious, 60–100 Dangerous. These are provisional rule weights, not statistically calibrated probabilities.

Unavailable checks add zero points but reduce displayed coverage; a partial low score is not a clean bill of health. Google no-match and valid TLS do not prove legitimacy. Domain age uses RDAP registration events rather than raw WHOIS. Brand matching uses the public suffix list via tldts. Official domains are a curated baseline that must be verified and maintained. Only `hostname_contains` custom patterns are supported by the two-table MVP; edit `blacklist_patterns` through SQL.

## Security and limits
Only HTTP/HTTPS on standard ports is accepted. Private/reserved IPs and domains resolving to them are blocked on every redirect. Download and TLS connections are pinned to validated public IP addresses to reduce DNS-rebinding risk. HTTP redirects can expand shorteners; JavaScript and meta-refresh redirects are not followed. Submitted and final URLs receive reputation checks; destination URL/domain checks accompany the content analysis. Requests have deadlines, header limits, and a 768 KiB response limit. Compressed responses are rejected rather than decompressed without bounds. Supporting pages are limited to the final site's origin. Credentials in URL userinfo are never used for page downloads. No forms are submitted and no account is created.

Google receives the submitted and final URLs when its API key is configured, and RDAP receives the destination domain. The operator's investment claims and regulatory registration are not verified; users must independently consult Nigeria's SEC. A trusted SSL certificate only establishes connection encryption and certificate identity, not business legitimacy.

History has no authentication and is shared by anyone accessing this installation. Keep the MVP local; do not submit links containing passwords or private tokens. Before public deployment add authentication/history isolation, abuse controls at the hosting layer, and an evaluated phishing/benign dataset. The scan route is limited to 10 requests per minute per IP in one server process. URL credentials are retained to support the detection demonstration; do not use real credentials.

## Defence preparation
Explain each score contribution, show unavailable-check behavior, and compare official domains with deceptive subdomain examples in the automated tests. Evaluate false positives/negatives on a labelled, legally obtained dataset before claiming detection accuracy.

References: Nigeria SEC https://sec.gov.ng/for-investors/keep-track-of-circulars/public-notice-unregistered-online-investment-schemes/ ; Investor.gov https://www.investor.gov/protect-your-investments/fraud/types-fraud/ponzi-scheme ; IC3 https://www.ic3.gov/CrimeInfo/Investment ; Google Safe Browsing v4 https://developers.google.com/safe-browsing/v4/ ; Node TLS https://nodejs.org/api/tls.html ; Node DNS https://nodejs.org/api/dns.html . Google Safe Browsing is for non-commercial use.
