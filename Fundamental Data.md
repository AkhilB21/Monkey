Fundamental Data Scraping

Gemini
Yes, both Yahoo Finance and Screener.in can be accessed via Python to retrieve fundamental data, but they differ significantly in how well they handle Indian market metrics.1. Yahoo Finance (yfinance library)You can use the open-source yfinance Python library to pull data directly from Yahoo Finance without web scraping.  Pros: It has a built-in API structure, making it very fast to pull standard income statements, balance sheets, and cash flows.Cons: Yahoo Finance is notoriously unreliable for Indian small and mid-cap stocks. It frequently lacks the specific Indian regulatory breakdowns you are looking for, such as Capital Work in Progress (CWIP), Gross Block, and detailed Promoter/FII/DII shareholding splits.2. Scraping Screener.in (Recommended Approach)Since you are already familiar with Screener.in and it provides the exact format you need (including CWIP and precise institutional holdings), scraping it directly is the best approach.Because Screener formats its financial data in standard HTML tables, you do not need to write complex web scrapers to parse individual numbers. Python's pandas library can read the HTML tables directly from the page URL.Here is the core logic to accomplish this:

Python

import pandas as pd
import requests

# Set the symbol exactly as it appears on Screener
symbol = "RELIANCE"
url = f"https://www.screener.in/company/{symbol}/consolidated/"

# Use headers to mimic a browser, preventing your script from being blocked
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
response = requests.get(url, headers=headers)

# pandas.read_html automatically finds all <table> tags and converts them to DataFrames
tables = pd.read_html(response.text)

# The tables are returned as a list. The exact index may vary slightly, but generally:
quarterly_results = tables[0] 
profit_and_loss = tables[1]
balance_sheet = tables[6] # Look for "CWIP" and "Gross Block" here
shareholding = tables[9]  # Look for "FII", "DII", and "Promoters" here

# Display the shareholding pattern table
print(shareholding.head())
3. Targeting the top 1,000 to 1,500 Market Cap Universe To scale this script to your specific universe of stocks, you should break the task into two steps:Generate the Universe: Instead of writing Python to scrape the entire NSE list and filter by market cap, it is much faster to create a free account on Screener.in and run a query like Market Capitalization > 1000 AND Market Capitalization < 1500. Export this resulting list of companies to a CSV file.Iterate and Extract: Write a Python loop that reads the symbols from your downloaded CSV file, dynamically alters the URL in the script above for each ticker, and extracts only the specific rows you need (e.g., CWIP, FII %) into a master DataFrame.Important Note: When running a loop across hundreds of companies, always add a time.sleep(2) delay between requests. Hitting Screener's servers instantly hundreds of times will result in your IP being temporarily blocked.Scraping Screener.in Data with PythonThis tutorial walks through the exact process of identifying the HTML structure of Screener.in and using Python to retrieve and parse the company financial tables.

Claude
No single source covers all four categories cleanly. Assuming a free/DIY pipeline, not a paid vendor.

## Verdict
- Yahoo/yfinance: rule out for Indian fundamentals — skip it.
- Screener.in: no API, manual export only, ToS blocks automation.
- NSE's own endpoints, via the `nse` Python package, are the best free structured path.
- Three of your four data points map cleanly. One (Gross Block) doesn't.

## Sources Ruled Out

**Yahoo Finance / yfinance**
- yfinance is an unofficial scraper of Yahoo's site, not a stable API.
- Its `financials`, `balance_sheet`, and quarterly methods return empty for many tickers since Yahoo changed its page layout and broke that part of the library.
- Coverage thins out fast below large-cap — exactly your 1,000–1,500 Cr band.
- No CWIP, Gross Block, or FII/DII fields exist. Yahoo's schema is US-shaped, not Schedule III.

**Screener.in**
- Screener does not provide an API; CSV export only covers columns from a saved screen, not full time series.
- Full company financials require clicking "Export to Excel" on each company page individually, while logged in.
- No bulk or scriptable version of that export exists.
- Automating it via browser automation would violate their terms — usable as a manual cross-check only.

## NSE's Own Endpoints — the Viable Free Path
- NSE's corporate-filings pages support direct CSV download and XBRL-to-Excel conversion for shareholding pattern filings.
- `pip install nse` wraps these as typed Python calls covering shareholding, financial-results filings, results comparison, and annual reports.
- Actively maintained — PyPI releases through July 2026, 141 GitHub stars, current version 3.2.1.
- Handles NSE's session-cookie anti-bot layer internally.
- Hard rate limit of 3 requests/second, enforced by the library; run bulk pulls after market hours.

## Mapping Your Four Data Points

### Quarterly P&L (Sales, Op Profit, PBT, PAT, EPS)
- `nse.results_comparison(symbol)` returns the quarterly P&L summary directly, typically the last ~5 quarters, with values in Rupees Lakhs — divide by 100 for Crores.
- Beyond 5 quarters, you must snapshot quarterly and accumulate your own history.
- The March-quarter filing's XBRL also carries full-year YTD figures — that covers "yearly."

### Annual P&L, Balance Sheet, CWIP
- `nse.financial_results(period="annual", symbol=...)` returns filing metadata plus an optional XBRL link per year.
- CWIP sits on the face of the Balance Sheet under Schedule III — it's inside that XBRL.
- Gross Block is not. It's disclosed only in Annual Report notes, not the exchange filing.
- For Gross Block/Capex specifically: parse the AR PDF, or use MCA's AOC-4 XBRL (paid, per-filing, far more granular).

### FII/DII holding change
- `nse.shareholding(symbol)` returns only the top-level Promoter (`pr_and_prgrp`) and Public (`public_val`) split per quarter.
- FII (now "FPI") and DII sit one level deeper — inside the linked XBRL.
- Each shareholding record includes an `xbrl` field pointing to that quarter's full filing.
- Write one generic XPath parser against it — the shareholding taxonomy is standardized across companies. Not every historical filing guarantees this link; treat it as optional.

### Promoter change
- Directly available, zero parsing: `pr_and_prgrp` per quarter, no XBRL needed.
- Diff consecutive quarters per symbol to flag promoter buying or selling.

## Working Example
```python
from datetime import datetime, timedelta
from nse import NSE

with NSE("nse_data") as nse:
    symbol = "YOURSYMBOL"

    # Promoter/Public split, latest quarter first
    sh = nse.shareholding(symbol)
    print(sh[0]["date"], sh[0]["pr_and_prgrp"], sh[0]["xbrl"])

    # Trailing quarterly P&L, values in Lakhs
    qtrs = nse.results_comparison(symbol)["resCmpData"]
    for q in qtrs:
        print(q["re_to_dt"], q.get("re_total_inc"), q.get("re_net_profit"))

    # Annual filing metadata + XBRL, for Balance Sheet / CWIP
    annuals = nse.financial_results(
        period="annual", symbol=symbol,
        from_date=datetime.now() - timedelta(days=800),
        to_date=datetime.now(),
    )
```

