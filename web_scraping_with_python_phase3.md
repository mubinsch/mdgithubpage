# ⚡ **Phase 3: Dynamic Content Scraping**  
*When JavaScript renders content after page load (React, Angular, SPAs)*

---

## 🔍 **Why This Phase Exists**

| Static Sites | Dynamic Sites |
|--------------|---------------|
| Content exists in raw HTML | Content loads *after* JS executes |
| `requests` + `BeautifulSoup` works | Raw HTML shows `<div id="app"></div>` (empty!) |
| Fast, simple, reliable | Requires browser automation |
| ✅ 80% of sites | ⚠️ 20% (but growing) |

**How to detect JS-rendered content:**
1. View page source (`Ctrl+U` or `Cmd+Option+U`)
2. Search for target text (e.g., product price)
3. **If missing in source → JS-rendered → need browser automation**

> 💡 **Pro Tip:** Always check the **Network tab** (DevTools → Network → XHR/Fetch) first! Many "JS sites" actually load data from hidden APIs — *scrape the API instead* (faster, more reliable).

---

## 🛠️ **Tool Choice: Playwright (Recommended)**

| Tool | Pros | Cons | Best For |
|------|------|------|----------|
| **Playwright** | Fast, modern, auto-waits, headless by default | Newer (smaller community) | ✅ **New projects** |
| **Selenium** | Mature, huge docs/examples | Slower, flakier waits | Legacy systems |
| **Requests-HTML** | Simple JS execution | Limited, unmaintained | Quick prototypes |

We'll focus on **Playwright** — the future-proof choice.

---

## 🚀 **Step 1: Install Playwright**

```bash
# Install library
pip install playwright

# Install browsers (Chromium, Firefox, WebKit)
playwright install chromium

# Verify
python -c "from playwright.sync_api import sync_playwright; print('✅ Playwright ready!')"
```

---

## 🧪 **Step 2: Basic Playwright Workflow**

```python
from playwright.sync_api import sync_playwright
import time

with sync_playwright() as p:
    # Launch browser (headless=False to see it run)
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    
    # Navigate to page
    page.goto("http://quotes.toscrape.com/js/")
    
    # ⚠️ CRITICAL: Wait for dynamic content to load
    page.wait_for_selector("div.quote")  # Waits until element appears
    
    # Optional: Wait for network to be idle (all JS/XHR done)
    # page.wait_for_load_state("networkidle")
    
    # Extract content
    quotes = page.query_selector_all("div.quote")
    
    print(f"Found {len(quotes)} quotes:\n")
    for quote in quotes[:3]:
        text = quote.query_selector(".text").inner_text()
        author = quote.query_selector(".author").inner_text()
        print(f'"{text}" — {author}')
    
    browser.close()
```

**Key Playwright Concepts:**
| Method | Purpose | Why It Matters |
|--------|---------|----------------|
| `page.goto(url)` | Navigate | Like typing URL in browser |
| `page.wait_for_selector("css")` | Wait for element | Prevents scraping empty containers |
| `page.wait_for_load_state("networkidle")` | Wait for all requests | Ensures JS/XHR fully loaded |
| `page.query_selector()` | Find first match | Like `soup.select_one()` |
| `page.query_selector_all()` | Find all matches | Like `soup.select()` |
| `element.inner_text()` | Get visible text | Respects CSS visibility |

---

## 🔎 **Step 3: The Hidden API Shortcut (Always Try First!)**

Many dynamic sites load data via XHR/Fetch requests. **Scraping the API is better than browser automation.**

**How to find hidden APIs:**
1. Open DevTools → **Network tab**
2. Filter by **XHR** or **Fetch**
3. Reload page → watch requests appear
4. Look for JSON responses with your target data
5. Replicate request with `requests`

**Example: Quotes with JS site**
```python
# Instead of Playwright, hit the API directly:
import requests

response = requests.get("http://quotes.toscrape.com/api/quotes?page=1")
data = response.json()

for quote in data['quotes'][:3]:
    print(f"'{quote['text']}' — {quote['author']['name']}")
```

✅ **Benefits:**  
- 10-100x faster than browser automation  
- No JS parsing overhead  
- More stable (APIs change less than HTML)  
- Lower resource usage  

> 📌 **Rule of thumb:** *Always inspect Network tab before reaching for Playwright/Selenium.*

---

## ⚠️ **Step 4: When Browser Automation Is Unavoidable**

Use Playwright when:
- Site uses anti-bot measures (Cloudflare, fingerprinting)
- Content requires user interaction (click "Load More")
- No visible API exists

**Example: Infinite scroll (click "Next" button)**
```python
from playwright.sync_api import sync_playwright

all_quotes = []

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("http://quotes.toscrape.com/scroll")
    
    while True:
        # Wait for quotes to load
        page.wait_for_selector("div.quote")
        
        # Extract current quotes
        quotes = page.query_selector_all("div.quote")
        for quote in quotes:
            text = quote.query_selector(".text").inner_text()
            author = quote.query_selector(".author").inner_text()
            all_quotes.append({"text": text, "author": author})
        
        # Check for "Next" button
        next_btn = page.query_selector("li.next a")
        if not next_btn:
            break  # No more pages
        
        # Click and wait for new content
        next_btn.click()
        page.wait_for_load_state("networkidle")
    
    browser.close()

print(f"Scraped {len(all_quotes)} total quotes")
```

---

## 🛡️ **Critical Production Considerations**

| Issue | Solution |
|-------|----------|
| **Flakiness** | Use explicit waits (`wait_for_selector`), not `time.sleep()` |
| **Detection** | Rotate user agents, disable automation flags:<br>`page.add_init_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")` |
| **Performance** | Reuse browser instances; avoid launching per request |
| **Resource use** | Headless mode saves RAM/CPU; close pages/browsers properly |
| **Maintenance** | JS sites break scrapers often — monitor for layout changes |

---

## 📝 **Phase 3 Exercise**

**Task:** Scrape quotes from `http://quotes.toscrape.com/js/` using **both methods**:

1. **Method A (Playwright):**  
   - Launch browser, wait for `.quote` elements  
   - Extract first 5 quotes + authors  
   - Print results

2. **Method B (Hidden API):**  
   - Inspect Network tab → find JSON endpoint  
   - Use `requests` to fetch data directly  
   - Extract same 5 quotes

**Compare:**
- Which was faster? (`time.time()` before/after)
- Which required less code?
- Which is more maintainable?

**Starter code for Method A:**
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("http://quotes.toscrape.com/js/")
    # ... your code here ...
    browser.close()
```

---

### ✅ **Phase 3 Complete When:**
- [ ] You can detect JS-rendered content
- [ ] You've launched Playwright and extracted dynamic content
- [ ] You've used explicit waits (`wait_for_selector`)
- [ ] You've found and scraped a hidden API (preferred method)
- [ ] You understand tradeoffs: speed vs. reliability vs. complexity

---

**Ready for Phase 4 (Advanced Scenarios: logins, pagination, proxies)?**  
Just say **"Next"** 👇