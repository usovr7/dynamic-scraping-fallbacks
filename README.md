# Scrape Dynamic Websites Python Can't Reach: Why `requests` Comes Back Empty, Three Fixes That Work, and When a Scraping API Beats Running Your Own Browser

A `requests.get()` that returns 200 while BeautifulSoup finds nothing means the data was never in the HTML you downloaded. JavaScript built it afterwards, in the browser, from a separate response. You have three ways out: render the page with a real browser, call the endpoint the page itself calls, or let an API handle rendering and proxy rotation for you. Which one you need depends on where the data actually lives, and that takes about thirty seconds to check.

> **The short version:** search the raw page source for one string you can see on screen. If it is missing, a browser or an API is doing the work. ScraperAPI's free plan carries 1,000 credits a month with 5 concurrent connections, and the 7-day trial adds 5,000 credits without asking for a card. 👉 [Compare the plans and claim the trial credits](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Check the page source before you install anything

Open the target page, press Ctrl+U, and search for a word that is clearly visible on the rendered page. "View page source" shows the HTML the server sent before any JavaScript ran. The Inspect tool shows the DOM after JavaScript has executed and rewritten it. Those are two different documents, and only the second one contains the data you want.

A concrete case: on the AJAX practice page at scrapethissite.com, clicking "2015" loads that year's Oscar winners, *Spotlight* included. Search the page source for "Spotlight" and you get nothing. The film table is fetched and injected by JavaScript after the click. The same test in Python:

python
import requests

html = requests.get("https://www.scrapethissite.com/pages/ajax-javascript/").text
print("Spotlight" in html)   # False: the table is built client-side


The distinction that trips people up is parsing versus rendering. BeautifulSoup parses: it turns an HTML string into Python objects you can query. Rendering is the browser executing JavaScript and building the DOM. `requests` only ever hands you the pre-render string, so BeautifulSoup has nothing to find. It is not that BeautifulSoup is weak; it is being fed the wrong document.

One useful side effect of that test: when JavaScript is disabled, some sites fall back to plain pagination or a static list. If yours does, you may not need a browser at all.

## Build or buy: the two numbers that decide it

Local browser automation is free and unlimited, and it costs you a Chrome process per worker, a driver to keep in sync, and a proxy bill the moment your target starts blocking IPs. ScraperAPI charges in credits instead, and the credit price of a page varies enormously by domain. That variation matters more than the plan name.

| What you are fetching | Credits per request |
| --- | --- |
| Standard HTML page | 1 |
| Amazon (all Amazon domains) | 5 |
| Google or Bing, including subdomains | 25 |
| LinkedIn | 30 |
| Cloudflare, DataDome or PerimeterX bypass | +10 on top of the base cost |

The bypass surcharge is applied automatically when ScraperAPI activates its bypass logic. A 100,000-credit allowance therefore buys 100,000 ordinary pages, 4,000 Google queries, or roughly 3,333 LinkedIn profiles. Budgets built on the 1-credit figure blow up when the target list is mostly SERPs or marketplace pages.

Two controls exist for exactly that: a Domain Cost Estimator in the dashboard that quotes the exact cost of any URL, and a `max_cost` parameter you can set per request so a single call cannot burn more than you allow.

Every paid tier includes the same core stack: JavaScript rendering, rotating proxy pools with residential and mobile IPs, anti-bot bypass, CAPTCHA handling, JSON autoparse, custom headers, automatic retries, unlimited bandwidth and a 99.9% uptime guarantee. What changes between tiers is volume, overage behaviour and price.