## Coverage Gaps
- History beyond ~5 live quarters: not available from NSE directly. Two paths:
  - Start snapshotting now, build your own database going forward.
  - One-time manual backfill via Screener's per-company Export to Excel.
- A Kaggle dataset exists with pre-scraped NSE/BSE financials and shareholding patterns for ~4,500 companies — bootstrap only, unofficial, verify a sample against Screener before trusting.
- Cash flow / capex: not filed quarterly at all, and not in structured exchange form even annually. Derive from AR notes, or drop it.

## One Design Flag
- Results file 30–45 days after quarter-end. Shareholding files within 21 days.
- If this feeds any scoring or backtest engine, store the filing/broadcast date, not the period-end date.
- Otherwise you score a quarter using information that wasn't public yet — a look-ahead leak.

GLM'S Response

# Fundamental Data Sources for NSE Stocks - Analysis & Solutions

## Source Comparison Matrix

| Source | Quarterly P&L | Balance Sheet | Shareholding | CWIP/Gross Block | Free/Legal to Scrape |
|--------|---------------|---------------|--------------|------------------|---------------------|
| **Yahoo Finance** | ❌ No | ❌ Limited | ❌ No | ❌ No | ✅ Yes |
| **NSE India** | ✅ Yes | ⚠️ Partial | ✅ Yes | ❌ No | ✅ Yes (with headers) |
| **Screener.in** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Against ToS |
| **MoneyControl** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ❌ Heavy JS |
| **EODHD API** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes | 💰 Paid ($19-49/mo) |
| **Trendlyne API** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | 💰 Paid |

---

## Solution 1: NSE India (Free & Legal) - Recommended Starting Point

```python
import requests
import pandas as pd
import time
from datetime import datetime
import json

class NSEFundamentalScraper:
    """
    Scrapes NSE India for quarterly results and shareholding patterns
    Legal to use with proper headers and rate limiting
    """
    
    def __init__(self):
        self.base_url = "https://www.nseindia.com"
        self.session = requests.Session()
        self._setup_session()
    
    def _setup_session(self):
        """Setup session with required headers to avoid 403"""
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
            'Accept': 'application/json, text/plain, */*',
            'Accept-Language': 'en-US,en;q=0.9',
            'Accept-Encoding': 'gzip, deflate, br',
            'Connection': 'keep-alive',
        })
        # First visit homepage to get cookies
        try:
            self.session.get(self.base_url, timeout=10)
        except:
            pass
    
    def get_quarterly_results(self, symbol: str) -> pd.DataFrame:
        """
        Get quarterly results from NSE
        symbol: NSE symbol like 'RELIANCE', 'TCS', etc.
        """
        url = f"{self.base_url}/api/financial-results"
        params = {
            'index': 'equities',
            'symbol': symbol
        }
        
        try:
            response = self.session.get(url, params=params, timeout=15)
            if response.status_code == 200:
                data = response.json()
                return self._parse_quarterly_results(data)
            else:
                print(f"Error {response.status_code} for {symbol}")
                return None
        except Exception as e:
            print(f"Exception for {symbol}: {e}")
            return None
    
    def _parse_quarterly_results(self, data: dict) -> pd.DataFrame:
        """Parse the quarterly results JSON response"""
        results = []
        
        if 'financialResults' not in data:
            return pd.DataFrame()
        
        for item in data['financialResults']:
            result = {
                'company_name': data.get('companyName', ''),
                'symbol': data.get('symbol', ''),
                'period': item.get('period', ''),
                'quarter_end': item.get('quarterEnd', ''),
                'sales': self._parse_number(item.get('sales', '')),
                'other_income': self._parse_number(item.get('otherIncome', '')),
                'total_income': self._parse_number(item.get('totalIncome', '')),
                'total_expenditure': self._parse_number(item.get('totalExpenditure', '')),
                'pbdt': self._parse_number(item.get('pbdt', '')),
                'pbt': self._parse_number(item.get('pbt', '')),
                'pat': self._parse_number(item.get('pat', '')),
                'reported_eps': self._parse_number(item.get('reportedEPS', '')),
                'diluted_eps': self._parse_number(item.get('dilutedEPS', '')),
            }
            results.append(result)
        
        return pd.DataFrame(results)
    
    def get_shareholding_pattern(self, symbol: str) -> pd.DataFrame:
        """
        Get shareholding pattern from NSE corporate filings
        """
        url = f"{self.base_url}/api/corporate-shareholding"
        params = {
            'index': 'equities',
            'symbol': symbol
        }
        
        try:
            response = self.session.get(url, params=params, timeout=15)
            if response.status_code == 200:
                data = response.json()
                return self._parse_shareholding(data)
            return None
        except Exception as e:
            print(f"Shareholding error for {symbol}: {e}")
            return None
    
    def _parse_shareholding(self, data: dict) -> pd.DataFrame:
        """Parse shareholding pattern data"""
        results = []
        
        if not data or 'shareHoldingPattern' not in data:
            return pd.DataFrame()
        
        for period_data in data['shareHoldingPattern']:
            quarter_end = period_data.get('quarterEnd', '')
            category_data = period_data.get('category', [])
            
            row = {'quarter_end': quarter_end}
            for cat in category_data:
                name = cat.get('name', '')
                percentage = cat.get('percentage', '0')
                shares = cat.get('noOfShares', '0')
                row[f"{name}_pct"] = self._parse_number(percentage)
                row[f"{name}_shares"] = self._parse_number(shares)
            results.append(row)
        
        return pd.DataFrame(results)
    
    def _parse_number(self, value) -> float:
        """Parse number from various formats"""
        if not value:
            return 0.0
        if isinstance(value, (int, float)):
            return float(value)
        # Remove commas, spaces, %, Cr, etc.
        cleaned = str(value).replace(',', '').replace(' ', '').replace('%', '').replace('Cr', '')
        try:
            return float(cleaned)
        except:
            return 0.0
    
    def get_company_info(self, symbol: str) -> dict:
        """Get basic company info including market cap"""
        url = f"{self.base_url}/api/equity-meta-info"
        params = {'symbol': symbol}
        
        try:
            response = self.session.get(url, params=params, timeout=15)
            if response.status_code == 200:
                return response.json()
        except:
            pass
        return {}


# Usage Example
if __name__ == "__main__":
    scraper = NSEFundamentalScraper()
    
    # Get quarterly results
    quarterly = scraper.get_quarterly_results("RELIANCE")
    print("Quarterly Results:")
    print(quarterly)
    
    # Get shareholding pattern
    shareholding = scraper.get_shareholding_pattern("RELIANCE")
    print("\nShareholding Pattern:")
    print(shareholding)
    
    # Always add delay between requests
    time.sleep(1)
```

