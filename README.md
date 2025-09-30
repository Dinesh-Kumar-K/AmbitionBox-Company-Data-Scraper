```markdown
# AmbitionBox Company Data Scraper

This document outlines the process of scraping company information from AmbitionBox.com. It covers initial data fetching, asynchronous processing, and leveraging a scraping API for more detailed information.

## Setup and Imports

First, we import the necessary Python libraries for web scraping, data manipulation, and asynchronous programming.

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import urllib.parse
import aiohttp
import asyncio
import nest_asyncio
import time
from concurrent.futures import ThreadPoolExecutor
from tqdm import tqdm
import re  # Imported for the clean_string function
```

## Initial URL Generation

We create a list of URLs to scrape from AmbitionBox, targeting different pages for company listings.

```python
list_url=[]
for i in range(1,10):
    link=f'https://www.ambitionbox.com/list-of-companies?industries=analytics-and-kpo,edtech,bpo,fintech,media-and-entertainment,gaming,design,emerging-technologies,nbfc,computer-software,analytics,graphics-and-animation,telecom,software-product&sortBy=popular&page={i}'
    list_url.append(link)
```

## Headers for Requests

A user-agent header is defined to mimic a web browser.

```python
header={'User-agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36'}
```

## Data Storage Lists

Lists are initialized to store the scraped company names, descriptions, links, and their corresponding page numbers and status codes.

```python
page_num_li=[]
company_name_li=[]
company_desc_li=[]
company_link_list=[]
status_codes_list=[]
```

## Basic Scraping Function (`ambitionbox`)

This function fetches the content of a given URL, parses it using BeautifulSoup, and extracts company names, descriptions, and links from the company cards. It also records the HTTP status code.

```python
def ambitionbox(url):
  header={'User-agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36'}

  res=requests.get(url,headers=header)
  print(res) # Prints the status of the request

  soup=BeautifulSoup(res.content,'lxml')

  fin_tag=soup.find_all('div',class_="companyCardWrapper")

  for i in fin_tag:
    company_name=i.find('h2',class_='companyCardWrapper__companyName').text.strip()
    company_desc=i.find('span',class_='companyCardWrapper__interLinking').text.strip()
    company_link=i.find('a',class_="companyCardWrapper__companyName").get('href')

    company_link_list.append(company_link)
    company_name_li.append(company_name)
    company_desc_li.append(company_desc)
    status_codes_list.append(res.status_code)
```

## Example of Basic Scraping

A single URL is tested with the `ambitionbox` function.

```python
ambitionbox('https://www.ambitionbox.com/salaries/integrated-capital-services-salaries')
```

## Displaying Scraped Links

The collected company links are displayed.

```python
company_link_list
```

## Asynchronous Scraping Setup (Initial Attempt)

This section sets up an asynchronous approach to scrape multiple URLs concurrently. Note that the `ambitionbox` function used here would need to be adapted for `aiohttp` (as suggested by the commented-out `async with session.get(...)`).

```python
async def main():
  tasks=[]
  async with aiohttp.ClientSession() as session:
    for url in list_url:
      # Assuming ambitionbox function is modified to be async and take session
      # task = asyncio.ensure_future(ambitionbox(url,session))
      # tasks.append(task)
      pass # Placeholder for actual async task creation

    # for task in asyncio.as_completed(tasks):
    #     await task
    pass # Placeholder for waiting for tasks to complete
```

## Running the Asynchronous Scraper

The `nest_asyncio` library is applied to allow running asyncio within environments that might already have a running event loop (like Jupyter notebooks). The `main` function is then executed.

```python
if __name__ == "__main__":
    t = time.perf_counter()

    nest_asyncio.apply()
    # asyncio.run(main()) # Uncomment to run the async main function

    t2 = time.perf_counter()
    print(f"Asynchronous scraping took: {t2-t:.2f} seconds")
```

## Threaded Scraping

This section demonstrates using `ThreadPoolExecutor` for concurrent execution of the `ambitionbox` function across a subset of the generated URLs. `tqdm` is used for a progress bar.

```python
list_url_mo=list_url[:10] # Using a smaller subset for demonstration

with tqdm(total=len(list_url_mo)) as progress:
  with ThreadPoolExecutor(max_workers=5) as exe:
      for __ in exe.map(ambitionbox,list_url_mo):
        progress.update()
        # This line might cause issues if company_name_li is empty during initial runs
        # progress.set_description(f'Company Name {str(company_name_li[-1])}')
```

## DataFrame Creation

The scraped data is organized into a Pandas DataFrame.

```python
mdf=pd.DataFrame(zip(company_name_li,company_link_list,company_desc_li),columns=['Company_Name','Company_Url','Company_Desc'])
```

## Displaying the DataFrame

The initial DataFrame is shown.

```python
mdf
```

## Enhanced Scraping with a Scraper API

This section introduces a new approach using a scraping API (`scraper.api.airforce`) to fetch more structured data, likely from a different endpoint. It also involves mounting Google Drive for input/output.

### Mounting Google Drive

```python
from google.colab import drive
drive.mount('/content/drive')
```

### Revised `ambitionbox` Function

This version of `ambitionbox` uses the scraping API and parses JSON data embedded within a script tag.

```python
def ambitionbox(url):
    header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36'}
    # The URL is constructed to fetch data for a specific company overview page
    res = requests.get(f'https://scraper.api.airforce/scrape?url=https://www.ambitionbox.com/overview/{url}-overview', headers=header)
    soup = BeautifulSoup(res.content, 'lxml')
    div_tag = soup.find('script',id='__NEXT_DATA__')
    # Evaluates the stringified JSON data, converting null, true, false to Python equivalents
    json_data = eval(str(div_tag.string).replace('null','None').replace('true','True').replace('false','False'))
    json_data['url_id'] = url # Adds the company identifier to the data
    return json_data
```

