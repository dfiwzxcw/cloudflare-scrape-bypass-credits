# Cloudflare Scraper: Why the Python Bypass Stopped Getting Updates, What Cloudflare's Own /scrape and /crawl Really Do, and What Each Bypassed Request Costs (Plan Table + Credit Math)

If a target site greets you with "Checking your browser before accessing…", the free Python route is the first thing to try and the one most likely to eat your afternoon. cloudscraper, the package most tutorials still link to, last shipped version **1.2.71 on 25 April 2023**. It solves the JavaScript challenge on Cloudflare's "I'm Under Attack Mode" page and stops there: a reCAPTCHA challenge defeats it outright, and it requires you to keep using the same user agent and the same IP that solved the challenge.

A managed API sidesteps both constraints. ScraperAPI prices a Cloudflare bypass at **10 credits on top of the base request**, and JavaScript rendering at another 10, so a protected page that also needs rendering runs about 20 credits. The free tier is 1,000 credits at 5 concurrent connections, enough to find out whether the domain you actually care about comes back clean before you pay anything.

Below: what the challenge page really checks, the three routes out of it, the default settings on Cloudflare's own endpoints that quietly wreck a crawl, and the credit arithmetic that decides which plan you need.

> **Quick pick:** start on the free tier (1,000 credits, 5 concurrent connections, 7-day trial) and measure your success rate on the real domain. If it holds, Hobby's 100,000 credits cover roughly 5,000 Cloudflare pages that also need rendering. 👉 [Start the free trial](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The plan lineup and the credit allowances that decide your tier

Every ScraperAPI plan includes the same anti-bot toolkit: JS rendering, rotating and premium proxy pools, CAPTCHA and anti-bot detection, custom sessions, custom headers, automatic retries and unlimited bandwidth. The tiers differ on credits, concurrency and geotargeting, not on whether Cloudflare handling exists.