**Limitations of NSE:**
- No CWIP/Gross Block directly
- No Balance Sheet in structured format
- No consolidated vs standalone toggle easily
- Quarterly results format differs from Screener

---

## Solution 2: Screener.in Scraper (With Caveats)

```python
import requests
import pandas as pd
import time
import re
from bs4 import BeautifulSoup
import cloudscraper

class ScreenerScraper:
    """
    Scrapes Screener.in for comprehensive fundamental data
    WARNING: Against ToS - Use at your own risk
    - May get IP blocked
    - Cloudflare protection
    - Rate limited heavily
    """
    
    def __init__(self, use_stealth: bool = True):
        self.base_url = "https://www.screener.in"
        
        if use_stealth:
            # Cloudscraper bypasses Cloudflare
            self.session = cloudscraper.create_scraper(
                browser={'browser': 'chrome', 'platform': 'windows', 'mobile': False}
            )
        else:
            self.session = requests.Session()
        
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.5',
        })
        
        # Optional: Login if you have account (better rate limits)
        self.logged_in = False
    
    def login(self, username: str, password: str) -> bool:
        """Login to Screener for better rate limits"""
        login_url = f"{self.base_url}/login/"
        
        # Get CSRF token
        try:
            response = self.session.get(login_url, timeout=15)
            soup = BeautifulSoup(response.text, 'html.parser')
            csrf = soup.find('input', {'name': 'csrfmiddlewaretoken'})
            
            if csrf:
                login_data = {
                    'csrfmiddlewaretoken': csrf.get('value'),
                    'username': username,
                    'password': password,
                }
                headers = {'Referer': login_url}
                response = self.session.post(login_url, data=login_data, 
                                           headers=headers, timeout=15)
                self.logged_in = response.status_code == 200
                return self.logged_in
        except Exception as e:
            print(f"Login failed: {e}")
        
        return False
    
    def get_company_by_name(self, company_name: str) -> dict:
        """Search for company and get its data"""
        # First search for the company
        search_url = f"{self.base_url}/api/company/search/"
        params = {'q': company_name}
        
        try:
            response = self.session.get(search_url, params=params, timeout=10)
            if response.status_code == 200:
                results = response.json()
                if results and len(results) > 0:
                    # Get the first result's URL
                    slug = results[0].get('slug', '')
                    if slug:
                        return self.get_company_data(slug)
        except Exception as e:
            print(f"Search error: {e}")
        
        return {}
    
    def get_company_data(self, company_slug: str, consolidated: bool = True) -> dict:
        """
        Get comprehensive company data from Screener
        company_slug: URL slug like 'reliance-industries', 'tcs', etc.
        """
        url = f"{self.base_url}/company/{company_slug}/"
        if consolidated:
            url += "?consolidated=True"
        
        try:
            response = self.session.get(url, timeout=20)
            
            if response.status_code == 403:
                print("Blocked by Cloudflare - try again later or use proxy")
                return {}
            
            if response.status_code != 200:
                print(f"Error {response.status_code}")
                return {}
            
            soup = BeautifulSoup(response.text, 'html.parser')
            
            data = {
                'quarterly_results': self._parse_quarterly_table(soup),
                'profit_loss': self._parse_annual_table(soup, 'Profit & Loss'),
                'balance_sheet': self._parse_annual_table(soup, 'Balance Sheet'),
                'cash_flows': self._parse_annual_table(soup, 'Cash Flows'),
                'ratios': self._parse_annual_table(soup, 'Ratios'),
                'shareholding': self._parse_shareholding(soup),
                'company_name': self._get_company_name(soup),
            }
            
            return data
            
        except Exception as e:
            print(f"Error fetching {company_slug}: {e}")
            return {}
    
    def _get_company_name(self, soup) -> str:
        """Extract company name from page"""
        name_tag = soup.find('h1', class_='company-name')
        return name_tag.text.strip() if name_tag else ""
    
    def _parse_quarterly_table(self, soup) -> pd.DataFrame:
        """Parse quarterly results table"""
        tables = soup.find_all('table', {'data-table': 'quarterly-results'})
        
        if not tables:
            # Fallback: find by section
            for section in soup.find_all('section'):
                heading = section.find('h2')
                if heading and 'Quarterly' in heading.text:
                    tables = section.find_all('table')
                    break
        
        if not tables:
            return pd.DataFrame()
        
        return self._extract_table_data(tables[0])
    
    def _parse_annual_table(self, soup, table_name: str) -> pd.DataFrame:
        """Parse annual tables (P&L, Balance Sheet, Cash Flows, Ratios)"""
        for section in soup.find_all('section'):
            heading = section.find('h2')
            if heading and table_name in heading.text:
                table = section.find('table')
                if table:
                    return self._extract_table_data(table)
        
        return pd.DataFrame()
    
    def _extract_table_data(self, table) -> pd.DataFrame:
        """Extract data from HTML table"""
        rows = table.find_all('tr')
        
        if not rows:
            return pd.DataFrame()
        
        # Extract headers (first row usually has periods)
        header_row = rows[0]
        headers = ['Metric']
        for cell in header_row.find_all(['th', 'td'])[1:]:
            headers.append(cell.text.strip())
        
        data = []
        for row in rows[1:]:
            cells = row.find_all(['th', 'td'])
            if not cells:
                continue
            
            row_data = [cells[0].text.strip()]
            for cell in cells[1:]:
                row_data.append(self._parse_number(cell.text.strip()))
            
            if row_data[0]:  # Only add if there's a metric name
                data.append(row_data)
        
        df = pd.DataFrame(data, columns=headers)
        df = df.set_index('Metric')
        return df
    
    def _parse_shareholding(self, soup) -> pd.DataFrame:
        """Parse shareholding pattern table"""
        for section in soup.find_all('section'):
            heading = section.find('h2')
            if heading and 'Shareholding' in heading.text:
                table = section.find('table')
                if table:
                    return self._extract_table_data(table)
        
        return pd.DataFrame()
    
    def _parse_number(self, value) -> float:
        """Parse number from Indian format (1,234.56)"""
        if not value or value == '-':
            return 0.0
        if isinstance(value, (int, float)):
            return float(value)
        
        # Handle percentage
        if '%' in value:
            value = value.replace('%', '')
        
        # Remove commas
        cleaned = value.replace(',', '')
        
        try:
            return float(cleaned)
        except:
            return 0.0
    
    def get_cwip_and_capex(self, company_slug: str) -> dict:
        """Extract specific CWIP and Capex data"""
        data = self.get_company_data(company_slug)
        
        result = {
            'cwip': [],
            'gross_block': [],
            'capex': []  # Calculated from change in gross block + CWIP
        }
        
        if 'balance_sheet' in data and not data['balance_sheet'].empty:
            bs = data['balance_sheet']
            
            if 'CWIP' in bs.index:
                result['cwip'] = bs.loc['CWIP'].tolist()
            
            if 'Gross Block' in bs.index:
                result['gross_block'] = bs.loc['Gross Block'].tolist()
                # Calculate Capex as change in gross block
                gb_values = result['gross_block']
                result['capex'] = [0] + [gb_values[i] - gb_values[i-1] 
                                        for i in range(1, len(gb_values))]
        
        if 'cash_flows' in data and not data['cash_flows'].empty:
            cf = data['cash_flows']
            if 'Capital Expenditure' in cf.index:
                result['capex_from_cf'] = cf.loc['Capital Expenditure'].tolist()
        
        return result


# Usage with caution
if __name__ == "__main__":
    scraper = ScreenerScraper(use_stealth=True)
    
    # Option 1: Direct slug if you know it
    data = scraper.get_company_data('reliance-industries')
    
    print("Company:", data.get('company_name'))
    print("\nQuarterly Results:")
    print(data.get('quarterly_results'))
    
    print("\nBalance Sheet:")
    print(data.get('balance_sheet'))
    
    print("\nShareholding:")
    print(data.get('shareholding'))
    
    # Option 2: Search by name
    # data = scraper.get_company_by_name("Reliance Industries")
    
    # CRITICAL: Add delay between requests (at least 5-10 seconds)
    time.sleep(10)
```