### Testing the Revised Function

The new `ambitionbox` function is tested with a sample company identifier.

```python
ambitionbox('tcs')
```

### Loading Input Data

An Excel file containing company URLs is loaded.

```python
input_df = pd.read_excel('/content/drive/MyDrive/Company Projects/AmbitionBox/data/BPO Companies in India Fomm Ambitionbox.xlsx')
```

### Extracting `url_id`

A `url_id` is extracted from the company URLs for use with the scraping API.

```python
input_df['url_id'] = input_df['Urls'].apply(lambda x: str(x).split('/')[-1].split('-jobs-cmp')[0])
```

### Displaying the Input DataFrame

The DataFrame with the extracted `url_id` is shown.

```python
input_df
```

### Concurrent Scraping with Revised Function

The revised `ambitionbox` function is used with `ThreadPoolExecutor` to scrape data for all companies in the input DataFrame. The results are collected in `collect_result`.

```python
collect_result = []
with tqdm(total=len(input_df)) as progress:
  with ThreadPoolExecutor(max_workers=2) as exe:
      for i in exe.map(ambitionbox,input_df['url_id']):
        progress.update(1)
        collect_result.append(i)
```

### Merging DataFrames

The scraped JSON data is normalized into a DataFrame (`collect_result_df`) and then merged with the original input DataFrame (`input_df`) based on `url_id`.

```python
collect_result_df = pd.json_normalize(collect_result)
final_df = pd.merge(input_df, collect_result_df, on='url_id', how='left')
```

### Cleaning Company Description

A helper function `clean_string` is defined to remove illegal characters from strings, making them compatible with Excel.

```python
def clean_string(text):
    """Removes illegal characters from a string for Excel compatibility."""
    # Define the regex to match illegal characters based on openpyxl's definition
    ILLEGAL_CHARACTERS_RE = re.compile(r'[\000-\010]|[\013-\014]|[\016-\037]')
    # Replace illegal characters with an empty string
    return ILLEGAL_CHARACTERS_RE.sub('', text)

# Apply the cleaning function to the 'Company_Desc' column of your DataFrame
# Assuming 'Company_Desc' exists in the merged DataFrame and needs cleaning
if 'Company_Desc' in final_df.columns:
    final_df['Company_Desc'] = final_df['Company_Desc'].apply(clean_string)
else:
    print("Warning: 'Company_Desc' column not found for cleaning.")
```

### Renaming Columns

Specific columns are renamed for clarity, mapping them from their nested JSON structure to more readable names.

```python
# The actual column names in the DataFrame might differ based on the JSON structure.
# This assumes a structure like 'props.pageProps.companyMetaInformation.primaryIndustry'
try:
    final_df.rename(columns={
        'props.pageProps.companyMetaInformation.primaryIndustry':'primary_industry',
        'props.pageProps.companyMetaInformation.secondaryIndustry':'secondary_industry'
    }, inplace=True)
    print("Columns renamed successfully.")
except KeyError as e:
    print(f"Error renaming columns: {e}. Check if the original column names exist.")

```

### Listing Columns

All columns in the processed DataFrame are listed.

```python
list(final_df.columns)
```

### Installing `xlsxwriter`

The `xlsxwriter` engine is installed for enhanced Excel file writing.

```python
pip install xlsxwriter
```

### Accessing and Processing Industry Data

The `secondary_industry` column is inspected.

```python
final_df['secondary_industry']
```

New columns `primary_industry1` and `secondary_industry1` are created by extracting the 'name' from nested list/dictionary structures within the `primary_industry` and `secondary_industry` columns, respectively. This is a common step when dealing with scraped JSON data containing lists of entities.

```python
# Extracting primary industry name
final_df['primary_industry1'] = final_df['primary_industry'].apply(
    lambda x: x[0]['name'] if isinstance(x, list) and len(x) > 0 and isinstance(x[0], dict) and 'name' in x[0] else None
)

# Extracting all secondary industry names as a list
final_df['secondary_industry1'] = final_df['secondary_industry'].apply(
    lambda x: [i['name'] for i in x] if isinstance(x, list) else None
)
```

### Displaying Processed Industry Data

The newly created `secondary_industry1` column is shown.

```python
final_df['secondary_industry1']
```

### Empty DataFrame Selection (Example)

This line attempts to select columns from an empty list, which would result in an empty DataFrame. This is likely for demonstration or debugging.

```python
final_df[[]]
```

### Saving to Excel

The final processed DataFrame is saved to an Excel file using the `xlsxwriter` engine.

```python
# Saving the DataFrame to an Excel file
output_excel_path = "/content/drive/MyDrive/Company Projects/AmbitionBox/data/BPO Companies in India Fomm Ambitionbox Output 03-12-2024.xlsx"

try:
    with pd.ExcelWriter(output_excel_path, engine='xlsxwriter') as out_file:
        final_df.to_excel(out_file, index=False)
    print(f"Successfully saved the DataFrame to {output_excel_path}")
except Exception as e:
    print(f"Error saving DataFrame to Excel: {e}")

```

### Alternative Excel Saving Method

Another way to save to Excel is demonstrated, ensuring the `index` is not written to the file.

```python
# Alternative method for saving to Excel
# out_file = pd.ExcelWriter('/content/drive/MyDrive/Company Projects/AmbitionBox/data/BPO Companies in India Fomm Ambitionbox Output 03-12-2024.xlsx')
# final_df.to_excel(out_file, index=False)
# out_file.close() # It's important to close the writer object
```
```