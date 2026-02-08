# 🧩 **Phase 4: Advanced Scenarios**  
*Logins, pagination, anti-bot evasion, and scaling — with heavy emphasis on ethics*

---

## ⚠️ **Critical Reminder Before We Begin**

| Scenario | Ethical/Legal Risk | Recommendation |
|----------|-------------------|----------------|
| **Login walls** | ⚠️ High (ToS violation likely) | ✅ Only scrape *your own account data* with explicit permission |
| **Bypassing paywalls** | ❌ Illegal (DMCA/CFAA violations) | ❌ Never attempt |
| **Scraping personal data** | ❌ Illegal (GDPR/CCPA) | ❌ Never attempt |
| **Ignoring rate limits** | ⚠️ Medium (can trigger blocks/lawsuits) | ✅ Always throttle requests |
| **Scraping competitors** | ⚠️ Medium-High (depends on jurisdiction) | ✅ Consult lawyer first |

> 💡 **Golden Rule:** *If you wouldn't want it done to your site, don't do it to others.*  
> 🔒 **For trading systems:** Most financial data providers explicitly forbid scraping. Use official APIs (Alpha Vantage, Yahoo Finance API, etc.).

---

## 🔐 **Scenario 1: Authenticated Scraping (Your Own Account Only)**

**Use case:** Exporting *your own* order history from a service you use.

### Method A: Session Cookies (Simplest)
```python
import requests

# 1. Log in manually once → copy cookies from DevTools → Application → Cookies
cookies = {
    "sessionid": "abc123...",
    "csrftoken": "xyz789..."
}

# 2. Use cookies in requests
session = requests.Session()
session.cookies.update(cookies)

response = session.get("https://example.com/my-data")
# Parse response as usual
```

### Method B: Form Submission (Programmatic Login)
```python
import requests

session = requests.Session()

# 1. Get CSRF token (if required)
login_page = session.get("https://example.com/login")
soup = BeautifulSoup(login_page.text, 'lxml')
csrf_token = soup.select_one('input[name="csrfmiddlewaretoken"]')['value']

# 2. Submit login form
login_data = {
    "username": "your_email@example.com",
    "password": "your_password",  # ⚠️ Never hardcode in production!
    "csrfmiddlewaretoken": csrf_token
}

response = session.post(
    "https://example.com/login",
    data=login_data,
    headers={"Referer": "https://example.com/login"}
)

# 3. Verify login success
if "Welcome" in response.text:
    print("✅ Login successful")
    # Now scrape authenticated pages
    data_page = session.get("https://example.com/protected-data")
```

> 🔒 **Security Practices:**  
> - Never commit passwords to Git (`import os; pwd = os.environ["PWD"]`)  
> - Use `.env` files with `python-dotenv`  
> - Rotate credentials regularly  
> - Delete sessions after use (`session.close()`)

---

## 📖 **Scenario 2: Pagination (Multi-Page Scraping)**

### Pattern A: URL Parameters (`?page=1`, `?page=2`)
```python
import requests
from bs4 import BeautifulSoup
import time

base_url = "https://example.com/products?page={page}"
all_products = []

for page_num in range(1, 6):  # Pages 1-5
    url = base_url.format(page=page_num)
    
    try:
        resp = requests.get(url, headers=headers, timeout=10)
        resp.raise_for_status()
        
        soup = BeautifulSoup(resp.text, 'lxml')
        products = soup.select('div.product')
        
        if not products:  # No more products → stop
            print(f"⚠️ Page {page_num} empty — stopping")
            break
        
        for prod in products:
            all_products.append({
                "name": prod.select_one('.name').text.strip(),
                "price": prod.select_one('.price').text.strip()
            })
        
        print(f"✅ Scraped page {page_num} ({len(products)} products)")
        time.sleep(2)  # Be polite!
    
    except Exception as e:
        print(f"❌ Error on page {page_num}: {e}")
        break  # Stop on failure (or implement retry)

print(f"\n📊 Total products scraped: {len(all_products)}")
```

### Pattern B: "Next" Button (Dynamic Pagination)
```python
from urllib.parse import urljoin

url = "https://example.com/products"
all_products = []

while url:
    resp = requests.get(url, headers=headers, timeout=10)
    soup = BeautifulSoup(resp.text, 'lxml')
    
    # Extract products...
    
    # Find next page link
    next_link = soup.select_one('a.next-page')
    url = urljoin(url, next_link['href']) if next_link else None
    time.sleep(2)
```