---

## Solution 3: Hybrid Approach Using Multiple Free Sources

```python
import requests
import pandas as pd
import time
from typing import Dict, List, Optional
import json

class HybridFundamentalDataFetcher:
    """
    Combines multiple free sources to get comprehensive data
    More reliable than single source scraping
    """
    
    def __init__(self):
        self.nse = NSEFundamentalScraper()
        self.cache = {}
    
    def get_complete_data(self, symbol: str, screener_slug: Optional[str] = None) -> Dict:
        """
        Get comprehensive data for a stock using multiple sources
        """
        print(f"Fetching data for {symbol}...")
        
        result = {
            'symbol': symbol,
            'timestamp': pd.Timestamp.now().isoformat(),
            'quarterly_results': None,
            'annual_pnl': None,
            'balance_sheet': None,
            'cash_flows': None,
            'ratios': None,
            'shareholding': None,
            'cwip_capex': None,
            'data_quality': {}  # Track which sources worked
        }
        
        # Source 1: NSE India (always try first - legal and reliable)
        try:
            quarterly = self.nse.get_quarterly_results(symbol)
            if quarterly is not None and not quarterly.empty:
                result['quarterly_results'] = quarterly
                result['data_quality']['nse_quarterly'] = 'success'
            else:
                result['data_quality']['nse_quarterly'] = 'no_data'
            time.sleep(1)
        except Exception as e:
            result['data_quality']['nse_quarterly'] = f'error: {str(e)}'
        
        try:
            shareholding = self.nse.get_shareholding_pattern(symbol)
            if shareholding is not None and not shareholding.empty:
                result['shareholding'] = shareholding
                result['data_quality']['nse_shareholding'] = 'success'
            else:
                result['data_quality']['nse_shareholding'] = 'no_data'
            time.sleep(1)
        except Exception as e:
            result['data_quality']['nse_shareholding'] = f'error: {str(e)}'
        
        # Source 2: Screener (if slug provided - use with caution)
        if screener_slug:
            try:
                screener_data = self._get_screener_data_safe(screener_slug)
                if screener_data:
                    result.update(screener_data)
                    result['data_quality']['screener'] = 'success'
            except Exception as e:
                result['data_quality']['screener'] = f'error: {str(e)}'
        
        return result
    
    def _get_screener_data_safe(self, slug: str) -> Optional[Dict]:
        """Safely get screener data with proper error handling"""
        try:
            scraper = ScreenerScraper(use_stealth=True)
            data = scraper.get_company_data(slug)
            
            if data and data.get('quarterly_results') is not None:
                return {
                    'annual_pnl': data.get('profit_loss'),
                    'balance_sheet': data.get('balance_sheet'),
                    'cash_flows': data.get('cash_flows'),
                    'ratios': data.get('ratios'),
                    'cwip_capex': scraper.get_cwip_and_capex(slug),
                }
        except:
            pass
        return None
    
    def extract_fii_dii_changes(self, shareholding_df: pd.DataFrame) -> Dict:
        """Calculate FII/DII holding changes from shareholding data"""
        if shareholding_df is None or shareholding_df.empty:
            return {}
        
        changes = {}
        
        # Look for FII/FPI columns
        fii_cols = [c for c in shareholding_df.columns if any(x in c.upper() 
                   for x in ['FII', 'FPI', 'FOREIGN'])]
        
        dii_cols = [c for c in shareholding_df.columns if any(x in c.upper() 
                   for x in ['DII', 'MUTUAL', 'INSTITUTIONAL'])]
        
        promoter_cols = [c for c in shareholding_df.columns if 'PROMOTER' in c.upper()]
        
        for col in fii_cols:
            if '_pct' in col:
                values = shareholding_df[col].dropna().tolist()
                if len(values) >= 2:
                    changes['fii_latest'] = values[-1]
                    changes['fii_previous'] = values[-2]
                    changes['fii_change'] = values[-1] - values[-2]
                    changes['fii_change_bps'] = (values[-1] - values[-2]) * 100
        
        for col in dii_cols:
            if '_pct' in col:
                values = shareholding_df[col].dropna().tolist()
                if len(values) >= 2:
                    changes['dii_latest'] = values[-1]
                    changes['dii_previous'] = values[-2]
                    changes['dii_change'] = values[-1] - values[-2]
        
        for col in promoter_cols:
            if '_pct' in col:
                values = shareholding_df[col].dropna().tolist()
                if len(values) >= 2:
                    changes['promoter_latest'] = values[-1]
                    changes['promoter_previous'] = values[-2]
                    changes['promoter_change'] = values[-1] - values[-2]
        
        return changes
    
    def get_capex_metrics(self, balance_sheet: pd.DataFrame, 
                          cash_flows: pd.DataFrame) -> Dict:
        """Extract and calculate capex related metrics"""
        if balance_sheet is None or balance_sheet.empty:
            return {}
        
        metrics = {}
        
        # CWIP
        if 'CWIP' in balance_sheet.index:
            cwip = balance_sheet.loc['CWIP']
            metrics['cwip_latest'] = cwip.iloc[-1] if len(cwip) > 0 else 0
            metrics['cwip_trend'] = cwip.tolist()
        
        # Gross Block (if available)
        if 'Gross Block' in balance_sheet.index:
            gb = balance_sheet.loc['Gross Block']
            metrics['gross_block_latest'] = gb.iloc[-1] if len(gb) > 0 else 0
            metrics['gross_block_trend'] = gb.tolist()
            
            # Calculate Capex as change in gross block
            gb_list = gb.tolist()
            if len(gb_list) >= 2:
                metrics['capex_change_in_gb'] = gb_list[-1] - gb_list[-2]
        
        # Net Fixed Assets
        if 'Net Fixed Assets' in balance_sheet.index:
            nfa = balance_sheet.loc['Net Fixed Assets']
            metrics['net_fixed_assets'] = nfa.tolist()
        
        # Capex from Cash Flow (Capital Expenditure is usually negative)
        if cash_flows is not None and not cash_flows.empty:
            if 'Capital Expenditure' in cash_flows.index:
                capex_cf = cash_flows.loc['Capital Expenditure']
                metrics['capex_from_cf'] = [abs(x) if x < 0 else 0 for x in capex_cf.tolist()]
            elif 'Purchase of Fixed Assets' in cash_flows.index:
                pfa = cash_flows.loc['Purchase of Fixed Assets']
                metrics['capex_from_cf'] = [abs(x) if x < 0 else 0 for x in pfa.tolist()]
        
        return metrics


# Symbol to Screener slug mapping (you'll need to build this)
SYMBOL_TO_SLUG = {
    'RELIANCE': 'reliance-industries',
    'TCS': 'tcs',
    'INFY': 'infosys',
    'HDFCBANK': 'hdfc-bank',
    'ICICIBANK': 'icici-bank',
    # Add more as needed
}


# Main execution
if __name__ == "__main__":
    fetcher = HybridFundamentalDataFetcher()
    
    symbols = ['RELIANCE', 'TCS', 'INFY']
    all_data = {}
    
    for symbol in symbols:
        slug = SYMBOL_TO_SLUG.get(symbol)
        data = fetcher.get_complete_data(symbol, slug)
        all_data[symbol] = data
        
        # Print summary
        print(f"\n{'='*50}")
        print(f"{symbol} Data Summary:")
        print(f"{'='*50}")
        print(f"Data Quality: {data['data_quality']}")
        
        if data.get('shareholding') is not None:
            changes = fetcher.extract_fii_dii_changes(data['shareholding'])
            print(f"FII/DII Changes: {changes}")
        
        if data.get('balance_sheet') is not None:
            capex = fetcher.get_capex_metrics(data['balance_sheet'], data.get('cash_flows'))
            print(f"Capex Metrics: {capex}")
        
        # IMPORTANT: Rate limiting
        time.sleep(5)


# Save to JSON/CSV
def save_data(all_data: Dict, filename: str = 'fundamental_data'):
    """Save fetched data to files"""
    import os
    
    # Save raw data as JSON
    with open(f'{filename}.json', 'w') as f:
        json.dump(all_data, f, indent=2, default=str)
    
    # Create summary DataFrame for shareholding changes
    summary_rows = []
    fetcher = HybridFundamentalDataFetcher()
    
    for symbol, data in all_data.items():
        row = {'symbol': symbol}
        
        if data.get('shareholding') is not None:
            changes = fetcher.extract_fii_dii_changes(data['shareholding'])
            row.update(changes)
        
        if data.get('balance_sheet') is not None:
            capex = fetcher.get_capex_metrics(data['balance_sheet'], data.get('cash_flows'))
            row['cwip_latest'] = capex.get('cwip_latest', 0)
            row['capex_from_cf_latest'] = capex.get('capex_from_cf', [0])[-1] if capex.get('capex_from_cf') else 0
        
        summary_rows.append(row)
    
    summary_df = pd.DataFrame(summary_rows)
    summary_df.to_csv(f'{filename}_summary.csv', index=False)
    print(f"Data saved to {filename}.json and {filename}_summary.csv")
```

