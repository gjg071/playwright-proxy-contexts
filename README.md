# Playwright proxy: global vs per-context setup, why a bad password times out instead of a 407, and how to rotate IPs without dropping a login (code and a failure checklist)

Set the proxy for the whole browser with `chromium.launch(proxy={...})`, and every page and context inherits it. Set it per session with `browser.new_context(proxy={...})` when each job needs a different IP. Either way the credentials belong in the `username` and `password` fields of that dictionary; packed into the server URL the way cURL accepts them, they are ignored and the navigation hangs until the timeout fires.

What follows covers the two configuration levels, the two authentication failures that eat the most debugging time, the rotation and sticky-session patterns that actually work, and what to check when the proxy is clearly carrying traffic and the site still blocks you.

> **If you would rather not run the proxy layer yourself:** ScraperAPI's Startup plan costs **$149/month**, or **$134.10/month on annual billing ($1,609.20/year)**, and includes 1,000,000 API credits plus 50 concurrent threads per month. It takes over the rotating pool, the session bookkeeping and the per-country routing you would otherwise build around `new_context()`, and the free tier gives you 1,000 credits to test against your own targets first. 👉 [See the current ScraperAPI plans and what a credit actually buys](https://www.scraperapi.com/pricing?fp_ref=coupons)

## All ScraperAPI plans at a glance