| Plan | Where it fits | Limits and overage behaviour | Get started |
| --- | --- | --- | --- |
| Free | Testing the API against your own targets | 1,000 credits, max 5 concurrent connections, no card | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | First paid tier, small recurring jobs | Hard stop at the limit; auto-upgrade available | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | Growing pipelines with steady monthly volume | Hard stop at the limit; auto-upgrade available | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | Team-scale workloads on ordinary pages | Hard stop at the limit; auto-upgrade available | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | Heavy or unpredictable volume | Pay-as-you-go past the limit at a fixed per-credit rate, with a monthly cap you set | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | Production jobs that cannot pause | Pay-as-you-go plus auto-upgrade | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | High sustained volume | Pay-as-you-go plus auto-upgrade | [See plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom volume and terms | Custom pricing, quoted by sales | [Talk to sales](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Three billing facts worth knowing before you pick a tier. Annual billing is exactly 10% cheaper than monthly, and annual plans can renew early for a fresh batch of credits. Credits reset on each monthly renewal date and unused ones do not roll over. Exceeding your allowance never triggers an overage charge: you get a 429, the request is turned away rather than queued, and scraping pauses until credits renew unless auto-upgrade or pay-as-you-go is switched on.

## Fix 1: Selenium, when you need to click your way to the data

Selenium drives a real Chrome or Firefox instance, executes the page's JavaScript, and gives you the finished DOM. Six commands cover most jobs:

- `webdriver.Chrome()` launches the browser
- `.get(url)` opens a page
- `.find_element(by, value)` and `.find_elements(by, value)` locate one or many elements
- `.click()` interacts with an element
- `.page_source` returns the full HTML after JavaScript has run
- `.quit()` closes the browser

Skip the manual ChromeDriver download. `webdriver-manager` fetches the build that matches your installed Chrome:

python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from webdriver_manager.chrome import ChromeDriverManager
from bs4 import BeautifulSoup

options = Options()
options.add_argument("--headless")
options.add_argument("--disable-gpu")
options.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                     "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.6778.265 Safari/537.36")

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
driver.get("https://www.scrapethissite.com/pages/ajax-javascript/")
driver.find_element(By.ID, "2015").click()

# wait for a selector inside the loaded table, then grab the rendered HTML
html = driver.page_source
driver.quit()

soup = BeautifulSoup(html, "lxml")


Headless mode runs the same browser without a visible window, which is what you want on a server. The user-agent string matters: an empty one is a detectable signal, so set something realistic.

The line you should not copy from older tutorials is `sleep(3)`. It works when the network is fast and fails silently when it is not. Wait for the element you actually need:

python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

WebDriverWait(driver, 30).until(
    EC.presence_of_element_located((By.CLASS_NAME, "soft-link"))
)


That pattern comes from a job-board scrape where the links only appear after JavaScript finishes, and the wrong wait returns an empty list rather than an error, which is the worst failure mode a scraper can have.

## Fix 2: Playwright, when you want less infrastructure

Playwright automates Chromium, Firefox and WebKit and ships its own browser binaries, so there is no driver version to match. Two commands:


pip install playwright
playwright install chromium


The skeleton, with the infinite-scroll loop most dynamic sites eventually demand:

python
from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeout

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com/infinite-scroll")

    while True:
        page.evaluate("window.scrollTo(0, document.body.scrollHeight);")
        try:
            page.wait_for_selector(".new-content", timeout=1000)
        except PlaywrightTimeout:
            break

    html = page.inner_html("body")
    browser.close()


Both libraries render JavaScript, so neither is more "correct". The practical difference is setup: Playwright installs its own browsers and its selectors auto-wait, while Selenium leans on the Chrome you already have. Playwright also handles scrolling, clicks and form fills in the same API, which is handy when the data only appears after interaction.

## Fix 3: call the AJAX endpoint and skip the browser entirely

Loading a browser costs CPU, RAM and bandwidth for CSS, images and layout you never read. You want the data, and the data almost always arrives in one of two places: embedded as JSON inside a `<script>` tag on the page itself, or fetched from a separate endpoint you can call directly. This is the fastest route, and the one most tutorials skip because it requires reading network traffic rather than writing browser code.

**Case one, data on the same page.** Open `quotes.toscrape.com/js` in Chrome, hit Ctrl+U, and search for "Albert". Inside the source you will find a JSON array where each item carries `tags`, `text`, and an `author` object with a name, a Goodreads link and a slug, including Einstein's "The world as we have created it is a process of our thinking." No browser needed:

python
import json, re, requests

html = requests.get("https://quotes.toscrape.com/js/").text
start = html.index("[{")
end = html.index("}]", start) + 2
quotes = json.loads(html[start:end])

for q in quotes:
    print(q["author"]["name"], q["text"][:60])


The author slugs in that payload are also the entry point to every author page, which is how you turn one request into a full crawl.

**Case two, data on another endpoint.** Librivox renders a search results page at one URL, but the actual search runs against `advanced_search`. Reproduce the request your browser made and you get JSON back:

python
import requests
from bs4 import BeautifulSoup

headers = {"X-Requested-With": "XMLHttpRequest"}
url = ("https://librivox.org/advanced_search?title=&author=&reader=&keywords=&genre_id=0"
       "&status=all&project_type=either&recorded_language=&sort_order=alpha"
       "&search_page=1&search_form=advanced&q=The%20Time%20Machine")

data = requests.get(url, headers=headers).json()
soup = BeautifulSoup(data["results"], "lxml")
for a in soup.select("h3 > a"):
    print(a.text)


The `X-Requested-With: XMLHttpRequest` header is the part people miss. Drop it and the server may hand you a full HTML page instead of the JSON fragment. To find these calls yourself, open DevTools, switch to the Network tab, filter by Fetch/XHR, and interact with the page. Whatever appears is a candidate you can replay with `requests`.

## Where DIY stops making sense

Seven things break a home-grown scraper, and they tend to arrive together on the sites worth scraping:

- JavaScript rendering, so plain HTTP requests return an empty shell
- Pop-ups and overlays that block the content underneath
- Dropdowns and live search suggestions that only render after input
- Real-time updates that change faster than you can snapshot them
- Sortable tables whose data reorders with every interaction
- Infinite scroll that requires custom logic to reach the bottom
- Anti-bot protection: CAPTCHAs and IP blocks

The last one is what usually ends the DIY phase. You can rotate user agents with `fake-useragent`, add randomised delays, and route CAPTCHA challenges through third-party solving libraries, but every one of those is a moving part you now maintain, plus a residential proxy bill. A managed API collapses rendering, proxy rotation and bypass into a single request, and the pricing above is the honest comparison: at 1 credit per ordinary page you are paying for someone else's browser fleet and IP pool, and at 25 credits per Google query you are paying for a job that is genuinely hard to do yourself.

If your target list is mostly ordinary pages and you scrape a few hundred a week, stay local. If it is thousands of pages a month behind Cloudflare, run the numbers on credits. 👉 [Check which plan covers your monthly credit total](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Which route fits your case

- **Data sits in a `<script>` block or an XHR response:** `requests` plus `json.loads`, nothing else. No browser, no credits spent.
- **A handful of pages, one-off, no blocking:** local Selenium or Playwright. Cheapest and fastest to write.
- **Needs clicking, logging in or scrolling on a small scale:** Playwright locally, with explicit waits.
- **Recurring jobs across hundreds to thousands of pages:** an API. The maintenance time alone exceeds the subscription.
- **SERP or marketplace data at volume:** do the credit arithmetic first. Google at 25 credits and Amazon at 5 change the plan you need by an order of magnitude.
- **Delivery deadlines, no infrastructure appetite:** API from day one.

## FAQ

**Can BeautifulSoup render JavaScript?**
No. It parses HTML you already have. Rendering is the browser's job, which is why you either render first and pass `driver.page_source` to BeautifulSoup, or skip HTML and parse the JSON endpoint directly.

**Headless or normal browser mode?**
Same HTML, no visible window. Headless is what you run on a server, and a realistic user-agent string matters because an empty one is easy to fingerprint.

**How do I scrape infinite scroll?**
Scroll to `document.body.scrollHeight` in a loop and wait for a new-content selector each time, as in the Playwright snippet above. Before writing that loop, test the page with JavaScript disabled: some sites fall back to ordinary pagination, which is far easier to crawl.

**Do I still need to download ChromeDriver by hand?**
No. `webdriver-manager` resolves the matching driver, and Playwright installs its own browsers with `playwright install`.

**What happens when I hit the credit limit?**
You get a 429 and the request is refused rather than queued, with no overage charge. Depending on your plan you can enable auto-upgrade to the next tier, or pay-as-you-go at a fixed per-credit rate up to a cap you set.

**Is there a refund policy?**
Yes, 7 days, no questions asked. Combined with the free plan and the 5,000-credit trial, that is enough runway to test your real targets before committing to a year. 👉 [Start with the trial and test your own URLs](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Before you pay

Let the target decide the tool. If the data is sitting in a `<script>` tag, you are ten lines of Python away from finishing and should not buy anything. If it needs a rendered DOM once a week, install Playwright and move on. If it needs rendered DOMs, residential IPs and Cloudflare bypass every day, the subscription is the cheaper line item.

Two things to settle before checkout. Run your actual URLs through the Domain Cost Estimator and set `max_cost`, because the 1-credit headline is not what Google or Amazon pages cost. Then decide monthly versus annual: the yearly option is 10% cheaper, but only worth it once you know your real monthly credit burn rather than your estimate. 👉 [See the full plan comparison and current rates](https://www.scraperapi.com/pricing/?fp_ref=coupons)