---

## Solution 4: Paid API (Most Reliable for Production)

```python
import requests
import pandas as pd
from typing import Dict, Optional

class EODHDFundamentalFetcher:
    """
    EOD Historical Data API - Has good Indian stock fundamentals
    Pricing: $19/month for basic, $49/month for all features
    Website: eodhd.com
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://eodhistoricaldata.com/api"
    
    def get_fundamentals(self, symbol: str, exchange: str = "NSE") -> Dict:
        """
        Get fundamental data for Indian stock
        symbol: e.g., 'RELIANCE', 'TCS'
        exchange: 'NSE' or 'BSE'
        """
        url = f"{self.base_url}/fundamentals/{symbol}.{exchange}"
        params = {
            'api_token': self.api_key,
            'filter': ''  # Empty for all data
        }
        
        try:
            response = requests.get(url, params=params, timeout=30)
            if response.status_code == 200:
                return response.json()
            else:
                print(f"Error {response.status_code}: {response.text}")
                return {}
        except Exception as e:
            print(f"Error: {e}")
            return {}
    
    def get_income_statement(self, symbol: str, quarterly: bool = True) -> pd.DataFrame:
        """Get income statement (P&L)"""
        data = self.get_fundamentals(symbol)
        
        if not data:
            return pd.DataFrame()
        
        key = 'quarterly' if quarterly else 'annual'
        income = data.get('Financials', {}).get('Income_Statement', {}).get(key, {})
        
        if not income:
            return pd.DataFrame()
        
        return pd.DataFrame(income).T
    
    def get_balance_sheet(self, symbol: str, quarterly: bool = False) -> pd.DataFrame:
        """Get balance sheet"""
        data = self.get_fundamentals(symbol)
        
        if not data:
            return pd.DataFrame()
        
        key = 'quarterly' if quarterly else 'annual'
        bs = data.get('Financials', {}).get('Balance_Sheet', {}).get(key, {})
        
        if not bs:
            return pd.DataFrame()
        
        return pd.DataFrame(bs).T
    
    def get_cash_flow(self, symbol: str, quarterly: bool = False) -> pd.DataFrame:
        """Get cash flow statement"""
        data = self.get_fundamentals(symbol)
        
        if not data:
            return pd.DataFrame()
        
        key = 'quarterly' if quarterly else 'annual'
        cf = data.get('Financials', {}).get('Cash_Flow', {}).get(key, {})
        
        if not cf:
            return pd.DataFrame()
        
        return pd.DataFrame(cf).T
    
    def get_cwip_capex(self, symbol: str) -> Dict:
        """Extract CWIP and Capex data"""
        bs = self.get_balance_sheet(symbol)
        cf = self.get_cash_flow(symbol)
        
        result = {}
        
        if not bs.empty:
            # Look for capital work in progress
            for idx in bs.index:
                if 'capital' in idx.lower() and 'progress' in idx.lower():
                    result['cwip'] = bs.loc[idx].to_dict()
                if 'gross' in idx.lower() and ('fixed' in idx.lower() or 'block' in idx.lower()):
                    result['gross_fixed_assets'] = bs.loc[idx].to_dict()
        
        if not cf.empty:
            for idx in cf.index:
                if 'capital' in idx.lower() and 'expenditure' in idx.lower():
                    result['capex'] = cf.loc[idx].to_dict()
                if 'purchase' in idx.lower() and 'fixed' in idx.lower():
                    result['purchase_of_fixed_assets'] = cf.loc[idx].to_dict()
        
        return result


# Usage example
if __name__ == "__main__":
    # Get your API key from eodhd.com
    API_KEY = "YOUR_API_KEY"
    
    fetcher = EODHDFundamentalFetcher(API_KEY)
    
    # Get all fundamental data
    fundamentals = fetcher.get_fundamentals("RELIANCE")
    
    # Get specific statements
    income = fetcher.get_income_statement("RELIANCE", quarterly=True)
    balance_sheet = fetcher.get_balance_sheet("RELIANCE")
    cash_flow = fetcher.get_cash_flow("RELIANCE")
    cwip_capex = fetcher.get_cwip_capex("RELIANCE")
    
    print("Income Statement:")
    print(income.head())
    
    print("\nCWIP/Capex:")
    print(cwip_capex)
```

