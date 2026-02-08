# 🎯 **Phase 1: Foundations**

Let's build a solid base before writing any scraping code. This phase is about **understanding the landscape**.

---

## 📋 **1. Prerequisites Checklist**

### ✅ Python Basics (You should be comfortable with):
```python
# Variables & data types
name = "scraper"
count = 10
is_active = True

# Lists & dictionaries
urls = ["https://site1.com", "https://site2.com"]
data = {"title": "Article", "price": 29.99}

# Loops
for url in urls:
    print(url)

# Functions
def fetch_page(url):
    return f"Content from {url}"

# Error handling (critical for scraping!)
try:
    result = risky_operation()
except Exception as e:
    print(f"Error: {e}")
```

> 💡 **Your trading background means you likely know this well.** If any concept feels shaky, say the word and we'll review.

---

### ✅ HTML/CSS Fundamentals

**Why this matters:** Websites are built with HTML. You need to "read" the structure to extract data.

**Key HTML Tags:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Page Title</title>
</head>
<body>
    <h1>Main Heading</h1>
    <p>This is a paragraph.</p>
    
    <div class="product">
        <h2 class="product-name">Widget</h2>
        <span id="price">$19.99</span>
        <a href="/buy-now">Buy</a>
    </div>
    
    <ul class="categories">
        <li>Electronics</li>
        <li>Books</li>
    </ul>
</body>
</html>
```

**Key Concepts:**
| Term | Meaning | Example |
|------|---------|---------|
| **Tag** | HTML element | `<h1>`, `<div>`, `<a>` |
| **Attribute** | Extra info about a tag | `class="product"`, `id="price"` |
| **Class** | Reusable identifier (can appear multiple times) | `class="product"` |
| **ID** | Unique identifier (appears once per page) | `id="main-header"` |
| **Nested elements** | Tags inside other tags | `<div><h2>Text</h2></div>` |

---

### ✅ CSS Selectors (How you "target" elements)

| Selector | Meaning | Example |
|----------|---------|---------|
| `tag` | All elements with that tag | `div` → all `<div>` elements |
| `.class` | Elements with that class | `.product` → all elements with `class="product"` |
| `#id` | Element with that ID | `#price` → element with `id="price"` |
| `tag.class` | Tag with specific class | `h2.title` → `<h2 class="title">` |
| `parent child` | Child inside parent | `div p` → `<p>` inside `<div>` |
| `parent > child` | Direct child only | `ul > li` → direct `<li>` children of `<ul>` |

---

### ✅ HTTP Basics

**HTTP Request-Response Cycle:**
```
Your Script → (GET request) → Server → (HTML response) → Your Script
```

**Common HTTP Methods:**
| Method | Purpose | Scraping Relevance |
|--------|---------|-------------------|
| `GET` | Fetch a page | ✅ Most common |
| `POST` | Submit data (forms) | ⚠️ Login walls, searches |
| `PUT`/`DELETE` | Update/delete | ❌ Rarely used in scraping |

**HTTP Status Codes:**
| Code | Meaning | Action |
|------|---------|--------|
| `200` | Success | ✅ Proceed |
| `403` | Forbidden | ❌ Check headers/permissions |
| `404` | Not Found | ⚠️ URL changed |
| `429` | Too Many Requests | ⚠️ Slow down! |
| `500` | Server Error | ⚠️ Retry later |

**Headers (Important for scraping):**
```python
headers = {
    "User-Agent": "Mozilla/5.0...",  # Identifies your "browser"
    "Accept": "text/html",           # What you accept back
    "Referer": "https://google.com"  # Where you came from
}
```

---

## 🔧 **2. Tools Setup**

### Install Core Libraries:
```bash
# Core scraping stack
pip install requests beautifulsoup4 lxml

# Data handling (you likely have this)
pip install pandas

# Verification
python -c "import requests, bs4, pandas; print('✅ All installed!')"
```

### Optional but Recommended:
```bash
# For virtual environments (keeps projects clean)
pip install virtualenv

# Create isolated environment
virtualenv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

---

## 🧪 **3. Quick Self-Test**

**Exercise 1: Inspect a Website**
1. Go to any news site (e.g., `https://news.ycombinator.com`)
2. Right-click → "Inspect" (or press F12)
3. Find:
   - The `<title>` tag content
   - A class name used on article titles
   - An `<a>` tag with an `href` attribute

**Exercise 2: Python Check**
```python
# Run this - no errors = you're ready
import requests
from bs4 import BeautifulSoup

print("✅ Python environment ready!")
```

---

## 📝 **Phase 1 Summary**

| Concept | Status |
|---------|--------|
| Python basics | ✅ Assumed known |
| HTML structure | 📖 Review if needed |
| CSS selectors | 📖 Critical for targeting |
| HTTP fundamentals | 📖 Understand requests/responses |
| Tools installed | 🔧 Verify with pip |

---

### ✅ **When you're ready:**
Just say **"Next"** or **"Phase 2"**, and we'll dive into **static site scraping with `requests` + `BeautifulSoup`** — where the real coding begins! 🐍

Any questions about Phase 1 before we proceed?