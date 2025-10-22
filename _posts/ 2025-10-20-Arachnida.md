---
layout: post
title: "A Deep Dive into Web Scraping and Metadata Analysis"
date: 2025-08-01
author: "ACH4Q"
tags: ["Cybersecurity", "python"]
draft: false
---

# Building Arachnida: A Deep Dive into Web Scraping and Metadata Analysis

In the world of cybersecurity, our mantra is that data is everywhere. Every action we take online, every photo we share, and every website we build leaves a digital footprint. Understanding how to gather and interpret this data is a foundational skill, whether you're performing Open-Source Intelligence (OSINT) reconnaissance or securing your own digital assets.

Today, I want to walk you through a project I built called **Arachnida**. It’s a hands-on introduction to two powerful concepts: web scraping and metadata analysis. We'll build two command-line tools from scratch in Python:

1.  **`spider.py`**: A recursive web scraper that crawls a website to download all of its images.
2.  **`scorpion.py`**: A metadata analyzer that extracts hidden data from those downloaded image files.

This project is a perfect way to learn because it forces us to build the core logic ourselves, without relying on powerhouse libraries like `Scrapy`. Let's dive in.

## Part 1: The `Spider` - Weaving Through the Web

The first challenge is to retrieve data from a website automatically. Our `spider` tool will be given a starting URL and its mission is to find and download all the images, even following links to other pages on the same site.

### The Fundamental Tools

To build our spider, we only need a few key libraries:

-   **`requests`**: This is our HTTP client. It's how our script will act like a web browser, sending a `GET` request to a server and receiving the raw HTML content of a page.
-   **`BeautifulSoup`**: A website's content is structured in HTML. Simply getting the raw text isn't enough; we need to parse it. `BeautifulSoup` transforms the messy blob of HTML into a structured object, allowing us to easily search for specific elements, like image tags (`<img>`) or links (`<a>`).
-   **`urllib.parse`**: A built-in Python library that helps us work with URLs, for example by combining a base URL with a relative path.

### The Technical Blueprint

Our spider's logic can be broken down into a recursive function. Recursion here is like a librarian who, after reading a book, follows every reference in its bibliography to find more books, and then does the same for each of those.

Here’s the high-level workflow for our `scrape_page` function:

**1. The Base Case (Stopping Condition):** The most important part of any recursive function is knowing when to stop. Our spider stops if:
    - The recursion depth limit (`-l` option) is exceeded.
    - It encounters a URL it has already visited (to avoid getting stuck in loops).

```python
def scrape_page(url, current_depth, args, visited_urls):
    if current_depth > args.level or url in visited_urls:
        return
    visited_urls.add(url)
```

**2. Fetch and Parse:** We use `requests` to get the page and `BeautifulSoup` to parse it. Robust error handling with a `try...except` block is crucial here to manage network failures or dead links.

```python
import requests
from bs4 import BeautifulSoup

try:
    response = requests.get(url, timeout=10)
    response.raise_for_status() # Checks for errors like 404 Not Found
    soup = BeautifulSoup(response.text, 'html.parser')
except requests.RequestException as e:
    print(f"Could not process page {url}: {e}")
    return
```

**3. Find and Download Images:** We search the parsed `soup` object for all `<img>` tags, extract their source (`src`) attribute, and download the content. A key challenge is handling both **absolute** (`http://.../img.png`) and **relative** (`/images/img.png`) URLs. Python's `urljoin` is perfect for this.

```python
from urllib.parse import urljoin

img_tags = soup.find_all('img')
for img_tag in img_tags:
    img_url = img_tag.get('src')
    if not img_url:
        continue
    # This resolves relative paths into full URLs
    full_img_url = urljoin(url, img_url)
    # ... (code to download and save the image) ...
```

**4. Find Links and Recurse:** If the `-r` (recursive) option is enabled, we find all `<a>` (link) tags. For each link that points to the same website, we call `scrape_page` again, increasing the depth by one.

```python
from urllib.parse import urljoin, urlparse

if args.recursive:
    link_tags = soup.find_all('a')
    for link_tag in link_tags:
        link_url = link_tag.get('href')
        full_link_url = urljoin(url, link_url)
        # We only follow links on the same domain
        if urlparse(full_link_url).netloc == urlparse(url).netloc:
            scrape_page(full_link_url, current_depth + 1, args, visited_urls)
```

With this logic, `spider.py` becomes a capable tool for mapping and extracting image content from a target domain.

## Part 2: The `Scorpion` - The Sting is in the Data

Now that we have a folder full of images, what can they tell us? This is where `scorpion.py` comes in. Its job is to look *inside* the files for metadata.

### The Fundamental Concept: EXIF Data

Metadata is "data about data." For an image file, this could be its size or creation date. But the real treasure is **EXIF (Exchangeable Image File Format) data**. When you take a photo with a digital camera or a smartphone, a huge amount of information is embedded directly into the image file:
- Camera make and model
- Date and time the photo was taken
- Camera settings (ISO, aperture, shutter speed)
- And sometimes, the holy grail of OSINT: **GPS coordinates**.

### The Technical Blueprint

To extract this data, we use the powerful **`Pillow`** library, a fork of the Python Imaging Library (PIL).

**1. Reading Basic File Info:** We can get simple metadata like file size and modification time using Python's built-in `os` module.

```python
import os
from datetime import datetime

file_stat = os.stat(filepath)
print(f"File Size: {file_stat.st_size / 1024:.2f} KB")
print(f"Last Modified: {datetime.fromtimestamp(file_stat.st_mtime)}")
```

**2. Extracting EXIF Data:** `Pillow` makes this surprisingly easy. We open the image and call the `_getexif()` method.

```python
from PIL import Image

image = Image.open(filepath)
exif_data = image._getexif()
```

**3. Making Sense of the Data:** The `exif_data` object is a dictionary, but its keys are numerical tags, not human-readable strings. For example, the camera model is stored under tag ID `272`. `Pillow` provides a helper, `PIL.ExifTags.TAGS`, to translate these IDs into meaningful names. We can loop through the dictionary and build a clean report.

```python
from PIL.ExifTags import TAGS

if exif_data:
    for tag_id, value in exif_data.items():
        tag_name = TAGS.get(tag_id, tag_id)
        print(f"  {tag_name}: {value}")
```

## The Cybersecurity Connection: Why This Matters

The Arachnida project is more than a coding exercise; it's a practical demonstration of OSINT and operational security (OpSec).

-   **Offensive Perspective (OSINT):** An attacker could use a tool like `spider` to quickly map out a target website's assets. By running `scorpion` on downloaded images (e.g., photos of employees or offices), they could potentially uncover what kind of cameras or phones are used, when a picture was taken, or even the exact location of an office or event if GPS data was not stripped.

-   **Defensive Perspective (OpSec):** This project teaches us a vital lesson: **scrub your metadata**. Before uploading images to a public website, especially photos of sensitive locations or personnel, all EXIF data should be removed. Many platforms do this automatically, but you should never assume.

## Conclusion

Building `spider.py` taught us how to programmatically navigate the web, handle network requests, and parse complex HTML. Building `scorpion.py` showed us that files often contain much more information than what's visible on the surface.

Together, they form the Arachnida project—a small but potent reminder that the web is a vast network of interconnected data, and with the right tools, we can begin to unravel its threads.

You can find the complete source code for this project on my [GitHub repository](https://github.com/ACH4Q/Arachnida). Feel free to clone the repository, experiment with the code, and see what you can uncover.