---

## Solution 5: Direct from Corporate Filings (Most Accurate)

```python
import requests
import pandas as pd
from bs4 import BeautifulSoup
import time
import re
import os
from io import BytesIO

class CorporateFilingParser:
    """
    Parse quarterly results directly from NSE/BSE corporate filings
    Most accurate but requires more parsing
    """
    
    def __init__(self):
        self.nse_base = "https://www.nseindia.com"
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
        })
    
    def get_recent_filings(self, symbol: str) -> list:
        """Get recent corporate filings for a company"""
        url = f"{self.nse_base}/api/corporate-filings"
        params = {
            'index': 'equities',
            'symbol': symbol,
            'period': '1M'  # Last 1 month
        }
        
        try:
            self.session.get(self.nse_base, timeout=10)  # Get cookies
            response = self.session.get(url, params=params, timeout=15)
            
            if response.status_code == 200:
                data = response.json()
                filings = []
                
                for f in data:
                    if 'financial results' in f.get('filingsDesc', '').lower():
                        filings.append({
                            'date': f.get('filingDate', ''),
                            'description': f.get('filingsDesc', ''),
                            'pdf_url': f.get('pdfUrl', ''),
                        })
                
                return filings
        except Exception as e:
            print(f"Error: {e}")
        
        return []
    
    def download_and_parse_excel(self, pdf_url: str) -> Optional[pd.DataFrame]:
        """
        Many companies upload Excel files disguised as .pdf
        Try to parse them directly
        """
        try:
            response = self.session.get(pdf_url, timeout=30)
            
            # Check if it's actually an Excel file
            if response.content[:4] == b'PK\x03\x04':  # ZIP/XLSX magic number
                df = pd.read_excel(BytesIO(response.content))
                return df
            elif b'\xd0\xcf\x11\xe0' in response.content[:8]:  # XLS magic number
                df = pd.read_excel(BytesIO(response.content))
                return df
                
        except Exception as e:
            pass
        
        return None


# Alternative: Use BSE API which is more structured
class BSEFundamentalFetcher:
    """BSE India has better structured data APIs"""
    
    def __init__(self):
        self.base_url = "https://www.bseindia.com"
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
        })
    
    def get_quote_data(self, scrip_code: str) -> dict:
        """Get basic quote and fundamental data from BSE"""
        url = f"{self.base_url}/api/MarketData/GetQuote"
        params = {
            'flag': 'EQ',
            'scripcode': scrip_code,
        }
        
        try:
            response = self.session.get(url, params=params, timeout=15)
            if response.status_code == 200:
                return response.json()
        except Exception as e:
            print(f"Error: {e}")
        
        return {}
    
    def get_shareholding_pattern(self, scrip_code: str) -> pd.DataFrame:
        """Get shareholding pattern from BSE"""
        # BSE provides this as a downloadable CSV
        url = f"https://www.bseindia.com/corporates/shpSecurities.html"
        params = {
            'scripcode': scrip_code,
            'qtrId': '',  # Empty for latest
        }
        
        try:
            response = self.session.get(url, params=params, timeout=15)
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # Parse the table
            table = soup.find('table', {'id': 'shpTable'})
            if table:
                rows = table.find_all('tr')
                data = []
                for row in rows[1:]:  # Skip header
                    cols = row.find_all('td')
                    if cols:
                        data.append([c.text.strip() for c in cols])
                
                if data:
                    df = pd.DataFrame(data)
                    df.columns = ['Category'] + [f'Q{i}' for i in range(len(data[0])-1)]
                    return df
        except Exception as e:
            print(f"Error: {e}")
        
        return pd.DataFrame()
```

---

## Recommended Architecture for Your Use Case

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOUR ANALYSIS PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ Stock List   │───▶│ Data Fetcher │───▶│ Data Store   │       │
│  │ (Top 1000-   │    │ (Hybrid)     │    │ (SQLite/CSV) │       │
│  │  1500 Cr)    │    │              │    │              │       │
│  └──────────────┘    └──────┬───────┘    └──────┬───────┘       │
│                             │                   │               │
│              ┌──────────────┼──────────────┐    │               │
│              ▼              ▼              ▼    ▼               │
│         ┌────────┐    ┌────────┐    ┌────────┐ ┌────────┐      │
│         │  NSE   │    │Screener│    │  BSE   │ │  API   │      │
│         │(Legal) │    │(Risky) │    │(Backup)│ │(Paid)  │      │
│         └────────┘    └────────┘    └────────┘ └────────┘      │
│                                                                  │
│  Priority: NSE → BSE → API → Screener (last resort)             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Complete Solution with Local Database