| Plan | API credits / month | Concurrent threads | Geotargeting | Price | Get started |
| --- | --- | --- | --- | --- | --- |
| Free trial | 1,000 | 5 | — | $0 | [ Free trial](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | — | [ Choose Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | — | [ Choose Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | — | [ Choose Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | $475/mo, or $427.50/mo billed annually | [ Choose Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | — | [ Choose Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | — | [ Choose Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Custom | [ Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Scaling is the one tier whose price is spelled out in the pricing page text: $475/month, or $427.50/month on annual billing, which works out to $5,130/year and a $570 saving. The tiers around it are priced by credit volume, so their live figures sit on their own plan pages. Analytics history splits at Business: the two smallest plans keep the last 30 days, everything from Business up keeps it unlimited.

Two plan mechanics matter more than the headline numbers. Hobby, Startup, Business, Scaling and Professional can switch on **auto-upgrade**, which moves your account to the next tier when credits run out. Only Scaling, Professional, Advanced and Enterprise get **pay-as-you-go**, the fixed per-credit overflow rate with a monthly cap you set. If your workload is spiky and you'd rather burst than change tiers, Hobby and Startup don't give you that option.

## What Cloudflare's challenge page actually checks

The interstitial is a JavaScript check. The browser has to execute the challenge script before it's let through, which is why every library in this space ships a JavaScript engine rather than a parser. Cloudflare's own description of the page, quoted in the cloudflare-scrape README, is that it "currently just checks if the client supports JavaScript," with the caveat that other techniques may be added.

Pass the check and you get two cookies: one proving you cleared the challenge page, one tracking the session. Everything that goes wrong afterwards goes wrong on one of three points.

**The challenge type.** cloudflare-scrape and cloudscraper both document the same boundary in the same words: "If there is a reCAPTCHA challenge, you're out of luck." Enough sites use the harder branch that you should check which one you're hitting before you build anything around a library.

**User agent consistency.** The integration functions return a cookie and a user agent string together, and the docs bold the rule: you must use the same user-agent string for obtaining tokens and for making requests with those tokens, otherwise Cloudflare flags you as a bot. Pass the cookie to curl with a different UA and the clearance is worthless.

**IP consistency.** If you mint a clearance token through a proxy, you have to keep using that proxy. Cloudflare requires the challenge-solving IP and the visitor IP to stay the same. This is the rule that kills rotating proxy pools built on top of a library: the pool rotates, the token doesn't follow it.

## Route 1: cloudscraper, and the 5-second tax

cloudscraper's API is barely an API. `cloudscraper.create_scraper()` returns an object that behaves like a `requests.Session`, so `scraper.get()` and `scraper.post()` replace `requests.get()` and `requests.post()`, and non-Cloudflare sites pass through untouched. For a small job against one site, that's about as little integration work as exists.

The dependency list is longer than the API. Because Cloudflare keeps hardening the protection page, the package needs a JavaScript engine, and it offers several: **native**, a self-written Python solver that is the default and ships with the package; **js2py** 0.67 or newer, pulled in as a default dependency; **ChakraCore**; **Node.js**; and **V8** through Sony's v8eval module. Older forks went different ways on this. The original Anorov repo requires Node.js, while the obskyr fork transpiles JavaScript to Python with Js2Py to stay pure Python. Whichever you pick, you own the engine install, and a broken engine looks exactly like a broken scrape.

There's also a fixed delay. The first visit to any Cloudflare-protected site sleeps about 5 seconds, with no delay after the first request. If a site is under heavy load the guidance is to raise it to 10 or 15 seconds through `create_scraper(delay=...)`, but only when the library itself errors and tells you to.

The part worth weighing is the release history. Version 1.2.71 landed in April 2023. Cloudflare "regularly modifies their anti-bot protection page and improves their bot detection capabilities," in the words of the library's own README. A bypass package whose last release is years old, aimed at a target that changes on its own schedule, has an obvious maintenance gap. That doesn't tell you it fails on your specific target this week. It tells you to test before you build a pipeline on it, and to have a second option ready.

## Route 2: Cloudflare's own /scrape and /crawl

Cloudflare sells browser automation against the same infrastructure everyone else is being blocked by, which sounds like an unfair advantage and is really just a rendering service with a few sharp edges.

### /scrape pulls elements, not pages

The endpoint is `POST https://api.cloudflare.com/client/v4/accounts/<accountId>/browser-run/scrape`. You authenticate either with an API token carrying the **Browser Run - Edit** permission or from inside a Worker through `env.BROWSER.quickAction("scrape", ...)`, which needs no token.

The payload requires an `elements` array, each item with a `selector`, plus exactly one of `url` or `html`. Each result comes back with `text`, `html`, `attributes` (href, for links) and the element's `height`, `width`, `top` and `left`. That makes it a decent fit for pulling headings, prices or repeated cards by CSS selector, and for grabbing title, description and canonical metadata.

The failure mode is JavaScript-heavy pages and SPAs, where the default load behaviour returns empty or partial results because the browser considers the page finished before the framework has rendered anything. The fix is `gotoOptions.waitUntil` set to `networkidle0` or `networkidle2`, or, faster, `waitForSelector` if you know which selector signals the content is in. Other parameters cover `userAgent` at the top level, `authenticate` for HTTP credentials, `cookies` and `gotoOptions`. What you're buying is a real browser's view of the page. If the page the browser lands on is the challenge interstitial, that's what you'll be extracting selectors from.

### /crawl has defaults that will cost you a rerun

`POST https://api.cloudflare.com/client/v4/accounts/{account_id}/browser-run/crawl` starts at one URL, follows links, and returns HTML, Markdown or JSON. The parameter defaults are where the surprises are, and they're the part most tutorials skip:

| Parameter | Default | What it does to you |
| --- | --- | --- |
| `limit` | 10 (max 100,000) | The crawl stops at ten pages. Ten. |
| `depth` | 100,000 (max 100,000) | No practical depth ceiling, so `limit` is your only brake |
| `maxAge` | 86,400 s (max 604,800 s) | Cache is served from R2 only when URL and parameters match exactly |
| `render` | `true` | `false` does a fast HTML fetch with no JavaScript execution |
| `crawlPurposes` | search, ai-input, ai-train | A robots.txt Content-Signal that says no to any declared purpose returns a 400 |
| `contentUse` | `full` | A stricter `use` directive on the target returns a 400 |
| `formats` | HTML | JSON runs through Workers AI by default and bills Workers AI usage |

The `crawlPurposes` default is the one that bites hardest. Because the request declares `ai-train` unless you say otherwise, a target site whose robots.txt carries a Content-Signal rejecting AI training will reject your crawl with a 400 before fetching a single page. Narrow it to `["search"]` and set `contentUse` to `reference` when that's all you need, and the same crawl goes through.

The rest of the surface is sensible once you know the defaults. `source` picks between `all`, `sitemaps` and `links`; `options.includeExternalLinks` and `options.includeSubdomains` both default to false; `includePatterns` and `excludePatterns` take wildcards where `*` stops at a slash and `**` crosses it; `modifiedSince` takes a Unix timestamp so you only re-crawl pages that changed. Every URL gets an outcome, from `success` through `robots_disallowed`, `crawl_purpose_disallowed`, `connection_error`, `timeout`, `too_large` and a handful of others, and the response reports `browserSecondsUsed` so you can see what the render cost. Note that the `skipped` set only counts URLs evaluated one by one, so it isn't an exhaustive list of everything filtered out, especially when your source is a sitemap.

## Route 3: a managed API and the credits a bypass burns

ScraperAPI's approach is to put the retry loop behind an HTTP endpoint. Per its anti-bot documentation, every response is checked against ban and CAPTCHA detection logic, and a detected block triggers an internal retry from a different IP and user agent, so you don't get a 200 carrying a block page. Permanent CAPTCHAs embedded in a page, the kind guarding a form or a "reveal contact details" button, are not solved; that needs a dedicated solver, or an Enterprise arrangement negotiated per domain. If your success rate on a site drops below 70%, support will look at the specific case.

The billing is where Cloudflare users need to read carefully, because the credit cost is not one credit:

| Request type | Credits |
| --- | --- |
| Standard page | 1 |
| Bot-protection bypass (Cloudflare, Turnstile, DataDome, PerimeterX/HUMAN, Akamai) | +10 |
| JS rendering (`render=true`) | +10 |
| Premium proxies | +10 |
| Ultra Premium (`ultra_premium=true`) | 30 |
| E-commerce (Amazon, Walmart, eBay) | 5 |
| SERP (Google, Bing) | 25 |
| Social (LinkedIn) | 30 |

Geotargeting, sessions, device type, output format and auto-parsing add nothing.

So a Cloudflare-protected product page that also needs rendering lands around 20 credits, which turns Hobby's 100,000 credits into roughly 5,000 pages rather than 100,000. That single multiplication changes which tier makes sense: if your whole workload is protected pages, price the plan at a fifth of its face value. Don't trust the arithmetic blindly either. Every response carries the actual charge in the `sa-credit-cost` header, and you can check any URL's cost up front with the Domain Cost Estimator in the dashboard or cap a single request with `max_cost`.

👉 [See the credit allowances on each plan](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Which route fits your job

**A few hundred pages, one site, plain IUAM page, no reCAPTCHA.** Try cloudscraper first; it's free and the integration is two lines. Budget time for the user agent and IP rules, and keep the whole run on a single exit IP.

**You're already on Workers and need specific fields off known pages.** Use `/scrape` with explicit selectors and `waitUntil` set, and skip the crawling overhead entirely.

**You need a whole docs site or catalogue as Markdown, robots.txt is permissive, and you control the account.** `/crawl` is the cheapest per page of the three because you're paying Cloudflare for browser seconds rather than per request. Set `limit` deliberately, tune `maxAge` against how often the site changes, narrow `crawlPurposes`, and set `render: false` for any static section of the site.

**Ongoing production scraping across many domains with mixed protection.** A managed API is the only one of the three that doesn't require you to track each target's challenge implementation. Run the free tier against your five hardest domains first; 1,000 credits is about 50 rendered Cloudflare pages, which is enough to see whether the success rate holds.

**You need to declare what the crawl is for.** Only `/crawl` exposes purpose and content-use declarations as request parameters. If your legal or data team needs that on record, the other two routes can't express it.

## Before you pay

Two things to confirm on your own traffic rather than on a pricing page. First, whether credits are consumed on requests that come back blocked; the internal retry means a single call can involve several attempts, and the `sa-credit-cost` header on a failing request is the fastest way to see what you were actually charged. Second, whether you're on a tier that can overflow. Auto-upgrade is available from Hobby up, pay-as-you-go only from Scaling up, and the difference between "my plan quietly got bigger" and "my rate stayed flat" is worth knowing before a spike rather than after.

If the free trial returns your target pages clean, Hobby covers about 5,000 rendered Cloudflare pages a month at 20 concurrent threads, and Scaling at $475/month ($427.50/month annually) covers roughly 250,000. 👉 [Compare the plans and start the 7-day trial](https://www.scraperapi.com/pricing?fp_ref=coupons)
