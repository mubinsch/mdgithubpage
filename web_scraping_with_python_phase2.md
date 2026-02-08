# 🕸️ **Phase 2: Static Site Scraping**  
*Using `requests` + `BeautifulSoup` — the 80% solution for most scraping tasks*

---

## 🧭 **Why This Combo?**
| Tool | Role | Why It Wins |
|------|------|-------------|
| `requests` | Fetches HTML from URLs | Simple, reliable, minimal overhead |
| `BeautifulSoup` | Parses HTML → Python objects | Forgiving parser, intuitive selectors |
| `lxml` | Parser engine (under the hood) | Fastest parser for BeautifulSoup |

> ✅ **Perfect for:** Blogs, news sites, e-commerce product lists, documentation — any site where content exists in raw HTML (no JavaScript rendering required).

---

## 🧪 **Step 1: Warm-Up with a Practice Site**

We'll use **[Books to Scrape](http://books.toscrape.com/)** — a sandbox site *explicitly designed for learning scraping* (no legal/ethical concerns).

```python
import requests
from bs4 import BeautifulSoup
import time

# 1. Configure polite headers
headers = {
    "User-Agent": "LearningBot/1.0 (educational purpose)"
}

# 2. Make the request
url = "http://books.toscrape.com/"
response = requests.get(url, headers=headers, timeout=10)

# 3. Verify success
if response.status_code == 200:
    print("✅ Page fetched successfully!")
    print(f"   Content length: {len(response.text)} characters")
else:
    print(f"❌ Failed with status code: {response.status_code}")
```

> ⏱️ **Add `time.sleep(1)` between requests** in real projects. For this tutorial, we'll keep it minimal since it's a practice site.

---

## 🔍 **Step 2: Parse & Inspect the HTML**

```python
# Parse the HTML
soup = BeautifulSoup(response.text, 'lxml')

# Quick inspection tools:
print("\n=== PAGE SNAPSHOT ===")
print(f"Title: {soup.title.text.strip()}")
print(f"First 200 chars: {soup.body.text[:200]}...")
```

**Your turn:** Open `http://books.toscrape.com/` in a browser → **Right-click → Inspect** → find:
- The container holding book items (`<article class="product_pod">`)
- Book title location (`<h3><a title="...">`)
- Price location (`<p class="price_color">`)

---

## ✂️ **Step 3: Extract Data with CSS Selectors**

```python
# Target all book containers
books = soup.select('article.product_pod')

print(f"\nFound {len(books)} books on this page:\n")

# Extract details from each book
for book in books[:3]:  # Just first 3 for demo
    # Title (nested inside <h3><a>)
    title = book.select_one('h3 a')['title']
    
    # Price (text inside .price_color)
    price = book.select_one('.price_color').text.strip()
    
    # Availability (text inside .availability)
    availability = book.select_one('.availability').text.strip()
    
    print(f"📚 {title}")
    print(f"   💰 {price} | ✅ {availability}\n")
```

**Selector Cheat Sheet for This Page:**
| Target | Selector | Method |
|--------|----------|--------|
| All books | `article.product_pod` | `soup.select()` |
| First book only | `article.product_pod` | `soup.select_one()` |
| Book title | `h3 a` (then get `['title']` attr) | Attribute access |
| Price | `.price_color` | Class selector |
| Star rating | `p.star-rating` (check `['class']`) | Multi-class parsing |

---

## 💾 **Step 4: Structure & Save Data**

```python
import pandas as pd

# Full extraction loop
book_data = []

for book in books:
    try:
        title = book.select_one('h3 a')['title']
        price = book.select_one('.price_color').text.strip()
        availability = book.select_one('.availability').text.strip()
        rating_class = book.select_one('p.star-rating')['class']
        rating = rating_class[1]  # e.g., ['star-rating', 'Three'] → 'Three'
        
        book_data.append({
            "title": title,
            "price": price,
            "availability": availability,
            "rating": rating
        })
    except Exception as e:
        print(f"⚠️ Skipping malformed book: {e}")

# Convert to DataFrame
df = pd.DataFrame(book_data)
print("\n=== EXTRACTED DATA ===")
print(df.head())

# Save to CSV
df.to_csv("books.csv", index=False)
print("\n✅ Saved to books.csv")
```

---

## 🛡️ **Step 5: Production-Ready Patterns**

### Always wrap requests in error handling:
```python
def fetch_page(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            resp = requests.get(url, headers=headers, timeout=10)
            resp.raise_for_status()  # Raises HTTPError for 4xx/5xx
            return resp.text
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt+1} failed: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
    return None
```

### Always respect `robots.txt`:
```python
from urllib.robotparser import RobotFileParser

def is_allowed(url, user_agent="*"):
    rp = RobotFileParser()
    rp.set_url("http://books.toscrape.com/robots.txt")
    rp.read()
    return rp.can_fetch(user_agent, url)

print(f"Scraping allowed? {is_allowed('http://books.toscrape.com/')}")
# Returns: True (this site permits scraping!)
```

---

## 📝 **Phase 2 Exercise**

**Your mission:** Scrape the **first page** of `http://books.toscrape.com/` and:
1. Extract all 20 books' titles, prices, and star ratings
2. Convert star rating words ("Three", "Five") → numbers (3, 5)
3. Save results to `books_phase2.csv`

**Starter code:**
```python
# Fill in the blanks ↓
import requests
from bs4 import BeautifulSoup
import pandas as pd

url = "http://books.toscrape.com/"
# ... your code here ...

# Hint for rating conversion:
RATING_MAP = {"One": 1, "Two": 2, "Three": 3, "Four": 4, "Five": 5}
```

> 💡 **Tip:** Use browser DevTools → right-click an element → "Copy selector" to test selectors fast.

---

### ✅ **Phase 2 Complete When:**
- [ ] You can fetch a page with `requests`
- [ ] You can parse HTML with `BeautifulSoup`
- [ ] You can extract text/attributes using CSS selectors
- [ ] You can structure data into a pandas DataFrame
- [ ] You've saved scraped data to CSV

---

**Ready for Phase 3 (Dynamic Content / JavaScript Sites)?**  
Just say **"Next"** 👇
