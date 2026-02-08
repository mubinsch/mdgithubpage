# 🏭 **Phase 5: Production-Grade Scraping Systems**  
*Scrapy framework, data pipelines, monitoring, and legal safeguards*

---

## 🤔 **Why Move Beyond `requests` + `BeautifulSoup`?**

| Project Size | Tool Recommendation | Why |
|--------------|---------------------|-----|
| **< 10 pages** | `requests` + `BeautifulSoup` | Simple, minimal overhead |
| **10–100 pages** | Custom script + throttling | Manageable with discipline |
| **100+ pages / ongoing** | **Scrapy** | ✅ Built-in concurrency, retries, pipelines, export formats |
| **Enterprise-scale** | Scrapy + proxies + monitoring | ✅ Scalable, maintainable, auditable |

> 💡 **Rule of thumb:** *If you're writing the same scraping logic for >3 sites, use Scrapy.*

---

## 🕷️ **Step 1: Scrapy Fundamentals**

### Install & Create Project
```bash
# Install Scrapy
pip install scrapy

# Create project structure
scrapy startproject bookscraper
cd bookscraper

# Create a spider (scraping module)
scrapy genspider books books.toscrape.com
```

### Project Structure
```
bookscraper/
├── scrapy.cfg              # Project config
└── bookscraper/
    ├── __init__.py
    ├── items.py            # Data schema (like Pydantic models)
    ├── middlewares.py      # Request/response hooks
    ├── pipelines.py        # Data processing pipeline
    ├── settings.py         # Throttling, concurrency, etc.
    └── spiders/
        └── books.py        # Your scraping logic ← WE EDIT THIS
```

---

## ✍️ **Step 2: Build a Production Spider**

Edit `bookscraper/spiders/books.py`:

```python
import scrapy
from bookscraper.items import BookItem  # We'll define this next

class BooksSpider(scrapy.Spider):
    name = "books"
    allowed_domains = ["books.toscrape.com"]
    start_urls = ["http://books.toscrape.com/catalogue/page-1.html"]
    
    # ✅ Built-in politeness (settings.py)
    custom_settings = {
        'DOWNLOAD_DELAY': 2,          # 2s between requests
        'CONCURRENT_REQUESTS': 1,     # One at a time (be extra polite)
        'ROBOTSTXT_OBEY': True,       # Respect robots.txt automatically
        'USER_AGENT': 'BookScraperBot/1.0 (educational)'
    }
    
    def parse(self, response):
        # Extract books on current page
        for book in response.css('article.product_pod'):
            item = BookItem()
            item['title'] = book.css('h3 a::attr(title)').get()
            item['price'] = book.css('.price_color::text').get()
            item['availability'] = book.css('.availability::text').get().strip()
            item['url'] = response.urljoin(book.css('h3 a::attr(href)').get())
            yield item
        
        # Pagination: follow "next" link
        next_page = response.css('li.next a::attr(href)').get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

---

## 🧪 **Step 3: Define Data Schema (`items.py`)**

Edit `bookscraper/items.py`:

```python
import scrapy
from scrapy.loader.processors import MapCompose, TakeFirst
import re

def clean_price(text):
    """Convert '£51.77' → 51.77 (float)"""
    return float(re.search(r'[\d.]+', text).group())

def clean_availability(text):
    """'In stock (22 available)' → 'In stock'"""
    return text.split('(')[0].strip()

class BookItem(scrapy.Item):
    title = scrapy.Field(
        output_processor=TakeFirst()
    )
    price = scrapy.Field(
        input_processor=MapCompose(clean_price),
        output_processor=TakeFirst()
    )
    availability = scrapy.Field(
        input_processor=MapCompose(clean_availability),
        output_processor=TakeFirst()
    )
    url = scrapy.Field(
        output_processor=TakeFirst()
    )
    scraped_at = scrapy.Field()  # We'll populate this in pipelines
```

---

## ⚙️ **Step 4: Data Pipeline (`pipelines.py`)**

Edit `bookscraper/pipelines.py`:

```python
from datetime import datetime
import sqlite3

class TimestampPipeline:
    """Add scrape timestamp to every item"""
    def process_item(self, item, spider):
        item['scraped_at'] = datetime.utcnow().isoformat()
        return item

