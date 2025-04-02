# Using cloudscraper in Python

[![Bright Data Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/)

This guide explains how to use the `cloudscraper` Python library to bypass Cloudflare’s protection and handle errors:

- [Install Prerequisites](#install-prerequisites)
- [Write Initial Scraping Code](#write-initial-scraping-code)
- [Incorporate cloudscraper](#incorporate-cloudscraper)
- [Use Additional cloudscraper Features](#use-additional-cloudscraper-features)
  - [Proxies](#proxies)
  - [Change the User Agent and JavaScript Interpreter](#change-the-user-agent-and-javascript-interpreter)
  - [Handling CAPTCHAs](#handling-captchas)
- [Common cloudscraper Errors](#common-cloudscraper-errors)
  - [`module not found`](#module-not-found)
  - [`cloudscraper can’t bypass the latest Cloudflare version`](#cloudscraper-cant-bypass-the-latest-cloudflare-version)
- [cloudscraper Alternatives](#cloudscraper-alternatives)
- [Conclusion](#conclusion)

## Install Prerequisites

Ensure you have Python 3 installed, then install the necessary packages:

```bash
pip install tqdm==4.66.5 requests==2.32.3 beautifulsoup4==4.12.3
```

## Write Initial Scraping Code

This guide assumes you're scraping metadata from news articles published on a specific date on the [ChannelsTV website](https://www.channelstv.com/). Below is an initial Python script:

```python
import requests
from bs4 import BeautifulSoup
from datetime import datetime
from tqdm.auto import tqdm

def extract_article_data(article_source, headers):
    response = requests.get(article_source, headers=headers)
    if response.status_code != 200:
        return None

    soup = BeautifulSoup(response.content, 'html.parser')

    title = soup.find(class_="post-title display-3").text.strip()

    date = soup.find(class_="post-meta_time").text.strip()
    date_object = datetime.strptime(date, 'Updated %B %d, %Y').date()

    categories = [category.text.strip() for category in soup.find('nav', {"aria-label": "breadcrumb"}).find_all('li')]

    tags = [tag.text.strip() for tag in soup.find("div", class_="tags").find_all("a")]

    article_data = {
        'date': date_object,
        'title': title,
        'link': article_source,
        'tags': tags,
        'categories': categories
    }

    return article_data

def process_page(articles, headers):
    page_data = []
    for article in tqdm(articles):
        url = article.find('a', href=True).get('href')
        if "https://" not in url:
            continue
        article_data = extract_article_data(url, headers)
        if article_data:
            page_data.append(article_data)
    return page_data

def scrape_articles_per_day(base_url, headers):
    day_data = []
    page = 1

    while True:
        page_url = f"{base_url}/page/{page}"
        response = requests.get(page_url, headers=headers)

        if not response or response.status_code != 200:
            break

        soup = BeautifulSoup(response.content, 'html.parser')
        articles = soup.find_all('article')

        if not articles:
            break
        page_data = process_page(articles, headers)
        day_data.extend(page_data)

        page += 1

    return day_data

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36',
}

URL = "https://www.channelstv.com/2024/08/01/"

scraped_articles = scrape_articles_per_day(URL, headers)
print(f"{len(scraped_articles)} articles were scraped.")
print("Samples:")
print(scraped_articles[:2])
```

This script defines three key functions for scraping. The `extract_article_data` function retrieves content from an article’s webpage, extracting metadata such as title, publication date, tags, and categories into a dictionary.

Next, the `process_page` function iterates through all articles on a given page, extracting metadata using `extract_article_data` and compiling the results into a list.

Finally, the `scrape_articles_per_day` function systematically navigates through paginated results, incrementing the page number in a `while` loop until no more pages are found.

To execute the scraper, the script specifies a target URL with a filtering date of August 1, 2024. A user-agent header is set, and the `scrape_articles_per_day` function is called with the provided URL and headers. The total number of articles scraped is printed, along with a preview of the first two results.

However, the script does not work as expected because the ChannelsTV website employs Cloudflare protection, blocking direct HTTP requests made by `extract_article_data` and `scrape_articles_per_day`.

When running the script, the output typically appears as follows:

```
0 articles were scraped.
Samples:
[]
```

## Incorporate cloudscraper

Install `cloudscraper` to bypass Cloudflare:

```bash
pip install cloudscraper==1.2.71
```

Modify the script to use `cloudscraper`:

```python
import cloudscraper

def fetch_html_content(url, headers):
    try:
        scraper = cloudscraper.create_scraper()
        response = scraper.get(url, headers=headers)

        if response.status_code == 200:
            return response
        else:
            print(f"Failed to fetch URL: {url}. Status code: {response.status_code}")
            return None
    except Exception as e:
        print(f"An error occurred while fetching URL: {url}. Error: {str(e)}")
        return None
```

This function, `fetch_html_content`, takes a URL and request headers as input. It attempts to retrieve the webpage using `cloudscraper.create_scraper()`. If the request is successful (status code 200), the response is returned; otherwise, an error message is printed, and `None` is returned. If an exception occurs, the error is caught and displayed before returning `None`.

Following this update, all `requests.get` calls are replaced with `fetch_html_content`, ensuring compatibility with Cloudflare-protected websites. The first modification occurs in the `extract_article_data` function, as demonstrated above.

Now, replace `requests.get` calls with `fetch_html_content` in your scraping functions.

```python
def extract_article_data(article_source, headers):
    response = fetch_html_content(article_source, headers)
```

After that, replace the `requests.get` call in your `scrape_articles_per_day` function like this:

```python
def scrape_articles_per_day(base_url, headers):
    day_data = []
    page = 1

    while True:
        page_url = f"{base_url}/page/{page}" 
        response = fetch_html_content(page_url, headers)
```

By defining this function, the cloudscraper library can help you evade Cloudflare’s restrictions.

When you run the code, your output looks like this:

```
Failed to fetch URL: https://www.channelstv.com/2024/08/01//page/5. Status code: 404
55 articles were scraped.
Samples:
[{'date': datetime.date(2024, 8, 1),
  'title': 'Resilience, Tear Gas, Looting, Curfew As #EndBadGovernance Protests Hold',
  'link': 'https://www.channelstv.com/2024/08/01/tear-gas-resilience-looting-curfew-as-endbadgovernance-protests-hold/',
  'tags': ['Eagle Square', 'Hunger', 'Looting', 'MKO Abiola Park', 'violence'],
  'categories': ['Headlines']},
 {'date': datetime.date(2024, 8, 1),
  'title': 'Mother Of Russian Artist Freed In Prisoner Swap Waiting To 'Hug' Her',
  'link': 'https://www.channelstv.com/2024/08/01/mother-of-russian-artist-freed-in-prisoner-swap-waiting-to-hug-her/',
  'tags': ['Prisoner Swap', 'Russia'],
  'categories': ['World News']}]
```

## Use Additional cloudscraper Features

### Proxies

With cloudscraper, you can define proxies and pass them to your already created `cloudscraper` object like this:

```python
scraper = cloudscraper.create_scraper()
proxy = {
    'http': 'http://your-proxy-ip:port',
    'https': 'https://your-proxy-ip:port'
}
response = scraper.get(URL, proxies=proxy)
```

Here, you start by defining a scraper object with default values. Then, you define a proxy dictionary with `http` and `https` proxies. After that, you pass the proxy dictionary object to the `scraper.get` method as you would with a regular `request.get` method.

### Change the User Agent and JavaScript Interpreter

The cloudscraper library can autogenerate user agents and lets you specify the JavaScript interpreter and engine you use with your scraper. Here is some example code:

```python
scraper = cloudscraper.create_scraper(
    interpreter="nodejs",
    browser={
        "browser": "chrome",
        "platform": "ios",
        "desktop": False,
    }
)
```

The above script sets the interpreter as `"nodejs"` and passes a dictionary to the browser parameter. The browser is set to Chrome and the platform is set to `"ios"`. The desktop parameter is set to `False` which suggests that the browser runs on mobile.

### Handling CAPTCHAs

The cloudscraper library supports third-party CAPTCHA solvers to bypass reCAPTCHA, hCaptcha, and more. The following snippet shows you how to modify your scraper to handle CAPTCHA:

```python
scraper = cloudscraper.create_scraper(
  captcha={
    'provider': 'capsolver',
    'api_key': 'your_capsolver_api_key'
  }
)
```

The code uses Capsolver for CAPTCHA provider and the Capsolver API key. Both values are stored in a dictionary and passed to the CAPTCHA parameter in the `cloudscraper.create_scraper` method.

## Common cloudscraper Errors

### `module not found`

Ensure `cloudscraper` is installed:

```sh
pip install cloudscraper
```

Then, check if your virtual environment is activated. On Windows:

```sh
.<venv-name>\Scripts\activate.bat
```

On Linux or macOS:

```sh
source <venv-name>/bin/activate
```

### `cloudscraper can’t bypass the latest Cloudflare version`

Update the package:

```sh
pip install -U cloudscraper
```

## cloudscraper Alternatives

Bright Data provides robust proxy networks to bypass Cloudflare. Create an account, configure it, and get your API credentials. Then, use those credentials to access the data at your target URL like this:

```python
import requests

host = 'brd.superproxy.io'
port = 22225

username = 'brd-customer-<customer_id>-zone-<zone_name>'
password = '<zone_password>'

proxy_url = f'http://{username}:{password}@{host}:{port}'

proxies = {
    'http': proxy_url,
    'https': proxy_url
}

response = requests.get(URL, proxies=proxies)
```

Here, you make a `GET` request with the Python Requests library and pass in proxies via the `proxies` parameter.

## Conclusion

While `cloudscraper` is useful, it has its limits. Consider trying Bright Data proxy network and [Web Unlocker](https://brightdata.com/products/web-unlocker) to access Cloudflare-protected sites.

Start with a free trial today!