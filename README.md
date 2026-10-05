# web scraping with proxies: choosing the right proxy type, dodging blocks, and starting for $5 in traffic

Most people who search for this end up in the same spot: the scraper works fine on page one, then returns 403s, CAPTCHAs, or an empty shell of HTML by page fifty. The code didn't change. What changed is that the target started recognizing you.

Proxies are the standard fix, but "get proxies" is not a complete answer. The type you buy, how you rotate it, and how much bandwidth you burn decide whether your success rate sits at 60% or 97% — and whether your bill is $5 a month or $500. This guide walks through the whole chain: what actually gets a scraper blocked, which proxy tier matches which target, how rotation should be configured, and what a realistic setup costs on DataImpulse, which sells residential traffic from $1/GB with no subscription.

## What actually gets you blocked

Anti-bot systems score a request on several signals at once. Proxies fix some of them. Knowing which ones matters, because paying for better IPs to solve a header problem is just an expensive way to stay blocked.

- **IP reputation and rate limits.** One IP firing 20 requests a second at one domain is the easiest pattern to catch. This is the layer proxies genuinely solve: distribute the load across many exits, and no single address accumulates enough activity to look suspicious.
- **IP type.** Datacenter IPs are cheap and fast, but they're published ranges. Sites that care about bot traffic score them accordingly.
- **Fingerprinting.** TLS handshake details, header order, and JavaScript execution all get inspected. A residential IP paired with a `python-requests/2.x` user-agent is still a tell.
- **Behavior.** Browsers keep cookies, pause between clicks, and don't jump from Germany to Brazil mid-session. Scrapers that do all three get flagged on behavior alone.

The useful mental model: the proxy makes your address believable, hygiene makes your behavior believable. Neither substitutes for the other. Any provider selling you only the first half is selling you half a solution — and that's true of the cheap tier and the enterprise tier alike.

## Match the proxy type to the target, not to your budget alone

There's a table like this in most guides, and it's the part worth printing out. The rule underneath it: use the cheapest tier the target will tolerate, and escalate only when block rates prove you have to.

| Target | Start with | Escalate to |
| --- | --- | --- |
| Open data, APIs, small or unprotected sites | Datacenter | Residential |
| Major retail, search engines, marketplaces | Residential (rotating) | Mobile |
| Social platforms and public feeds with aggressive anti-bot | Residential or mobile | Mobile |
| Flows that require login and stay logged in | Static residential / ISP | Mobile (rarely) |
| Heavily defended targets where nothing else passes | Mobile | Rethink the approach |

A few things that fall out of this:

**Datacenter proxies are the bandwidth bargain.** DataImpulse sells them at $0.50/GB, with sub-100ms response times. If you're crawling news archives, public directories, or your own internal sites, datacenter exits will do the job for half the price of residential. Reaching for residential on a site that would have accepted datacenter traffic is just burning budget.

**Residential is the workhorse.** Because the IPs belong to real ISP connections, requests look like ordinary home users. Independent benchmark work from Proxyway in April 2025 recorded a 99.51% overall success rate and 1.22s average response time across DataImpulse's residential pool, with the predictable drop on the hard targets — around 93.66% on Amazon and 65.30% on Instagram. That last number is worth sitting with, because it's the honest kind of figure: no proxy tier gets you to 100% on a platform with serious bot defense, and any provider claiming otherwise is describing a demo, not a workload.

**Mobile proxies cost more because carrier NAT makes them hard to block.** One mobile IP is shared by hundreds of real subscribers, so blocking it means blocking real people. That's why mobile sits at the top of the price ladder and why it's the right tool for a narrow set of targets, not a default.

**Static residential or ISP proxies exist for session-critical work.** If your collection requires authenticating and staying authenticated, mid-session rotation will break it. Most pure collection jobs don't need them; account-bound ones cannot work without them.

