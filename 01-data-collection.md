# Data Collection and ETF Universe Definition

This chapter covers the comprehensive approach to collecting and organizing ETF data for portfolio construction. We'll define our investment universe based on GDP and MSCI classification, scrape ETF data from multiple sources, and verify ETF availability on trading platforms.

## Overview

Data collection is the foundation of any quantitative investment strategy. In this chapter, you'll learn how to:

- Define investment universe based on economic fundamentals
- Scrape ETF data from JustETF platform
- Integrate Alpha Vantage API for detailed ETF profiles and time series data
- Use OpenBB terminal for additional market data
- Organize data by regional categories for portfolio construction
- Handle API keys securely using environment variables

## Environment Setup

Before running the data collection code, you'll need to set up your Alpha Vantage API key:

```bash
export ALPHAVANTAGE_API_KEY="your_api_key_here"
```

Get your free API key from [Alpha Vantage](https://www.alphavantage.co/support/#api-key).

## Required Libraries and Setup

```{code-cell} python
import pandas as pd
import numpy as np
import requests
import json
import os
import justetf_scraping

# Install required package if not already installed
# !pip install git+https://github.com/druzsan/justetf-scraping.git
```

## Define Investment Universe

We'll start by creating a DataFrame of major economies, including their GDP (2023 estimates), MSCI market classification (Developed/Emerging), geographic region, and country codes for data collection.

```{code-cell} python
# Create the country DataFrame using list of lists
country_data = [
    ["United States", "$27.721 trillion", "Developed", "AmericasandUK", "US", "USD"],
    ["Canada", "$2.142 trillion", "Developed", "AmericasandUK", "CA", "CAD"],
    ["United Kingdom", "$3.381 trillion", "Developed", "AmericasandUK", "GB", "GBP"],
    ["Japan", "$4.204 trillion", "Developed", "APAC", "JP", "JPY"],
    ["Australia", "$1.728 trillion", "Developed", "APAC", "AU", "AUD"],
    ["Germany", "$4.526 trillion", "Developed", "EMEA", "DE", "EUR"],
    ["France", "$3.052 trillion", "Developed", "EMEA", "FR", "EUR"],
    ["Italy", "$2.301 trillion", "Developed", "EMEA", "IT", "EUR"],
    ["Spain", "$1.620 trillion", "Developed", "EMEA", "ES", "EUR"],
    ["Netherlands", "$1.154 trillion", "Developed", "EMEA", "NL", "EUR"],
    ["Switzerland", "$884.94 billion", "Developed", "EMEA", "CH", "CHF"],
    ["Brazil", "$2.174 trillion", "Emerging", "Americas", "BR", "BRL"],
    ["Mexico", "$1.789 trillion", "Emerging", "Americas", "MX", "MXN"],
    ["China", "$17.795 trillion", "Emerging", "APACandEMEA", "CN", "CNY"],
    ["India", "$3.568 trillion", "Emerging", "APACandEMEA", "IN", "INR"],
    ["South Korea", "$1.713 trillion", "Emerging", "APACandEMEA", "KR", "KRW"],
    ["Indonesia", "$1.371 trillion", "Emerging", "APACandEMEA", "ID", "IDR"]
]

columns = ["Country", "2023 GDP", "MSCI", "Region", "Short_name", "Currency"]
country_df = pd.DataFrame(country_data, columns=columns)

# Sort by MSCI classification then region
country_df.sort_values(by=["MSCI", "Region"], inplace=True)
print("Investment Universe:")
print(country_df)
```

## ETF Data Collection from JustETF

Now we'll collect ETF data for both equity and bond ETFs across our defined regions using the justetf-scraping library.

```{code-cell} python
import traceback

# Define asset classes to collect
asset_classes = [
    "class-equity", 
    "class-bonds"
]

# Group countries by region for data organization
region_countries = country_df.groupby('Region')['Short_name'].apply(list).to_dict()

# Create mapping dictionaries
country_to_region = dict(zip(country_df['Short_name'], country_df['Region']))
country_to_country_name = dict(zip(country_df['Short_name'], country_df['Country']))
country_to_msci = dict(zip(country_df['Short_name'], country_df['MSCI']))

# Create a mapping of MSCI and Region to Category
msci_region_to_category = {
    ('Developed', 'AmericasandUK'): 'Developed_AmericasandUK',
    ('Developed', 'EMEA'): 'Developed_EMEA',
    ('Developed', 'APAC'): 'Developed_APAC',
    ('Emerging', 'Americas'): 'Emerging_Americas',
    ('Emerging', 'APACandEMEA'): 'Emerging_APACandEMEA'
}

# Iterate over asset classes and countries
for asset_class in asset_classes:
    for country in country_df['Short_name']:
        try:
            # Load ETF data by country if equity, else load by currency
            if asset_class == "class-equity":
                df = justetf_scraping.load_overview(asset_class=asset_class, country=country, local_country="GB")
            else:
                # For bonds, we need to specify the currency as well
                currency = country_df[country_df['Short_name'] == country]['Currency'].values[0]
                df = justetf_scraping.load_overview(asset_class=asset_class, instrument_currency=currency, local_country="GB")
                        
            # Add region and country information
            df['region'] = country_to_region[country]
            df['country'] = country_to_country_name[country]
            
            # Map MSCI and Region to Category
            msci = country_to_msci[country]
            region = country_to_region[country]
            region_category = msci_region_to_category.get((msci, region), "Unknown")
            df['region_category'] = region_category
            
            # Create filename based on asset class and category
            filename = f'justetf_{asset_class}_{region_category.lower()}.csv'.lower()
            
            # Append to existing file if it exists, otherwise create new
            if os.path.exists(filename):
                existing_df = pd.read_csv(filename)
                combined_df = pd.concat([existing_df, df], ignore_index=True).drop_duplicates(subset=['ticker'])
                combined_df.to_csv(filename, index=False)
            else:
                df.to_csv(filename, index=False)
            
            print(f"Processed {country} data for {asset_class}")
        except Exception as e:
            print(f"Error scraping {asset_class} for {country}: {e}")
```

## Alpha Vantage API Integration

Alpha Vantage provides detailed ETF profiles and historical time series data. We'll use environment variables to securely handle the API key.

```{code-cell} python
import requests
import os

# Get API key from environment variable
api_key = os.getenv('ALPHAVANTAGE_API_KEY', 'your_api_key_here')

def get_etf_profile(symbol):
    """Get ETF profile information from Alpha Vantage"""
    url = f'https://www.alphavantage.co/query?function=ETF_PROFILE&symbol={symbol}&apikey={api_key}'
    try:
        r = requests.get(url)
        data = r.json()
        return data
    except Exception as e:
        print(f"Error fetching ETF profile for {symbol}: {e}")
        return None

def get_etf_time_series(symbol, outputsize='full'):
    """Get ETF time series data from Alpha Vantage"""
    url = f'https://www.alphavantage.co/query?function=TIME_SERIES_DAILY_ADJUSTED&symbol={symbol}&outputsize={outputsize}&apikey={api_key}'
    try:
        r = requests.get(url)
        data = r.json()
        
        if 'Time Series (Daily)' in data:
            df = pd.DataFrame(data['Time Series (Daily)']).T
            df.index = pd.to_datetime(df.index)
            df['5. adjusted close'] = pd.to_numeric(df['5. adjusted close'], errors='coerce')
            return df
        else:
            print(f"No time series data found for {symbol}")
            return None
    except Exception as e:
        print(f"Error fetching time series for {symbol}: {e}")
        return None

def search_symbol(keywords):
    """Search for symbols using Alpha Vantage"""
    url = f'https://www.alphavantage.co/query?function=SYMBOL_SEARCH&keywords={keywords}&apikey={api_key}'
    try:
        r = requests.get(url)
        data = r.json()
        return data
    except Exception as e:
        print(f"Error searching for {keywords}: {e}")
        return None

# Example usage
if api_key != 'your_api_key_here':
    # Get ETF profile
    etf_profile = get_etf_profile('AUAD.LON')
    print("ETF Profile:", etf_profile)
    
    # Get time series data
    etf_data = get_etf_time_series('AUAD.LON')
    if etf_data is not None:
        print("Time Series Data:")
        print(etf_data.head())
    
    # Search for symbols
    search_results = search_symbol('DBC')
    print("Search Results:", search_results)
else:
    print("Please set your ALPHAVANTAGE_API_KEY environment variable to use Alpha Vantage features")
```

## OpenBB Terminal Integration

OpenBB provides additional market data and analysis capabilities.

```{code-cell} python
try:
    from openbb import obb
    
    # Get available indices
    available_indices = obb.index.available(provider="yfinance").to_df()
    print("Available Indices:")
    print(available_indices.head())
    
    # Get historical data for SPY ETF
    spy_data = obb.equity.price.historical(symbol="SPY", provider="yfinance").to_df()
    print("\nSPY Historical Data:")
    print(spy_data.head())
    
    # Search for ETF symbols
    search_results = obb.equity.search("IBZL").to_df()
    print("\nETF Search Results:")
    print(search_results)
    
except ImportError:
    print("OpenBB not available. Install with: pip install openbb")
except Exception as e:
    print(f"Error using OpenBB: {e}")
```

## Data Collection Summary

After running the data collection process, you can summarize the collected ETFs by region and asset class:

```{code-cell} python
import os

# Summarize collected data
summary_data = []

# Iterate over CSV files created by the scraping process
for csv_file in os.listdir('.'):
    if csv_file.startswith('justetf_class-') and csv_file.endswith('.csv'):
        try:
            df = pd.read_csv(csv_file)
            asset_class = csv_file.split('_')[1].replace('class-', '').title()
            region_category = csv_file.split('_')[2].replace('.csv', '').title()
            
            summary_data.append({
                'Asset Class': asset_class,
                'Category': region_category,
                'Number of ETFs': len(df),
                'Distributing ETFs': len(df[df['dividends'] == 'Distributing']) if 'dividends' in df.columns else 0
            })
        except Exception as e:
            print(f"Error processing {csv_file}: {e}")

if summary_data:
    summary_df = pd.DataFrame(summary_data)
    print("Data Collection Summary:")
    print(summary_df)
else:
    print("No ETF data files found. Run the data collection process first.")
```

## Data Quality Considerations

When working with multiple data sources, consider these important factors:

- **API Rate Limits**: Both Alpha Vantage and JustETF have rate limits. Implement appropriate delays between requests.
- **Data Consistency**: Different sources may have different ticker formats or data structures.
- **Missing Data**: Handle cases where ETFs are not available in certain regions or currencies.
- **Currency Conversion**: Consider currency effects when comparing ETFs from different regions.
- **Data Freshness**: Ensure you're working with up-to-date information, especially for fund characteristics.

## Security Best Practices

- **Never commit API keys**: Always use environment variables or secure configuration files.
- **Rotate API keys regularly**: Especially if they're used in shared environments.
- **Monitor API usage**: Keep track of your API call limits and costs.

## Next Steps

Once you have collected and organized your ETF data:

1. **Data Validation**: Verify the completeness and accuracy of collected data
2. **Data Storage**: Consider using a database for larger datasets
3. **ETF Screening**: Proceed to the next chapter to learn how to filter and evaluate ETFs
4. **Portfolio Construction**: Use the screened ETFs to build optimized portfolios

The comprehensive data collection framework established in this chapter provides the foundation for systematic ETF portfolio construction and management.