| Plan | API credits per month | Concurrent threads | Geotargeting | Price | Sign up |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | — | $0 | [Start free](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | — | [Check Hobby pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | **$149/mo**, or $134.10/mo billed annually | [Check Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | — | [Check Business pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | — | [Check Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | — | [Check Professional pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | — | [Check Advanced pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Custom quote | [Request Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

A dash means the monthly figure is set per credit volume and I could not confirm it from ScraperAPI's published pricing data at the time of writing; the live number for those tiers sits on the pricing page linked in each row. Two things in this table matter more than the headline credit count. Country-level geotargeting starts at Business, so Hobby and Startup are limited to US and EU exits. And unlimited analytics history also starts at Business; Hobby and Startup keep 30 days.

## Global proxy: one setting, every page

Passing `proxy` to `chromium.launch()` configures the entire browser instance. Every context and page created afterwards goes through it.

python
from playwright.sync_api import sync_playwright

proxy = {
    "server": "http://proxy.example.com:3128",
    "username": "usr",
    "password": "pwd",
}

with sync_playwright() as p:
    browser = p.chromium.launch(proxy=proxy)
    page = browser.new_page()
    page.goto("https://httpbin.dev/anything", timeout=10_000)


If your suite runs on Playwright Test, the same settings go into `playwright.config.ts` under `use.proxy`, and the built-in `request` fixture picks them up automatically, so API calls made inside tests go through the same exit as the browser:

ts
export default defineConfig({
  use: {
    proxy: { server: 'http://my-proxy:8080', username: 'user', password: 'secret' },
  },
});


The `server` value carries a scheme. `http://`, `https://` and `socks5://` are all accepted, and a bare `host:port` with no scheme is treated as an HTTP proxy. Getting that wrong is a common cause of "the proxy connects but nothing loads", particularly with SOCKS5 endpoints that people write without the prefix.

## Credentials: use `username`, and never put them in the URL

Two mistakes account for most proxy authentication failures in Playwright.

The first is the field name. Tutorials from 2021 use `user`. Current Playwright builds expect `username`, and older examples copied from those threads silently fail or drop the credential. If you are pasting from a Stack Overflow answer, check the date before the code.

The second is the URL format. Everywhere else in HTTP, `http://usr:pwd@proxy.example.com:3128` is a valid way to pass credentials, and cURL accepts it. Playwright does not apply it to the proxy handshake. In published tests of this exact case, the request sat for 30 seconds and raised a timeout rather than authenticating or returning an error.

python
# ignored by Playwright: hangs until timeout
proxy = {"server": "http://usr:pwd@proxy.example.com:3128"}

# correct
proxy = {
    "server": "http://proxy.example.com:3128",
    "username": "usr",
    "password": "pwd",
}


## A wrong password looks like a hang, not a 407

Proxy authentication failure should produce an HTTP 407. In Playwright, the same published tests found no 407 at all: `page.goto()` blocked for the full 30 seconds and raised `TimeoutError`. A developer watching that assumes the proxy is dead or the network is slow, and spends the afternoon on the wrong problem.

The fix costs three lines. Give the first navigation a short timeout and treat a timeout as a credential problem until proven otherwise.

python
try:
    page.goto("https://httpbin.dev/anything", timeout=10_000)
except TimeoutError:
    print("timed out: check the proxy username and password first")


Ten seconds is enough for a healthy proxy to answer a small page, so a timeout there is far more likely to be authentication than latency. Once you have confirmed the credentials, raise the timeout back to whatever your target needs.

## Verify that the proxy is really carrying the traffic

Do not trust a page that loaded. Load a page that echoes your IP.

python
response = page.goto("https://httpbin.dev/anything", timeout=10_000)
data = json.loads(page.inner_text("body"))
print(response.status, data["origin"])


Run the same script once without a proxy. If `origin` changes, the proxy is in the path. If it does not, the browser is dialling out directly and the configuration never applied, which usually means the `proxy` argument landed on `new_page()` or on a launch option that does not exist in your Playwright version.

## Per-context proxies, and why rotation means new contexts

A context is the smallest unit that can hold a proxy. A page cannot have its own. So "one proxy per page" is really "one page per context", and that constraint shapes every rotation strategy you will write.

python
context = browser.new_context(proxy=next_proxy())
page = context.new_page()
page.goto(url, timeout=15_000)
context.close()


Every context also gets its own cookie jar and storage, which is what makes parallel multi-account work possible in a single browser process. Rotating an IP therefore means creating a context with the next proxy from your pool, doing the work, and closing it. The cost of rotation is context creation, not a setting change.

A retry loop on top of that handles dead proxies without extra machinery:

python
def scrape(browser, url, attempts=3):
    for _ in range(attempts):
        context = browser.new_context(proxy=next(proxy_pool))
        try:
            page = context.new_page()
            response = page.goto(url, timeout=15_000)
            if response and response.ok:
                return page.content()
            print("bad response:", response.status if response else None)
        except Error as e:
            print("proxy failed:", str(e).splitlines()[0])
        finally:
            context.close()
    raise RuntimeError(f"all {attempts} attempts failed for {url}")


Three attempts with a different exit each time is enough to tell a dead proxy from a blocked target. If all three fail on the same URL, the problem is the target or your fingerprint, not the pool.

## Sticky IPs, logins and carts

Any flow where the server ties state to an IP address needs one exit for the whole flow. That means IP-bound logins, carts and checkout steps. Keep the entire task inside a single context, and turn on your provider's sticky-session option so that new connections from that session reuse the same exit IP.

A fixed proxy gateway does not by itself guarantee a fixed exit IP. If you are behind a rotating gateway that assigns an IP per connection, one context can still leave through several addresses. The sticky setting lives on the provider side, not in Playwright.

On the login side, `storageState` moves cleanly between a `BrowserContext` and an `APIRequestContext`. You can authenticate through an API call, export the state, and build a browser context that already has the cookies, which is faster and more stable than driving the login form each time. Note that cookies and IP are separate concerns: the published tests of per-context isolation only demonstrated that cookies do not leak between contexts. They did not test whether changing IP invalidates a reused login, so treat "new IP, same cookies" as something to verify against your own target rather than a rule.

## The proxy works and the site still blocks you

This is the gap most proxy tutorials skip. A proxy changes your exit IP and nothing else. In the same tests, the browser still announced `HeadlessChrome` in the user agent and still reported the machine's local timezone, even while every request went through the proxy. A site that fingerprints the browser rather than the IP sees exactly what it would have seen without the proxy.

Two things follow. Set the user agent and timezone explicitly in the context so they match the exit location, or accept that a clean residential IP behind a headless Chrome shell is still a flagged session. And check whether your proxy provider terminates the browser at the exit point, because that is the only setup where UA, timezone and IP agree by default.

One more trap that looks like a proxy problem: if `browserContext.route()` or `page.route()` seems to miss requests entirely, a Service Worker is probably intercepting them before your handler. Set `serviceWorkers: 'block'` on the context and the events reappear.

## Corporate proxies: NTLM, self-signed certificates and bypass lists

Inside a company network the proxy is usually mandatory and rarely simple. Three settings cover most of it.

Domain authentication such as NTLM needs the corporate credentials in the same `username`/`password` fields. Traffic inspection appliances present self-signed certificates that break `https://` navigation until you set `ignoreHTTPSErrors: true`. And internal domains plus local development servers should bypass the proxy entirely, both to avoid the extra hop and to prevent authentication loops on hosts that never needed it. Playwright also supports detecting and using the system-configured proxy, which helps on CI runners where the proxy address varies by environment.

## Where a managed API changes the arithmetic

Everything above is maintainable at small scale. At a few thousand requests a day across several countries it becomes a part-time job: dead proxies to evict, sticky sessions to pin, per-country pools to refill, and blocked responses to attribute correctly.

ScraperAPI moves that layer off your script. You send a URL and get back HTML or parsed JSON, with rotation, retries and geotargeting handled on their side. The parts of your Playwright job that were only ever "fetch this page through a clean IP" stop needing a browser at all; keep Playwright for the flows that require clicking, typing or waiting on dynamic content.

Budget for the credit multipliers before you pick a tier. A standard page costs 1 credit, but Amazon, Walmart and eBay cost 5, Google and Bing cost 25, and LinkedIn costs 30. Bot-protection bypass, JavaScript rendering and premium proxies each add 10 credits, and premium proxies combined with rendering costs 25. A million credits is not a million pages of Google. Every response reports its own cost in the `sa-credit-cost` header, the dashboard's Domain Cost Estimator prices a URL before you scrape it, and `max_cost` caps a single request.

Two operational details worth knowing before you commit. Exceeding your concurrency returns a 429 rather than an extra charge, and unused credits do not carry over; scraping pauses at renewal until the new batch lands. Auto-upgrade to the next tier is available from Hobby through Professional, and pay-as-you-go overflow exists from Scaling upward, which is the difference between "cut off mid-month" and "keep going at a fixed rate".

👉 [Compare ScraperAPI's tiers against your monthly page volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Which setup fits your job

A single script hitting public pages from your own machine needs no proxy at all, and adding one only adds a failure mode.

A test suite that checks geo-specific behaviour needs per-context proxies with a small fixed list, one context per region, closed after each check.

Multi-account work needs one context per account with its own cookies and a sticky exit per context, plus `storageState` so you are not re-running logins.

Large-scale collection is the case where the maintenance cost of a self-built pool overtakes the subscription cost of a managed one, and where credit arithmetic matters more than the price on the plan page. Start on the free tier with your three hardest URLs, multiply the real credit cost you observe by your monthly volume, and only then decide between Startup at $149/month and the tier above it.

👉 [Start with the free 1,000 credits and price your own targets](https://www.scraperapi.com/pricing?fp_ref=coupons)