👉 [**Compare the proxy types and grab the 5 GB residential intro for $5**](https://bit.ly/dataimPulse)

## Rotation: per-request vs sticky sessions

Having a pool isn't a strategy. How you rotate is.

**Per-request rotation** suits stateless collection — independent product pages, no login, no cart. Every request gets a fresh exit, so nothing builds up. On DataImpulse, rotating sessions run on port 823 for HTTP/HTTPS and 824 for SOCKS5.

**Sticky sessions** suit anything multi-step: a paginated search, a flow that sets cookies, a checkout funnel. You hold one exit for a window — DataImpulse allows 1 to 120 minutes, with 30 minutes as the default if you don't specify — so the site sees a coherent visit instead of one that teleports between continents.

Two mistakes show up constantly:

- **Over-rotating on stateful flows.** A new IP on every request during a paginated search looks *less* human than a single IP would. Match rotation to the interaction.
- **Ignoring geography.** If a session hops Germany → Brazil → Japan across three requests, you've described a bot in one sentence. Pin a country per session.

Country-level targeting is included in DataImpulse's base per-GB price. City, ZIP, and ASN selection are billed at double the standard residential rate, which matters if your workflow needs hyper-local exits — a job that's mostly city-targeted is effectively a $2/GB job, not a $1/GB one. Datacenter proxies list those same filters as included, and if your budget depends on it, confirm the current billing treatment with support before you scale.

## Setting it up: credentials in, data out

DataImpulse is a raw-proxy provider, not a managed scraping API. There's no "give us a URL, get JSON" endpoint. You handle requests, parsing, retries, and CAPTCHA logic yourself, which is also why the price looks the way it does. For teams already writing their own collectors, that's a feature.

The gateway is a single host, and behavior is controlled through the username string — which is where most of the practical power sits:

- `_country-us` pins the exit country
- `_city-newyork` narrows to a city (billed at the advanced rate)
- `_session-abc123` holds the same IP for a sticky session

In Python, a rotating HTTP setup looks about like this:

python
import requests

proxy = "http://USER:PASS_country-us@gw.dataimpulse.com:823"
proxies = {"http": proxy, "https": proxy}

r = requests.get("https://example.com/product/123", proxies=proxies, timeout=20)


Sticky sessions use a port in the 10000–20000 range instead, with the session string in the credentials. Top-ups are handled from the dashboard, and unused traffic stays in the account rather than evaporating at month end — the single most useful thing about a pay-as-you-go model if your scraping load is bursty.

## The half proxies don't fix

This is where a lot of "I bought residential proxies and still get blocked" threads end up. The IP is one signal among several.

- **Send believable headers as a consistent set.** A real browser user-agent plus matching `Accept`, `Accept-Language`, and `Accept-Encoding`. Rotate the user-agent alongside the IP, not independently of it.
- **Persist cookies within a session.** Browsers keep state across a visit. Many scrapers throw it away and then look like a thousand amnesiac strangers arriving in sequence.
- **Pace requests like a person.** Somewhere in the 1–5 second range between requests to the same domain is a common working heuristic. Ten workers at one request per second beats one worker at ten per second, because concurrency limits apply per IP — and with rotating exits, you're spreading load across thousands of them.
- **Match TLS and JavaScript to your story.** On JS-heavy targets, a headless browser behind a residential proxy will outperform raw HTTP requests, because it produces a browser fingerprint as a side effect rather than having to fake one.

Then watch the data, not just the transport. A job can finish with flawless network statistics while returning a login page for every URL. Track expected-page rate, duplicate counts, empty fields, and requested market versus returned locale. Those checks catch the failures that proxy dashboards can't see.

## Where your bandwidth actually goes

Since residential pricing is per gigabyte, bandwidth discipline is cost control:

- A plain HTML page runs roughly 50–200 KB.
- The same page with images can hit 500 KB–5 MB.
- A full headless-browser render can consume 1–10 MB per page.
- API endpoints are usually 1–50 KB.

That's a 20–50× spread between the cheapest and most expensive way to fetch the same page. If you only need text, disable image and CSS loading, send `Accept-Encoding` for compression, cache responses so you don't re-scrape unchanged pages, and filter your URL queue before the requests go out rather than after.

## Every plan on the current pricing page

DataImpulse prices by traffic, with no subscription. The minimum first purchase is $5 across all four proxy types, and bought traffic doesn't expire. Here's the full published ladder.

**Residential proxies** — 90M+ IPs, rotating and sticky sessions, HTTP(S) and SOCKS5, free country targeting.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 5 GB | $5 | $1.00 | [Start with 5 GB for $5](https://bit.ly/dataimPulse) |
| Basic | 50 GB | $50 | $1.00 | [Buy 50 GB of residential traffic](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $800 | $0.80 | [Get the 1 TB volume tier](https://bit.ly/dataimPulse) |
| Custom | 5 TB+ | From $4,000 | Custom | [Talk to sales about 5 TB+](https://bit.ly/dataimPulse) |

**Mobile proxies** — rotating and sticky, 3G/4G/5G/LTE, country targeting.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 2.5 GB | $5 | $2.00 | [Try mobile proxies for $5](https://bit.ly/dataimPulse) |
| Basic | 25 GB | $50 | $2.00 | [Buy 25 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $1,600 | $1.60 | [Get the 1 TB mobile tier](https://bit.ly/dataimPulse) |
| Custom | 5 TB+ | From $8,000 | Custom | [Request 5 TB+ mobile pricing](https://bit.ly/dataimPulse) |

**Datacenter proxies** — 99.9% uptime target, randomized subnets, sub-100ms response times.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 10 GB | $5 | $0.50 | [Start with 10 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Basic | 100 GB | $50 | $0.50 | [Buy 100 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Advanced | 1 TB | $450 | $0.45 | [Get the 1 TB datacenter tier](https://bit.ly/dataimPulse) |
| Custom | 5 TB+ | From $2,250 | Custom | [Request 5 TB+ datacenter pricing](https://bit.ly/dataimPulse) |

**Premium residential proxies** — filtered high-trust pool, all targeting included, dedicated account manager.

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Intro | 1 GB | $5 | $5.00 | [Test the premium residential pool](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Basic | 10 GB | $50 | $5.00 | [Buy 10 GB of premium residential](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Custom | 5 TB+ | From $20,000 | Custom | [Request 5 TB+ premium pricing](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

Universal across every tier: no subscription, traffic that doesn't expire, free country targeting, and pay-as-you-go billing. Volume discounts land at the 1 TB mark rather than creeping in gradually — between 5 GB and roughly 850 GB, residential stays flat at $1/GB.

Two practical notes that don't show up in the headline price. There's no free trial, so the minimum $5 purchase is your entry point; HostAdvice's review of the terms notes that Intro plans carry a 7-day money-back guarantee on card payments, provided less than 80% of the traffic has been consumed, and that crypto-funded Intro purchases aren't refundable. And if your target is a platform with login- or account-based rules, be honest about which category you're in: DataImpulse isn't built for mass account registration, and providers that say no to that traffic are usually the ones that stay online.

## Cost per successful request, not cost per GB

The advertised $/GB is the wrong number to compare on, because a cheap exit that fails three times costs more than an expensive one that succeeds. Compare cost per validated record instead.

Illustrative arithmetic, not a benchmark: at $1/GB and 120 KB per page, one gigabyte covers roughly 8,000 page fetches — about $0.000125 per fetch. Add a 5% block rate with two retries on failures and you're paying for roughly 1.1 fetches per record, call it $0.00014 per validated row. Now swap in a network that blocks 20% of the time with heavier retry logic and that number nearly doubles, even at a nominally identical per-GB price. Multiply by the value of the dataset and the reason to pay for success rate rather than shelf price becomes obvious.

The corollary on the other side: if you're spending $5–$20 a month total, you are not optimizing your way to meaningful savings. You're testing. Run a representative sample of your real page types and regions against the network, hold your selectors and concurrency constant, and pick the cheapest tier that meets your threshold.

## Where DataImpulse fits, and where it doesn't

The pitch is deliberately unglamorous: $1/GB residential, no subscription, traffic that doesn't expire, country targeting included, HTTP(S) and SOCKS5, rotating and sticky sessions, and a 90M+ first-party pool the company builds through TraffMonetizer, its opt-in bandwidth-sharing app. That's the same corporate group as DataForSEO, and the parent company's other products explain the data-collection bent. The provider launched in 2022 and picked up Proxyway's "Newcomer of the Year" in 2024 and a "Greatest Progress" nod in 2025.

What that same profile means for you: it's a four-year-old brand with a thinner independent audit trail than decade-old incumbents, and pool depth in Tier-3 geographies (parts of Central Asia and sub-Saharan Africa) is thinner than the enterprise names. Gologin's mobile proxy testing also flagged repeated IP addresses within some regional pools and a relatively small footprint per country. If your job is a US-heavy crawl of mainstream retail and SEO targets, that's not an issue. If you need reliable exits in twenty small markets with no repeats, it is — and paying three to eight times more elsewhere would be the rational choice.

What it's clearly good at: continuous price comparison work, SEO and SERP monitoring, price and stock checks, ad verification, and mid-volume distributed crawling where you want the network to be your only fixed cost and don't want to plan your month around a bandwidth allotment. Pair it with sane concurrency, header hygiene, and retry logic, and the residential tier will cover most of what "web scraping with proxies" actually means in practice.

## Legal and ethical guardrails worth knowing

Proxies change where your traffic appears to come from. They don't change what you're allowed to collect.

The landscape is roughly this: in the US, the Ninth Circuit's hiQ v. LinkedIn decision held that scraping publicly available data doesn't violate the Computer Fraud and Abuse Act, while breach-of-contract and terms-of-service arguments remain live. In the EU, text and data mining is permitted for research purposes under the Copyright Directive with opt-out mechanisms, and GDPR and CCPA still govern personal data, meaning you need a lawful basis and have to honor data subject rights. Practically: read robots.txt, don't scrape personal data without a legal basis, throttle so you don't degrade a site for real users, and don't touch accounts or areas you aren't authorized to access. That's a compliance question, not a proxy configuration question — no provider's pool changes the answer.

## Quick answers

**Do I need residential proxies for scraping?** Only if the target scores datacenter IPs. Start with datacenter, measure the block rate, and escalate to residential when the numbers tell you to.

**How many proxies do I need?** With rotating exits, the question is bandwidth, not IP count — each request gets a different address from the pool. Rough planning figure for defended targets: one concurrent slot per 10–50 requests per hour, per target.

**Can I scrape JavaScript-rendered pages?** Yes, with a headless browser routed through the proxy. Budget 5–20× the bandwidth of a plain HTTP fetch, so only render when the content actually requires it.

**What about CAPTCHAs?** They're usually a reputation signal, not a wall. Retry on a different exit, slow the rate, fix your headers. If they persist, add a headless browser or a solving service.

**Is traffic really non-expiring?** Yes — purchased gigabytes stay in the account until they're consumed, which is the main structural difference from subscription-based providers.

**Can I test before committing?** The $5 Intro plan is the test. Five gigabytes on residential is enough to run a real sample of your targets and measure your own success rate rather than trusting someone else's.

If you're at the stage where a scraper works on ten pages and dies on the eleventh, the fix is usually less about finding a cleverer provider and more about matching the tier to the target and cleaning up the request itself. Start small, measure cost per validated record, and scale the tier that passes — not the one with the loudest benchmark page.

👉 [**Set up DataImpulse proxies and start with the $5 / 5 GB residential intro**](https://bit.ly/dataimPulse)
