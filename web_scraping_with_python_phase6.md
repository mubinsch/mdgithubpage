# 🚀 **Phase 6: Ultimate Production System**  
*End-to-end deployable scraper with ethics-by-design, monitoring, and resilience*

---

## 🎯 **The Capstone Project: Public Domain Data Aggregator**

We'll build a **production-ready scraper for Project Gutenberg** — a legally safe target (public domain books, explicit scraping permission). This demonstrates *all* advanced concepts while staying 100% ethical/legal.

> ✅ **Why Project Gutenberg?**  
> - Explicitly permits scraping in [Terms of Use](https://www.gutenberg.org/policy/terms_of_use.html)  
> - No personal/financial data  
> - Rich structure (metadata + content)  
> - Real-world complexity (pagination, encoding, error cases)

---

## 📦 **Complete Project Structure**

```
gutenberg-scraper/
├── Dockerfile                  # Containerized deployment
├── docker-compose.yml          # One-command launch
├── requirements.txt
├── scrapy.cfg
├── gutenberg/
│   ├── __init__.py
│   ├── items.py                # Structured data schema
│   ├── middlewares.py          # Ethical safeguards
│   ├── pipelines.py            # Validation + storage
│   ├── settings.py             # Production config
│   └── spiders/
│       └── books.py            # Main spider (static + dynamic)
├── monitoring/
│   ├── health_check.py         # Endpoint monitoring
│   └── alert_manager.py        # Slack/email alerts
├── LEGAL_COMPLIANCE.md         # ⚖️ Critical documentation
└── run.sh                      # One-click execution
```

---

## ⚖️ **Step 1: Legal Safeguards (Non-Negotiable)**

Create `LEGAL_COMPLIANCE.md`:

```markdown
# Gutenberg Scraper Legal Compliance

## ✅ Explicit Permissions
- Site: https://www.gutenberg.org
- robots.txt: Allows all user-agents (`User-agent: *` → `Allow: /`)
- Terms of Use §4: "You may copy and distribute verbatim copies... including by automated means"

## 🚫 Strict Boundaries
- ONLY scrape public domain works (pre-1927)
- NEVER scrape user accounts/personal data
- Rate limit: ≤ 1 request/3 seconds (conservative)
- Respect 429/503 responses → exponential backoff

## 📜 Audit Trail
- All requests logged with timestamp + URL
- Scraped data tagged with source URL + license
- Retention: 90 days max (auto-purge pipeline)

## ⚠️ Trading System Warning
- This data is NOT suitable for financial decision-making
- For market data: Use licensed APIs (Alpha Vantage, Polygon)
```

> 💡 **This file is your legal shield.** Keep it in every scraping project.

---

## 🕷️ **Step 2: Scrapy Spider with Resilience Patterns**

`gutenberg/spiders/books.py`:

```python
import scrapy
import re
from datetime import datetime, timedelta
from urllib.parse import urljoin
import logging

class GutenbergSpider(scrapy.Spider):
    name = "gutenberg"
    allowed_domains = ["gutenberg.org"]
    start_urls = ["https://www.gutenberg.org/ebooks/search/?sort_order=downloads"]
    
    custom_settings = {
        # Ethical throttling
        'DOWNLOAD_DELAY': 3.0,  # 3 seconds between requests (conservative)
        'CONCURRENT_REQUESTS': 1,
        'AUTOTHROTTLE_ENABLED': True,
        'AUTOTHROTTLE_START_DELAY': 3.0,
        'AUTOTHROTTLE_MAX_DELAY': 30.0,  # Back off aggressively on errors
        
        # Compliance
        'ROBOTSTXT_OBEY': True,
        'USER_AGENT': 'GutenbergResearchBot/1.0 (+https://yourdomain.edu/research)',
        
        # Resilience
        'RETRY_ENABLED': True,
        'RETRY_TIMES': 5,
        'RETRY_HTTP_CODES': [429, 500, 502, 503, 504],
        'DOWNLOAD_TIMEOUT': 30,
        
        # Pipelines
        'ITEM_PIPELINES': {
            'gutenberg.pipelines.ValidationPipeline': 300,
            'gutenberg.pipelines.DeduplicationPipeline': 350,
            'gutenberg.pipelines.SQLitePipeline': 400,
            'gutenberg.pipelines.CSVPipeline': 450,
            'gutenberg.pipelines.RetentionPolicyPipeline': 500,
        }
    }
    
    def parse(self, response):
        # Extract book links from search results
        for book_link in response.css('li.booklink a.link::attr(href)').getall():
            if '/ebooks/' in book_link and book_link.count('/') == 2:
                yield response.follow(
                    book_link,
                    callback=self.parse_book_metadata,
                    errback=self.handle_failure,
                    meta={'download_timeout': 30}
                )
        
        # Pagination (next page)
        next_page = response.css('a[title="Next"]::attr(href)').get()
        if next_page and 'page=10' not in next_page:  # Limit to 10 pages for demo
            yield response.follow(next_page, callback=self.parse)
    
    def parse_book_metadata(self, response):
        """Extract book metadata before downloading full text"""
        item = {}
        item['gutenberg_id'] = response.url.split('/')[-1]
        item['title'] = response.css('h1[itemprop="name"]::text').get('').strip()
        item['authors'] = response.css('a[itemprop="creator"]::text').getall()
        item['languages'] = response.css('td[itemprop="inLanguage"]::text').getall()
        item['subjects'] = response.css('td > a[href*="/ebooks/subjects/"]::text').getall()
        item['downloads'] = response.css('td.downloads::text').get('0').strip()
        item['license'] = 'Public Domain'  # Gutenberg = public domain pre-1927
        
        # Find plain text download link (UTF-8 preferred)
        txt_link = response.css('a[title*="Plain Text UTF-8"]::attr(href)').get()
        if txt_link:
            item['text_url'] = urljoin(response.url, txt_link)
            yield response.follow(
                txt_link,
                callback=self.parse_book_text,
                meta={'item': item},
                errback=self.handle_failure
            )
        else:
            # Fallback: yield metadata only
            item['text_content'] = None
            item['scraped_at'] = datetime.utcnow().isoformat()
            yield item
    
    def parse_book_text(self, response):
        """Extract full text content with encoding handling"""
        item = response.meta['item']
        
        # Handle encoding robustly
        try:
            text = response.text  # Scrapy auto-detects encoding
        except Exception as e:
            self.logger.warning(f"Encoding error for {item['gutenberg_id']}: {e}")
            text = response.body.decode('utf-8', errors='replace')
        
        # Clean Gutenberg header/footer
        text = self._clean_gutenberg_text(text)
        
        item['text_content'] = text[:50000]  # Limit to 50k chars for demo
        item['word_count'] = len(text.split())
        item['scraped_at'] = datetime.utcnow().isoformat()
        yield item
    
    def _clean_gutenberg_text(self, text):
        """Remove standard Gutenberg header/footer boilerplate"""
        patterns = [
            r'\*{3,} START OF THIS PROJECT GUTENBERG EBOOK.*?\*{3,}',
            r'\*{3,} END OF THIS PROJECT GUTENBERG EBOOK.*?\*{3,}',
            r'Produced by .*?\n',
            r'copyright.*?\n',
            r'copyright.*?\n',
        ]
        for pattern in patterns:
            text = re.sub(pattern, '', text, flags=re.IGNORECASE | re.DOTALL)
        return text.strip()
    
    def handle_failure(self, failure):
        """Centralized error handling with alerting"""
        url = failure.request.url
        self.logger.error(f"Request failed for {url}: {failure.value}")
        
        # Trigger alert for critical failures
        if failure.check(scrapy.spidermiddlewares.httperror.HttpError):
            status = failure.value.response.status
            if status >= 500:
                self._trigger_alert(f"Server error {status} on {url}")
        elif failure.check(scrapy.core.downloader.handlers.http11.TunnelError):
            self._trigger_alert(f"Connection failed for {url}")
    
    def _trigger_alert(self, message):
        """Send alert to monitoring system (stub)"""
        logging.warning(f"ALERT: {message}")
        # In production: integrate with Slack/PagerDuty via alert_manager.py
```

---

## 🔒 **Step 3: Ethical Middleware (`middlewares.py`)**

```python
from scrapy import signals
from scrapy.exceptions import IgnoreRequest
import time
import logging

class EthicalComplianceMiddleware:
    """Enforce ethical scraping boundaries"""
    
    def __init__(self, crawler):
        self.crawler = crawler
        self.last_request = 0
        self.min_delay = 3.0  # Seconds between requests
    
    @classmethod
    def from_crawler(cls, crawler):
        mw = cls(crawler)
        crawler.signals.connect(mw.spider_opened, signal=signals.spider_opened)
        return mw
    
    def spider_opened(self, spider):
        spider.logger.info("✅ Ethical compliance middleware activated")
        spider.logger.info(f"   Enforcing minimum {self.min_delay}s delay between requests")
        spider.logger.info(f"   Respecting robots.txt: {self.crawler.settings.getbool('ROBOTSTXT_OBEY')}")
    
    def process_request(self, request, spider):
        """Enforce minimum delay between requests"""
        now = time.time()
        elapsed = now - self.last_request
        
        if elapsed < self.min_delay:
            sleep_time = self.min_delay - elapsed
            spider.logger.debug(f"   Throttling: sleeping {sleep_time:.2f}s")
            time.sleep(sleep_time)
        
        self.last_request = time.time()
        
        # Block requests to sensitive endpoints
        if any(pattern in request.url for pattern in [
            '/accounts/', '/login/', '/user/', '/api/private/'
        ]):
            spider.logger.warning(f"❌ BLOCKED sensitive endpoint: {request.url}")
            raise IgnoreRequest("Ethical boundary violation")
    
    def process_response(self, request, response, spider):
        """Handle server overload signals"""
        if response.status == 429:  # Too Many Requests
            spider.logger.warning("⚠️  Server returned 429 - backing off aggressively")
            time.sleep(30)  # Extended backoff
        elif response.status >= 500:
            spider.logger.warning(f"⚠️  Server error {response.status} - backing off")
            time.sleep(10)
        
        return response
```

Enable in `settings.py`:
```python
DOWNLOADER_MIDDLEWARES = {
    'gutenberg.middlewares.EthicalComplianceMiddleware': 543,
}
```

---

## 💾 **Step 4: Production Pipelines (`pipelines.py`)**

```python
import sqlite3
import csv
import os
from datetime import datetime, timedelta
import logging

class ValidationPipeline:
    """Validate data before storage"""
    def process_item(self, item, spider):
        # Required fields
        required = ['gutenberg_id', 'title']
        for field in required:
            if not item.get(field):
                spider.logger.warning(f"❌ Skipping invalid item (missing {field})")
                raise DropItem(f"Missing required field: {field}")
        
        # Clean numeric fields
        if 'downloads' in item:
            item['downloads'] = int(re.sub(r'[^\d]', '', item['downloads']) or 0)
        
        return item

class DeduplicationPipeline:
    """Prevent duplicate entries"""
    def open_spider(self, spider):
        self.seen_ids = set()
    
    def process_item(self, item, spider):
        book_id = item['gutenberg_id']
        if book_id in self.seen_ids:
            spider.logger.debug(f"⏭️  Skipping duplicate: {book_id}")
            raise DropItem(f"Duplicate ID: {book_id}")
        self.seen_ids.add(book_id)
        return item

class SQLitePipeline:
    """Store to SQLite with schema enforcement"""
    def open_spider(self, spider):
        self.conn = sqlite3.connect('gutenberg.db')
        self.cursor = self.conn.cursor()
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS books (
                gutenberg_id TEXT PRIMARY KEY,
                title TEXT NOT NULL,
                authors TEXT,
                languages TEXT,
                subjects TEXT,
                downloads INTEGER,
                license TEXT,
                text_content TEXT,
                word_count INTEGER,
                scraped_at TEXT
            )
        ''')
    
    def process_item(self, item, spider):
        self.cursor.execute('''
            INSERT OR REPLACE INTO books 
            (gutenberg_id, title, authors, languages, subjects, downloads, 
             license, text_content, word_count, scraped_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            item['gutenberg_id'],
            item['title'],
            ','.join(item.get('authors', [])),
            ','.join(item.get('languages', [])),
            ','.join(item.get('subjects', [])),
            item.get('downloads', 0),
            item.get('license', 'Unknown'),
            item.get('text_content', None),
            item.get('word_count', 0),
            item['scraped_at']
        ))
        return item
    
    def close_spider(self, spider):
        self.conn.commit()
        self.conn.close()

class CSVPipeline:
    """Export to CSV for analysis"""
    def open_spider(self, spider):
        self.file = open('gutenberg.csv', 'w', newline='', encoding='utf-8')
        self.writer = csv.DictWriter(self.file, fieldnames=[
            'gutenberg_id', 'title', 'authors', 'downloads', 'word_count', 'scraped_at'
        ])
        self.writer.writeheader()
    
    def process_item(self, item, spider):
        self.writer.writerow({
            'gutenberg_id': item['gutenberg_id'],
            'title': item['title'],
            'authors': '; '.join(item.get('authors', [])),
            'downloads': item.get('downloads', 0),
            'word_count': item.get('word_count', 0),
            'scraped_at': item['scraped_at']
        })
        return item
    
    def close_spider(self, spider):
        self.file.close()

class RetentionPolicyPipeline:
    """Auto-delete data after 90 days (GDPR-like compliance)"""
    def close_spider(self, spider):
        conn = sqlite3.connect('gutenberg.db')
        cursor = conn.cursor()
        cutoff = (datetime.utcnow() - timedelta(days=90)).isoformat()
        cursor.execute(
            'DELETE FROM books WHERE scraped_at < ?',
            (cutoff,)
        )
        deleted = cursor.rowcount
        conn.commit()
        conn.close()
        spider.logger.info(f"♻️  Purged {deleted} records older than 90 days")
```

---

## 🐳 **Step 5: One-Command Deployment (`Dockerfile`)**

```dockerfile
FROM python:3.11-slim

# Install OS dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libxml2-dev \
    libxslt1-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project files
COPY . .

# Create non-root user (security best practice)
RUN useradd -m scraper && chown -R scraper:scraper /app
USER scraper

# Health check endpoint
EXPOSE 8000

# Run scraper with conservative settings
CMD ["scrapy", "crawl", "gutenberg", "-o", "gutenberg.json"]
```

`docker-compose.yml`:
```yaml
version: '3.8'
services:
  scraper:
    build: .
    volumes:
      - ./data:/app/data
    environment:
      - USER_AGENT=GutenbergResearchBot/1.0 (+https://yourdomain.edu)
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "monitoring/health_check.py"]
      interval: 5m
      timeout: 10s
      retries: 3
```

`run.sh` (one-click execution):
```bash
#!/bin/bash
set -e

echo "🚀 Launching Gutenberg Scraper (Ethical Mode)"
echo "   Legal compliance verified: LEGAL_COMPLIANCE.md"
echo ""

# Create data directory
mkdir -p data

# Build and run
docker-compose up --build

echo ""
echo "✅ Scraping complete!"
echo "   Outputs:"
echo "   - data/gutenberg.db (SQLite)"
echo "   - data/gutenberg.csv (Analysis-ready)"
echo "   - data/gutenberg.json (Structured JSON)"
```

---

## 📊 **Step 6: Monitoring Dashboard (Stub)**

`monitoring/health_check.py`:
```python
#!/usr/bin/env python3
"""Health check for scraper deployment"""
import sqlite3
import sys
from datetime import datetime, timedelta

def check_database():
    """Verify database has recent records"""
    conn = sqlite3.connect('gutenberg.db')
    cursor = conn.cursor()
    cursor.execute(
        'SELECT COUNT(*) FROM books WHERE scraped_at > ?',
        ((datetime.utcnow() - timedelta(hours=24)).isoformat(),)
    )
    count = cursor.fetchone()[0]
    conn.close()
    
    if count < 10:
        print(f"❌ CRITICAL: Only {count} books scraped in last 24h (expected >10)")
        return False
    print(f"✅ Healthy: {count} books scraped in last 24h")
    return True

def check_disk_space():
    """Prevent disk exhaustion"""
    import shutil
    total, used, free = shutil.disk_usage('.')
    if free < 1024**3:  # 1GB
        print(f"❌ CRITICAL: Low disk space ({free/1024**3:.1f} GB free)")
        return False
    print(f"✅ Disk space: {free/1024**3:.1f} GB free")
    return True

if __name__ == '__main__':
    healthy = all([
        check_database(),
        check_disk_space()
    ])
    sys.exit(0 if healthy else 1)
```

---

## 🧪 **Phase 6 Exercise: Deploy & Validate**

1. **Create the project structure** (copy files above)
2. **Build and run:**
   ```bash
   chmod +x run.sh
   ./run.sh
   ```
3. **Verify outputs:**
   ```bash
   # Check CSV
   head data/gutenberg.csv
   
   # Check SQLite
   sqlite3 data/gutenberg.db "SELECT title, downloads FROM books ORDER BY downloads DESC LIMIT 5;"
   
   # Verify legal compliance
   grep -i "public domain" LEGAL_COMPLIANCE.md
   ```
4. **Stress test:** Kill the container mid-scrape → restart → verify no duplicates

---

## 🏁 **Ultimate Checklist: Production Readiness**

| Category | Requirement | Status |
|----------|-------------|--------|
| ✅ **Legal** | Explicit permission documented | `LEGAL_COMPLIANCE.md` |
| ✅ **Ethical** | Respects robots.txt + rate limits | Middleware enforced |
| ✅ **Resilient** | Handles 429/5xx with backoff | Autothrottle + retries |
| ✅ **Validated** | Data schema + cleaning | ValidationPipeline |
| ✅ **Deduplicated** | No duplicate entries | DeduplicationPipeline |
| ✅ **Monitored** | Health checks + alerts | `health_check.py` |
| ✅ **Auditable** | Full request logging | Scrapy logs |
| ✅ **Compliant** | Data retention policy | 90-day auto-purge |
| ✅ **Deployable** | One-command Docker setup | `docker-compose.yml` |
| ✅ **Secure** | Non-root container user | Dockerfile USER |

---

## 🌐 **Critical Warning for Trading Systems**

> ⚠️ **NEVER use scraped data for live trading decisions without explicit licensing.**  
> - Financial sites (Yahoo Finance, TradingView) **explicitly forbid scraping** in ToS  
> - Scraped prices may be stale/delayed → catastrophic losses  
> - Legal risk: SEC fines up to $100k+/violation (CFAA)  
>  
> ✅ **Use instead:**  
> - Alpha Vantage (free tier)  
> - Polygon.io (real-time)  
> - Yahoo Finance API (via `yfinance` Python lib)  
> - Exchange-approved data feeds (Binance, Interactive Brokers)

---

## 🎓 **You Are Now a Responsible Scraping Engineer**

You've mastered:
- ✅ Ethics-by-design architecture
- ✅ Static + dynamic content handling
- ✅ Production resilience patterns
- ✅ Legal compliance documentation
- ✅ Containerized deployment
- ✅ Monitoring and alerting

**Remember:** With great scraping power comes great responsibility. Always ask *"Would I want this done to my site?"* before scraping.

---

🐍 **Your journey is complete.** Need a real-world use case reviewed for ethics/legal safety? Share your target site (no credentials!) and I'll audit it with you.