```python
import sqlite3
import pandas as pd
import json
import time
from datetime import datetime
from pathlib import Path

class FundamentalDataPipeline:
    """
    Complete pipeline with local SQLite storage
    Handles retries, caching, and incremental updates
    """
    
    def __init__(self, db_path: str = "fundamental_data.db"):
        self.db_path = db_path
        self.nse = NSEFundamentalScraper()
        self._init_db()
    
    def _init_db(self):
        """Initialize SQLite database"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Quarterly results table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS quarterly_results (
                symbol TEXT,
                period TEXT,
                quarter_end TEXT,
                sales REAL,
                other_income REAL,
                total_income REAL,
                total_expenditure REAL,
                pbdt REAL,
                pbt REAL,
                pat REAL,
                reported_eps REAL,
                diluted_eps REAL,
                fetched_at TEXT,
                PRIMARY KEY (symbol, period)
            )
        ''')
        
        # Shareholding pattern table
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS shareholding (
                symbol TEXT,
                quarter_end TEXT,
                category TEXT,
                percentage REAL,
                no_of_shares REAL,
                fetched_at TEXT,
                PRIMARY KEY (symbol, quarter_end, category)
            )
        ''')
        
        # Screener data (JSON format for flexibility)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS screener_data (
                symbol TEXT,
                data_type TEXT,
                data_json TEXT,
                fetched_at TEXT,
                PRIMARY KEY (symbol, data_type)
            )
        ''')
        
        # Data quality tracking
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS data_quality (
                symbol TEXT,
                source TEXT,
                status TEXT,
                message TEXT,
                fetched_at TEXT
            )
        ''')
        
        conn.commit()
        conn.close()
    
    def update_stock(self, symbol: str, screener_slug: str = None, 
                     max_retries: int = 3) -> dict:
        """
        Update all data for a single stock
        """
        result = {
            'symbol': symbol,
            'success': False,
            'updated': [],
            'errors': []
        }
        
        # 1. Try NSE quarterly results
        for attempt in range(max_retries):
            try:
                quarterly = self.nse.get_quarterly_results(symbol)
                if quarterly is not None and not quarterly.empty:
                    self._save_quarterly(symbol, quarterly)
                    result['updated'].append('nse_quarterly')
                    break
            except Exception as e:
                result['errors'].append(f"NSE quarterly attempt {attempt+1}: {e}")
            time.sleep(2 * (attempt + 1))
        
        # 2. Try NSE shareholding
        for attempt in range(max_retries):
            try:
                shareholding = self.nse.get_shareholding_pattern(symbol)
                if shareholding is not None and not shareholding.empty:
                    self._save_shareholding(symbol, shareholding)
                    result['updated'].append('nse_shareholding')
                    break
            except Exception as e:
                result['errors'].append(f"NSE shareholding attempt {attempt+1}: {e}")
            time.sleep(2 * (attempt + 1))
        
        # 3. Try Screener (if slug provided)
        if screener_slug:
            try:
                scraper = ScreenerScraper(use_stealth=True)
                data = scraper.get_company_data(screener_slug)
                
                if data:
                    for dtype in ['profit_loss', 'balance_sheet', 'cash_flows', 'ratios']:
                        if data.get(dtype) is not None:
                            self._save_screener_data(symbol, dtype, data[dtype])
                            result['updated'].append(f'screener_{dtype}')
                    
                    result['success'] = True
            except Exception as e:
                result['errors'].append(f"Screener: {e}")
        
        if result['updated']:
            result['success'] = True
        
        return result
    
    def _save_quarterly(self, symbol: str, df: pd.DataFrame):
        """Save quarterly results to database"""
        conn = sqlite3.connect(self.db_path)
        df['symbol'] = symbol
        df['fetched_at'] = datetime.now().isoformat()
        df.to_sql('quarterly_results', conn, if_exists='append', index=False)
        conn.close()
    
    def _save_shareholding(self, symbol: str, df: pd.DataFrame):
        """Save shareholding pattern to database (melt format)"""
        conn = sqlite3.connect(self.db_path)
        
        # Melt the DataFrame to long format
        id_vars = ['quarter_end']
        value_vars = [c for c in df.columns if c != 'quarter_end']
        
        if value_vars:
            melted = df.melt(id_vars=id_vars, value_vars=value_vars, 
                           var_name='category', value_name='percentage')
            melted['symbol'] = symbol
            melted['fetched_at'] = datetime.now().isoformat()
            melted.to_sql('shareholding', conn, if_exists='append', index=False)
        
        conn.close()
    
    def _save_screener_data(self, symbol: str, data_type: str, df: pd.DataFrame):
        """Save screener data as JSON"""
        conn = sqlite3.connect(self.db_path)
        
        data_json = df.to_json()
        fetched_at = datetime.now().isoformat()
        
        conn.execute('''
            INSERT OR REPLACE INTO screener_data (symbol, data_type, data_json, fetched_at)
            VALUES (?, ?, ?, ?)
        ''', (symbol, data_type, data_json, fetched_at))
        
        conn.commit()
        conn.close()
    
    def get_quarterly_results(self, symbol: str) -> pd.DataFrame:
        """Retrieve quarterly results from database"""
        conn = sqlite3.connect(self.db_path)
        df = pd.read_sql('''
            SELECT * FROM quarterly_results 
            WHERE symbol = ? 
            ORDER BY period DESC
        ''', conn, params=(symbol,))
        conn.close()
        return df
    
    def get_shareholding_changes(self, symbol: str, quarters: int = 4) -> pd.DataFrame:
        """Get shareholding pattern changes for last N quarters"""
        conn = sqlite3.connect(self.db_path)
        df = pd.read_sql('''
            SELECT quarter_end, category, percentage 
            FROM shareholding 
            WHERE symbol = ? AND category LIKE '%_pct'
            ORDER BY quarter_end DESC
            LIMIT ?
        ''', conn, params=(symbol, quarters * 10))
        conn.close()
        
        if df.empty:
            return df
        
        # Pivot to wide format
        return df.pivot(index='quarter_end', columns='category', values='percentage')
    
    def get_cwip_capex(self, symbol: str) -> dict:
        """Get CWIP and Capex data from screener data"""
        conn = sqlite3.connect(self.db_path)
        
        result = {}
        
        # Get balance sheet
        cursor = conn.execute('''
            SELECT data_json FROM screener_data 
            WHERE symbol = ? AND data_type = 'balance_sheet'
        ''', (symbol,))
        
        row = cursor.fetchone()
        if row:
            bs = pd.read_json(row[0])
            
            if 'CWIP' in bs.index:
                result['cwip'] = bs.loc['CWIP'].to_dict()
            
            if 'Gross Block' in bs.index:
                gb = bs.loc['Gross Block'].to_dict()
                result['gross_block'] = gb
                # Calculate capex as change
                values = list(gb.values())
                result['capex_change'] = {k: values[i] - values[i-1] 
                                         for i, k in enumerate(list(gb.keys())[1:], 1)}
        
        # Get cash flow for capex
        cursor = conn.execute('''
            SELECT data_json FROM screener_data 
            WHERE symbol = ? AND data_type = 'cash_flows'
        ''', (symbol,))
        
        row = cursor.fetchone()
        if row:
            cf = pd.read_json(row[0])
            
            for idx in cf.index:
                if 'capital expenditure' in idx.lower() or 'purchase' in idx.lower():
                    result['capex_from_cf'] = cf.loc[idx].to_dict()
        
        conn.close()
        return result
    
    def batch_update(self, symbols: list, slug_mapping: dict = None, 
                     delay: float = 5.0) -> dict:
        """
        Update multiple stocks with proper rate limiting
        """
        if slug_mapping is None:
            slug_mapping = {}
        
        results = {
            'total': len(symbols),
            'success': 0,
            'failed': 0,
            'details': []
        }
        
        for i, symbol in enumerate(symbols):
            print(f"[{i+1}/{len(symbols)}] Processing {symbol}...")
            
            slug = slug_mapping.get(symbol)
            result = self.update_stock(symbol, slug)
            result['updated'].sort()
            
            results['details'].append(result)
            
            if result['success']:
                results['success'] += 1
            else:
                results['failed'] += 1
            
            # Rate limiting
            if i < len(symbols) - 1:
                time.sleep(delay)
        
        return results
    
    def export_summary(self, output_path: str = "fundamental_summary.csv"):
        """Export a summary of all data to CSV"""
        conn = sqlite3.connect(self.db_path)
        
        # Get all symbols
        symbols = pd.read_sql('SELECT DISTINCT symbol FROM quarterly_results', conn)['symbol'].tolist()
        
        summary_rows = []
        for symbol in symbols:
            row = {'symbol': symbol}
            
            # Latest quarterly
            quarterly = pd.read_sql('''
                SELECT sales, pat, reported_eps FROM quarterly_results 
                WHERE symbol = ? ORDER BY period DESC LIMIT 1
            ''', conn, params=(symbol,))
            
            if not quarterly.empty:
                row['latest_sales'] = quarterly['sales'].iloc[0]
                row['latest_pat'] = quarterly['pat'].iloc[0]
                row['latest_eps'] = quarterly['reported_eps'].iloc[0]
            
            # Shareholding changes
            shareholding = self.get_shareholding_changes(symbol, 2)
            if not shareholding.empty:
                for col in shareholding.columns:
                    if len(shareholding[col]) >= 2:
                        change = shareholding[col].iloc[0] - shareholding[col].iloc[1]
                        row[f'{col}_change'] = change
            
            # CWIP/Capex
            capex = self.get_cwip_capex(symbol)
            if 'cwip' in capex:
                cwip_values = list(capex['cwip'].values())
                row['latest_cwip'] = cwip_values[0] if cwip_values else 0
            
            summary_rows.append(row)
        
        conn.close()
        
        summary_df = pd.DataFrame(summary_rows)
        summary_df.to_csv(output_path, index=False)
        print(f"Summary exported to {output_path}")
        return summary_df


# Main execution
if __name__ == "__main__":
    # Initialize pipeline
    pipeline = FundamentalDataPipeline("nse_fundamentals.db")
    
    # Define your stock universe
    symbols = [
        'RELIANCE', 'TCS', 'HDFCBANK', 'INFY', 'ICICIBANK',
        'HINDUNILVR', 'ITC', 'SBIN', 'BHARTIARTL', 'KOTAKBANK',
        'LT', 'AXISBANK', 'BAJFINANCE', 'MARUTI', 'HCLTECH',
        # Add more symbols as needed
    ]
    
    # Slug mapping (build this once)
    slug_mapping = {
        'RELIANCE': 'reliance-industries',
        'TCS': 'tcs',
        'HDFCBANK': 'hdfc-bank',
        'INFY': 'infosys',
        'ICICIBANK': 'icici-bank',
        # Add more...
    }
    
    # Run batch update (with 5 second delay between each)
    print("Starting batch update...")
    results = pipeline.batch_update(symbols, slug_mapping, delay=5.0)
    
    print(f"\nUpdate Complete:")
    print(f"  Success: {results['success']}")
    print(f"  Failed: {results['failed']}")
    
    # Export summary
    pipeline.export_summary()
    
    # Query specific data
    print("\n" + "="*50)
    print("Sample Query - RELIANCE Quarterly Results:")
    print("="*50)
    print(pipeline.get_quarterly_results('RELIANCE').head())
    
    print("\n" + "="*50)
    print("Sample Query - TCS Shareholding Changes:")
    print("="*50)
    print(pipeline.get_shareholding_changes('TCS', 4))
    
    print("\n" + "="*50)
    print("Sample Query - RELIANCE CWIP/Capex:")
    print("="*50)
    print(pipeline.get_cwip_capex('RELIANCE'))
```

---

## My Recommendation

| Scenario | Recommended Approach |
|----------|---------------------|
| **One-time analysis, 10-20 stocks** | Manual Screener export + NSE scraping |
| **Ongoing analysis, 100-500 stocks** | NSE (legal) + EODHD API ($19/mo) |
| **Production system, 1000+ stocks** | Paid API (EODHD/Trendlyne) + Local DB |
| **Budget constrained, learning** | NSE only + Manual Screener fallback |

### Key Points:

1. **Yahoo Finance is NOT suitable** for the data you need on Indian stocks

2. **NSE India is your best free option** but lacks CWIP/Gross Block

3. **Screener.in has all the data** but:
   - Against ToS
   - Will block you eventually
   - Use only for one-time dumps

4. **For production use EODHD API** - $19/month gives you:
   - Quarterly & Annual P&L
   - Balance Sheet (including fixed assets)
   - Cash Flows
   - No CWIP specifically but can calculate from gross block changes

5. **For shareholding patterns** - NSE or BSE are your best free legal sources

Would you like me to elaborate on any specific part or help you build the slug mapping for Screener?