class SQLitePipeline:
    """Store items in SQLite database"""
    def open_spider(self, spider):
        self.conn = sqlite3.connect('books.db')
        self.cursor = self.conn.cursor()
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS books (
                title TEXT,
                price REAL,
                availability TEXT,
                url TEXT UNIQUE,
                scraped_at TEXT
            )
        ''')
    
    def close_spider(self, spider):
        self.conn.commit()
        self.conn.close()
    
    def process_item(self, item, spider):
        # Upsert: update if URL exists, else insert
        self.cursor.execute('''
            INSERT OR REPLACE INTO books (title, price, availability, url, scraped_at)
            VALUES (?, ?, ?, ?, ?)
        ''', (
            item['title'],
            item['price'],
            item['availability'],
            item['url'],
            item['scraped_at']
        ))
        return item
```

Enable pipelines in `settings.py`:
```python
ITEM_PIPELINES = {
    'bookscraper.pipelines.TimestampPipeline': 300,
    'bookscraper.pipelines.SQLitePipeline': 400,
}
```

---

## 🚀 **Step 5: Run & Export Data**

```bash
# Run spider (auto-respects robots.txt + throttling)
scrapy crawl books

# Export to multiple formats simultaneously
scrapy crawl books -o books.csv -o books.json

# Export to custom SQLite DB (via pipeline)
scrapy crawl books
```

✅ **Scrapy automatically handles:**
- Request throttling (`DOWNLOAD_DELAY`)
- Retries on failure (`RETRY_ENABLED = True`)
- Duplicate URL filtering (`DUPEFILTER_CLASS`)
- robots.txt compliance (`ROBOTSTXT_OBEY = True`)
- Concurrent requests (configurable)

---

## 📊 **Step 6: Monitoring & Error Handling**

### Logging (built-in)
```python
# In your spider
self.logger.info(f"Scraped {len(books)} books from {response.url}")
self.logger.warning("Low stock detected!")
self.logger.error(f"Failed to parse price: {e}")
```

### Stats Collection (auto-generated after run)
```bash
scrapy crawl books --loglevel=INFO
```
Output includes:
```
2024-02-08 10:15:22 [scrapy.statscollectors] INFO: Dumping Scrapy stats:
{'downloader/request_count': 51,
 'downloader/response_status_count/200': 50,
 'downloader/response_status_count/404': 1,
 'item_scraped_count': 1000,
 'finish_reason': 'finished'}
```

### Alerting (simple email on failure)
```python
# In settings.py
import os
EXTENSIONS = {
    'scrapy.extensions.telnet.TelnetConsole': None,
}

# Custom extension (advanced): send email if item_scraped_count < threshold
```

---

## ⚖️ **Step 7: Legal Safeguards (Critical for Production)**

### 1. Document Permissions
Create `LEGAL.md` in your project:
```markdown
# Scraping Permissions

Site: books.toscrape.com
Permission: Explicitly permitted (sandbox site for learning)
robots.txt: Allows all user-agents
Terms of Service: N/A (educational sandbox)

Site: example-financial-data.com
Permission: ❌ NOT PERMITTED — requires API license
Action: DO NOT SCRAPE — use Alpha Vantage API instead
```

### 2. Rate Limiting Configuration (`settings.py`)
```python
# Be extra conservative for sensitive sites
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 5.0      # Start with 5s delay
AUTOTHROTTLE_MAX_DELAY = 60.0       # Back off up to 60s on errors
DOWNLOAD_TIMEOUT = 15               # Fail fast on slow responses
```

### 3. Data Retention Policy
```python
# In pipeline: auto-delete records > 30 days old
def close_spider(self, spider):
    self.cursor.execute('''
        DELETE FROM books WHERE scraped_at < datetime('now', '-30 days')
    ''')
    self.conn.commit()
```

---

## 📝 **Phase 5 Exercise**

**Task:** Build a complete Scrapy project for `books.toscrape.com` with:

1. ✅ Spider that paginates through all 50 pages
2. ✅ Item schema with price cleaning (`£10.99` → `10.99`)
3. ✅ Pipeline that adds `scraped_at` timestamp
4. ✅ Export to both CSV (`books.csv`) and SQLite (`books.db`)
5. ✅ Verify `robots.txt` compliance is enabled

**Starter commands:**
```bash
scrapy startproject bookscraper
cd bookscraper
scrapy genspider books books.toscrape.com
# Edit files as shown above
scrapy crawl books -o books.csv
```

**Verify success:**
```bash
# Check CSV
head books.csv

# Check SQLite
sqlite3 books.db "SELECT COUNT(*) FROM books;"
# Should return 1000 (50 pages × 20 books)
```

---

### ✅ **Phase 5 Complete When:**
- [ ] You've created a Scrapy project from scratch
- [ ] You've defined items with data validation/cleaning
- [ ] You've built a pipeline for timestamping/storage
- [ ] You've configured polite scraping (`DOWNLOAD_DELAY`, `ROBOTSTXT_OBEY`)
- [ ] You've exported data to multiple formats simultaneously
- [ ] You understand when Scrapy is appropriate vs. simpler tools

---

## 🎓 **What's Next? (Beyond This Course)**

| Topic | Why Learn It | Resource |
|-------|--------------|----------|
| **Proxy rotation** | Avoid IP bans on large jobs | ScraperAPI, Bright Data (use ethically!) |
| **Distributed scraping** | Scale across machines | Scrapy Cluster, Scrapy Cloud |
| **Legal frameworks** | Avoid lawsuits | *Web Scraping Legal Guide* (lawyers.com) |
| **Data enrichment** | Combine scraped + API data | Pandas merge operations |
| **ML on scraped data** | Sentiment analysis, pricing models | scikit-learn + scraped text |

---

### 🌟 **Final Wisdom**

> "The best scrapers are invisible: they leave no trace, respect boundaries, and add value without taking it away."  
> — Anonymous webmaster

✅ **You now know:**  
- How to scrape ethically and legally  
- Static vs. dynamic content techniques  
- Production-grade architecture with Scrapy  
- Critical safeguards (throttling, monitoring, documentation)

❌ **Never forget:**  
- Scraping is a **privilege**, not a right  
- When in doubt → **ask permission** or use APIs  
- Your algo-trading systems should **never rely on scraped financial data** without explicit licensing

---

**Want to dive deeper into a specific topic?** (e.g., "Show me proxy rotation ethically" or "How to scrape Twitter/X legally") — just ask! 🐍✨