> 💡 **Pro Tip:** Always check for a "Last Page" indicator or stop when duplicate content appears.

---

## 🛡️ **Scenario 3: Handling Anti-Bot Measures (Ethically)**

### Common Protections & Legitimate Responses:

| Protection | What It Is | Ethical Response |
|------------|------------|------------------|
| **Rate Limiting** | 429 errors after too many requests | ✅ Slow down (`time.sleep()`) |
| **IP Blocking** | Temporary/permanent IP ban | ✅ Stop scraping; contact owner for API access |
| **CAPTCHA** | Human verification challenge | ✅ **STOP** — do not bypass (illegal) |
| **Cloudflare** | JavaScript challenge before content | ✅ Use official API or stop |
| **Honeypot Traps** | Hidden links bots follow | ✅ Respect `robots.txt`; don't follow hidden links |

### ✅ Legitimate Throttling Pattern:
```python
import time
from datetime import datetime

class PoliteScraper:
    def __init__(self, delay=2.0):
        self.delay = delay
        self.last_request = 0
    
    def get(self, url, **kwargs):
        # Enforce minimum delay between requests
        elapsed = time.time() - self.last_request
        if elapsed < self.delay:
            time.sleep(self.delay - elapsed)
        
        print(f"[{datetime.now().strftime('%H:%M:%S')}] GET {url}")
        resp = requests.get(url, **kwargs)
        self.last_request = time.time()
        return resp

# Usage
scraper = PoliteScraper(delay=3.0)  # 3 seconds between requests
resp = scraper.get("https://example.com/page1", headers=headers)
```

> ❌ **Never use:**  
> - CAPTCHA-solving services (illegal bypass)  
> - Residential proxy networks without consent (often violates ToS)  
> - Headless browser farms to evade detection (aggressive scraping)

---

## 🌐 **Scenario 4: Finding Hidden APIs (The Smart Way)**

Many sites load data via JSON APIs — *always prefer this over HTML scraping*.

### Step-by-Step Discovery:
1. Open DevTools → **Network tab**
2. Filter by **XHR** or **Fetch**
3. Reload page or trigger action (e.g., click "Load More")
4. Look for requests returning JSON with your target data
5. Inspect **Headers** → copy as `curl` → convert to Python `requests`

### Example: Reddit-like site
```python
# Instead of scraping HTML:
url = "https://example.com/api/posts?page=1&limit=20"
headers = {
    "User-Agent": "Mozilla/5.0",
    "Accept": "application/json",  # Critical!
    "Authorization": "Bearer YOUR_TOKEN"  # If required
}

resp = requests.get(url, headers=headers)
data = resp.json()  # Clean structured data!

for post in data['results']:
    print(post['title'], post['upvotes'])
```

✅ **Benefits:**  
- No HTML parsing needed  
- Data is structured (no regex hell)  
- Often faster and more reliable  
- Less likely to break on UI changes

---

## 📝 **Phase 4 Exercise**

**Task:** Scrape paginated product listings from `http://books.toscrape.com/catalogue/page-1.html` (pages 1-3) with these requirements:

1. Extract: title, price, availability status
2. Implement polite throttling (2-second delay between pages)
3. Stop if a page returns 404 or has no products
4. Save results to `books_paginated.csv`

**Starter code:**
```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

base_url = "http://books.toscrape.com/catalogue/page-{}.html"
all_books = []

for page_num in range(1, 4):  # Pages 1, 2, 3
    url = base_url.format(page_num)
    
    # Your code here:
    # 1. Fetch page with requests
    # 2. Check status code (stop on 404)
    # 3. Parse with BeautifulSoup
    # 4. Extract books
    # 5. Throttle with time.sleep(2)
    
    time.sleep(2)  # Politeness delay

# Save to CSV
df = pd.DataFrame(all_books)
df.to_csv("books_paginated.csv", index=False)
print(f"✅ Scraped {len(df)} books across {page_num-1} pages")
```

> 💡 **Hint:** Page 51+ returns 404 on this sandbox site — test your error handling!

---

### ✅ **Phase 4 Complete When:**
- [ ] You've implemented pagination (URL-based)
- [ ] You've added polite request throttling
- [ ] You've handled 404/empty page gracefully
- [ ] You understand *when not to scrape* (logins, paywalls, personal data)
- [ ] You've discovered a hidden API endpoint in DevTools

---

**Ready for Phase 5 (Production Systems: Scrapy, data pipelines, monitoring)?**  
Just say **"Next"** 👇
