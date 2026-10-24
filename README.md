# selenium scraping data: Fix the Blocks, Proxies and CAPTCHAs That Break Your Scraper — Full Plan and Cost Breakdown for ScraperAPI

Selenium is the right tool when a page only renders its data with JavaScript. It's also the fastest way to get your IP rate-limited the moment you move from a practice sandbox to a real site. Search for selenium scraping data and you'll find dozens of tutorials that end at `driver.get(url)` — none of them tell you what happens on request 500, when the target site starts serving CAPTCHAs and your residential IP is burned.

This article covers both halves of the problem: how to actually wire a proxy service into a Selenium scraper without breaking it, and what the setup costs in practice. The examples use ScraperAPI, a scraping API that offers a proxy port mode built for exactly this use case — and, importantly, an official warning about the integration method that most tutorials get wrong.

> **Quick picks**
> Most Selenium users scraping a mix of normal pages and a few protected sites fit the **Hobby plan: $49/month (or $44.10/month billed annually) for 100,000 API credits and 20 concurrent requests**. Before paying anything, you can test with 5,000 free credits during a 7-day trial (no credit card required) or stay on the free plan with 1,000 credits/month.
>
> 👉 [Start the 7-day trial with 5,000 free credits](https://www.scraperapi.com/signup?fp_ref=coupons)

## Why Selenium scrapers break once they leave the tutorial sandbox

Selenium sends every request through a real browser, which solves the JavaScript problem and creates a new one: your traffic now comes from one IP, at browser-level fingerprint detail, in a pattern that's trivial to rate-limit. A script that works fine at 50 requests will start seeing 403s, CAPTCHA interstitials or empty shells once a site's bot protection wakes up.

The usual DIY answers — rotating residential proxy lists, CAPTCHA solving services, headless-browser fingerprint patches — each work partially and each is its own maintenance project. ScraperAPI takes a different approach: you route your traffic through its proxy infrastructure, and rotating IPs, CAPTCHA handling and anti-bot bypassing happen on their side. Every paid plan includes JS rendering, premium residential and mobile IPs, CAPTCHA and anti-bot detection, automatic retries and unlimited bandwidth, so these aren't upsells locked behind higher tiers.

For a Selenium workflow specifically, that means you keep writing your scraping logic exactly as before and change one thing: where the browser's traffic exits.

## ScraperAPI plans and prices: the full table

ScraperAPI's pricing page lists a free plan plus seven paid tiers, all billed in USD, with monthly and annual options (annual runs 10% cheaper). Here's everything as shown on the pricing page — worth double-checking against the live page before you subscribe, since published prices can change:

| Plan | Credits / month | Concurrent requests | Monthly price | Annual (per month) | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0 | — | [Sign up free](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | 100,000 | 20 | $49 | $44.10 | [Start Hobby trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | $149 | $134.10 | [Start Startup trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Business | 3,000,000 | 100 | $299 | $269.10 | [Start Business trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | $475 | $427.50 | [Start Scaling trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | $975 | $877.50 | [Start Professional trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | $1,975 | $1,777.50 | [Start Advanced trial](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Custom | Custom | Contact sales |

A few plan details that only show up if you read the fine print:

- **Geotargeting is limited on the two cheapest paid tiers.** Hobby and Startup can only target US and EU locations; country-level targeting starts at Business. If your Selenium scraper needs to appear as a user from a specific country outside those regions, Hobby won't do it.
- **Dashboard analytics history differs.** Hobby and Startup only keep the last 30 days of request analytics; Business and up get unlimited history. Minor, but annoying if you audit usage monthly.
- **Not every plan can pay its way past the limit.** Pay-As-You-Go (continuing to scrape at a fixed per-credit rate after exhausting your allowance) is only available on Scaling, Professional, Advanced and Enterprise. On Hobby through Professional monthly plans, your alternative is auto-upgrade.

## The credit system nobody explains before you get the bill

Most scraping API comparisons quote plan prices and stop there. That skips the part that actually determines your real cost: ScraperAPI doesn't charge one credit per request — it charges per request **based on the target domain and the parameters you use**.

The domain multipliers, per the official documentation:

| Target | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google, Bing (all subdomains) | 25 |
| LinkedIn | 30 |

On top of that, parameters add credits: `render=true` costs 10 extra, `premium=true` costs 10 extra, and using both costs 25. Requests to sites behind Cloudflare, Datadome, PerimeterX or Akamai trigger a bypass mechanism that adds 10 credits per scrape.

Put those together and the math changes fast. That Hobby plan's 100,000 credits is 100,000 ordinary pages — but only **4,000 Google search results pages** (25 credits each), or about **4,760 pages** if every request needs JS rendering behind Cloudflare protection (1 + 10 + 10 = 21 credits). If your Selenium project targets SERP data and you budget assuming one credit per page, you'll run out roughly 25 times sooner than expected.

Three things make this manageable:

1. **The cost is knowable in advance.** The dashboard's Domain Cost Estimator (also called Domain Multiplier) lets you look up the exact credit cost of any URL before you scrape, and the API Playground lets you run one real request and see actual usage. There's also a plain API endpoint, `https://api.scraperapi.com/account/urlcost`, that accepts the same parameters and returns the cost — handy if you want to check costs programmatically before a big job.
2. **You can cap any single request.** A `max_cost` parameter keeps one scrape from eating more credits than you intend.
3. **Every response header reports its own cost** via `sa-credit-cost`, so you can log actual consumption instead of estimating.

Billing is also narrower than you might expect: ScraperAPI only charges for successful requests — `200` and `404` responses — plus requests you cancel yourself before the 70-second window completes. Failed requests after retries don't eat your credits.

One free caveat worth knowing: `country_code`, `session_number`, `device_type`, `wait_for_selector`, `output_format`, `keep_headers` and `autoparse` all cost nothing extra. Geotargeting your requests, at least, is free — it's the plan-level geo restrictions (US/EU only on Hobby and Startup) that constrain you, not per-request fees.

## How to integrate ScraperAPI with Selenium (and the mistake that breaks everything)

The correct integration method is the proxy port, and the easiest implementation goes through **selenium-wire**. The official flow:

1. Install Selenium Wire: `pip install selenium-wire`
2. Point both HTTP and HTTPS traffic at ScraperAPI's proxy endpoint, using your API key as the proxy username:

python
API_KEY = 'YOUR_API_KEY'

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=option,
    seleniumwire_options=proxy_options
)


From there, every request the browser makes exits through ScraperAPI, and your Selenium code stays unchanged — same locators, same waits, same navigation.

The part almost no tutorial covers: **don't route Selenium through the API endpoint** (the `http://api.scraperapi.com/?api_key=...&url=...` style URL). It's tempting because it looks simpler, and ScraperAPI's own docs walk through why it fails. Two specific failure modes:

- When Selenium loads a page, it also requests relative assets — a stylesheet at `/static/main.css`, for example. Normally that resolves to the target site's server. Through the API endpoint, Selenium requests it from the API endpoint's domain instead, gets an error, and you end up with only the first HTML response. As the official guide puts it, that "kinda defeats the purpose of using a headless browser."
- Clicking relative links — the core of any Selenium flow that emulates a real user — fails for the same reason, so page navigation breaks.

If your scraper only fetches a single URL and never touches assets or navigates, the API endpoint technically works. For anything resembling a real browsing session, use the proxy port. Full working examples for both methods, including a scraper targeting a practice site, are in ScraperAPI's public GitHub repository of code examples.

A common debugging symptom worth flagging: if page assets aren't loading at all, you've probably used the API endpoint method instead of the proxy configuration above — switch to the proxy port setup and it resolves.

## What your Selenium project actually costs

With the credit system in hand, here's how to translate plan prices into real budgets.

Say your Selenium scraper pulls 20,000 product pages per month — half from Amazon, half from ordinary e-commerce sites — with JS rendering on all of them. An Amazon page costs 5 + 10 = **15 credits**; a standard page with rendering costs 1 + 10 = **11 credits**. That's 10,000 × 15 + 10,000 × 11 = **260,000 credits** per month. The Hobby plan's 100,000 falls well short; Startup's 1,000,000 at $149/month covers it with a wide margin, and annual billing brings it to $134.10/month.

Contrast that with a project scraping 5,000 ordinary article pages per month, no rendering: 5,000 credits, which doesn't fit the free plan's 1,000/month allowance — realistically it needs Hobby ($49), or you use the 7-day trial's 5,000 free credits for a short burst and re-evaluate.

The overage mechanics deserve attention too, because they decide whether an underestimate is annoying or expensive:

- **Default behavior:** scraping pauses until credits renew. No surprise charges.
- **Auto-upgrade** (opt-in, on Hobby through Professional monthly plans): when you hit 100% usage, your account moves to the next tier automatically — which usually means a better price per credit, but a bigger bill.
- **Pay-As-You-Go** (Scaling and up): keep scraping past your limit at a fixed per-credit rate, capped at a monthly spending limit you set. The docs are explicit that this is a spending *limit*, not a pre-charge — service won't exceed the cap unless you change it.

If you're budget-sensitive, the safe combination is: leave auto-upgrade off, set a max_cost on expensive domains, and check the Domain Cost Estimator before launching a new target. Rate uncertainty also has a soft landing: there's a **7-day no-questions-asked refund policy** if the service doesn't fit, and you can cancel anytime from the dashboard without a cancellation fee.

👉 [Check current plan prices and start a trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## When plain Selenium is still the right call

Honesty section: ScraperAPI isn't the answer for every selenium scraping data project, and pretending otherwise wastes your money.

If you're scraping a few hundred pages per month from sites with no meaningful bot protection — your own staging environment, public datasets, the classic practice sandbox — plain Selenium with no proxy layer costs $0 and works fine. Adding a paid API to that workflow buys you nothing.

Projects that are 100% SERP tracking at high volume may find specialist SERP vendors a better fit. ScraperAPI's own marketing positions itself as an alternative to Oxylabs, which tells you where the company sees its center of gravity — general-purpose scraping, not a single-vertical tool. Its SERP support works (25 credits per Google/Bing request, country targeting from Business tier up), but if search results are all you need, it's worth comparing dedicated tools before committing.

Where ScraperAPI clearly earns its price is the mixed workload: a Selenium scraper that hits normal pages, a couple of Amazon properties, some JS-heavy dashboards, and occasionally a Cloudflare-protected site. One integration covers all of it, and the credit system — for all its complexity — means you can see exactly what each target costs before committing.

## FAQ

**Is there a free version of ScraperAPI?**
Yes. The free plan includes 1,000 API credits per month with a maximum of 5 concurrent connections, no credit card required. For your first 7 days after signing up, you additionally get 5,000 free credits to test at larger scale, and you can contact support if you need more test credits.

**Does ScraperAPI work with Selenium specifically?**
Yes, through the proxy port at `proxy-server.scraperapi.com:8001`, best wired up with selenium-wire. Official code examples for Python Selenium exist in the company's GitHub repository, covering both the proxy method and (for completeness) the API endpoint method the docs recommend against.

**What happens when I run out of credits mid-month?**
By default scraping pauses until your plan renews. Depending on your tier, you can opt into auto-upgrade to the next plan, request a custom plan, or — on Scaling and above — use Pay-As-You-Go with a monthly spending cap you control.

**Can I get a refund?**
Yes — ScraperAPI offers a 7-day no-questions-asked refund policy. Contact support through the dashboard and they process it.

**Do failed requests consume credits?**
No. Only successful requests (200 and 404 status codes) are billed, plus requests you cancel within the 70-second processing window. Retries on failed requests don't drain your allowance.

**How much does scraping Google or Amazon cost?**
Amazon, Walmart and eBay cost 5 credits per request; Google and Bing (including subdomains) cost 25; LinkedIn costs 30. Sites behind Cloudflare, Datadome or PerimeterX add 10 credits per request when the bypass is used. Check any specific URL with the Domain Cost Estimator before committing.

## Before you pick a plan

Three quick checks will save you from the two most common wrong purchases:

1. **Estimate your real credit burn, not your request count.** Multiply each target type by its multiplier (standard 1, Amazon 5, Google/Bing 25, LinkedIn 30, +10 for rendering, +10 more for protected sites) and compare against plan allowances. Most "I need the $299 plan" conclusions turn into "Hobby was fine once I stopped rendering pages that don't need it."
2. **Match the tier to your geo needs.** Country-level targeting outside the US and EU starts at Business ($299/month); Hobby and Startup are US/EU only.
3. **Use the free testing window first.** The 7-day, 5,000-credit trial plus the API Playground will tell you your actual per-page costs on your real targets — which is the only number that matters for choosing between $49 and $149.

If your Selenium scraper is fighting blocks on more than a handful of sites, the Hobby plan with annual billing ($44.10/month effective) is the sensible entry point, and the refund window makes trying it low-risk.

👉 [Start your ScraperAPI trial with 5,000 free credits](https://www.scraperapi.com/signup?fp_ref=coupons)
