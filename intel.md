#### Intel Applied ML Project

- Purpose: design and implement an AI bot to scan through different websites (text, video, …) to capture the competitive analysis data for Intel latest CPU processor and other competitors’

- Detailed Data to capture:
  1) the system information under testing
  2) benchmarks and scores (please see the example tab in the excel file and watch the video)
  
- What to send back to us: all the code of your AI bot and the filled-in excel file (we will run your code and check the excel file for data accuracy and completion)

- Bonus: Auto-summarized the comparison across different benchmarks, give recommendations on where Intel is doing well and where competitors are doing well.
  
- Timeline: Ideally within one week

- Note: the solution is not needed to be perfect. Just try the best that you can figure out Follow-up: we will pick top 3 developers and follow up with more interviews on details of your implementations, then talk about roles, expectations, culture, compensation, etc.

**Work instructions:**

- Find the site and look for the new CPU reviews and find the review for 13th Gen. 
Log the header information at the top of the sheet: Title, Author, URL, test system specs, et .
Skim/read through the review to find the benchmark da

.  The primary data we are interested in  for:

- Core i9-13900K
- 
Ryzen 9 7950
- Core i7-13700K
- Ryzen 7 7800X3D
- Core i9-14900K
- Ryzen 5 7600X
- Core i5-14600K

Do not bother recording data for other processors.

Enter the data in columns E thru X noting there are columns for different memory configs. If there is not already a row that has the correct workload / benchmark, then copy and insert a new row and change the benchmark name (so the formulas copy). Make sure to note H/L for higher or lower is better. Take your best guess at category, etc., if unsure, highlight the cell in yellow to be checked later.
Collect any notable quotes on the Quotes tab, including the website, URL and author as well. Select another publication that is available and repeat.

Notes:

Some sites do video reviews and you will need to watch the videos and pause the clips to record the numbers. Gamers Nexus, Linus Tech Tips, Hardware Canucks and Hardware Unboxed are likely to be reviews on YouTube. Some sites will have slideshows embedded on the page and you will need to cycle through all of the images, each of which will have a graph to find all of the data.ph to find all of the dat

In summary: 

- Have AI bot to search across the entire web, find related websites or videos
- Have AI bot to scan through all the above sources and capture competitive data, put into Excel file, summarize ita.


```python
import os
import pandas as pd
import numpy as np
from bs4 import BeautifulSoup
import requests
import json
from datetime import datetime
import time
from googlesearch import search
# llm summary
from langchain.chat_models import ChatOpenAI
import transformers
from transformers import BertTokenizer, BertModel, pipeline
```


```python
os.environ['OPENAI_API_KEY'] = "xxx" # Replace with your OpenAI API token
```

**Search the web**


```python
# Define the query with site restriction
query = ["intel core i9 13900k reviews", "intel core i9 13900k reviews site:youtube.com"]

web = []
videos = []

# Perform the search and iterate through the results
for q in query:
    
    for result in search(q, num=5, stop=5, pause=2):
        if 'youtube' in q:
            videos.append(result)
        else:
            web.append(result)
```


```python
f"Web results: {web}"
```




    "Web results: ['https://www.pcmag.com/reviews/intel-core-i9-13900k', 'https://www.pcmag.com/reviews/intel-core-i9-13900k#testing-setup', 'https://www.pcmag.com/reviews/intel-core-i9-13900k#core-i9-13900k-cpu-performance', 'https://www.pcmag.com/reviews/intel-core-i9-13900k#core-i9-13900k-gaming-performance', 'https://www.theverge.com/23410428/intel-core-i9-13900k-review']"




```python
f"Video results: {videos}"
```




    "Video results: ['https://www.youtube.com/watch?v=yWw6q6fRnnI', 'https://www.youtube.com/watch?v=o6fLK9coBYc', 'https://www.youtube.com/watch?v=ua2FWVz_p5o', 'https://www.youtube.com/watch?v=gj9hZ51w198', 'https://www.youtube.com/watch?v=SvDNqSh-bss']"




```python
class IntelScraper():

    # Constructor to save website which we will pass while calling Scraper class
    def __init__(self):

        self.headers = {
            'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
            'accept-encoding': 'gzip, deflate, br',
            'accept-language': 'en-US,en;q=0.8',
            'upgrade-insecure-requests': '1',
            'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'
        }

    def get_soup(self, url):
        
        html_page = requests.get(url, self.headers)
        
        soup = BeautifulSoup(html_page.content, 'html.parser')
        
        return soup, html_page.url

    def summarize(self, url, meta, bench):
        # Generate LLM based summary from article body
        soup, web_url = self.get_soup(url)

        # Find the main article content
        article_text = ""

        for element in soup.find_all('p'):
            article_text += element.get_text() + "\n"  # Add newline for readability

        # LLM for summarization
        if article_text and meta and bench:

            # Create a prompt string template
            prompt = f"summarize: {article_text[0:10000]} {meta} {bench}"
            
            try:
        
                llm = ChatOpenAI(
                                 model = 'gpt-3.5-turbo-16k', 
                                 request_timeout=120
                                )
    
                # pass the snippet to get produce summary, limit to max tokens allowed for the model
                llm_summary = llm.invoke(prompt)
    
                print('Summary length:', len(llm_summary.content))

                return llm_summary.content
        
            except Exception as e:
                print('Exception', e)
                return np.nan

    # This is the main function that will scrape all data
    def scrape(self, url):

        metadata = []
        table = None
        data_list = None
        
        title = None
        date_val = None
        author = None
        org_name = None
        product_name = None
        
        #for url in web:
            
        print(f'Scraping: {url}')

        soup, web_url = self.get_soup(url)

        """
        Get metadata
        """
        title = soup.find('h1').text.strip()
        print('Title:', title)
        
        script_tags = soup.find_all('script', {'type': 'application/ld+json'})

        for tag in script_tags:
            # parse json string to dict
            data = json.loads(tag.string)
            #print("script", data)
    
            # Format JSON data with an indentation of 4 spaces
            #json_data = json.dumps(data, indent=4)

            if data.get("@type") == "Organization":
                org_name = data["name"]
                print("Organization:", org_name)
            
            if data.get("@type") == "Product":
               product_name = data['name']
               print('Product:', product_name)

            if data.get('review'):
                r = data.get('review')

                if 'author' in r:
                    a = r.get('author')
                    author = a[0]['name']
                    print('Author:', a[0]['name'])

                if 'pcmag' in url:
                    
                    if 'datePublished' in r:
                        dt = r.get('datePublished')
                        # Parse the date string into a datetime object
                        dateObj = datetime.strptime(dt, '%Y-%m-%dT%H:%M:%S%z')
                        date_val = str(dateObj.date())
                        print('Date:', date_val)

            if 'theverge' in url:
                if 'datePublished' in data:
                    dt = data['datePublished']
                    # Parse the date string into a datetime object
                    #dateObj = datetime.strptime(dt, '%Y-%m-%dT%H:%M:%S%z')
                    dateObj = datetime.strptime(dt, '%Y-%m-%dT%H:%M:%S.%fZ')
                    date_val = str(dateObj.date())
                    print('Date:', date_val)
                        
        """
        Get specifications
        """
        if 'pcmag' in url:
            table = soup.find('table', class_='default w-full table-fixed text-lg')
        elif 'verge' in url:
            data_list = soup.find("ul", class_="duet--article--unordered-list")

        if table or data_list:
            count = 0
            specs = {}

            if 'pcmag' in url:
                # skip headers
                for row in table.find_all('tr')[1:]:
                    #print(count, row)
                    #count += 1
                    name = row.find('td', class_='hyphens').text.strip()
                    value = row.find('td', class_='md:text-left').text.strip()
                    # Append to dict
                    specs[name] = value
                    
            elif 'verge' in url:
                # Find the <ul> tag with the specified class and index value
                ul_tag = soup.find_all('ul', class_='duet--article--unordered-list my-20 list-disc pl-18 marker:text-blurple/100 selection:bg-franklin-20 dark:text-white dark:selection:bg-blurple [&_a:hover]:shadow-highlight-franklin dark:[&_a:hover]:shadow-highlight-franklin [&_a]:shadow-underline-black dark:[&_a]:shadow-underline-white')[2]

                for tag in ul_tag:
                    # Split the text by colon (:)
                    text = tag.text.strip().split(": ")
                    
                    # Extract key
                    key = text[0].strip()
                    # Extract value
                    value = text[1] if len(text) > 1 else ""
                    # Append to dict
                    specs[key] = value

        #print(url, specs)
        
        """
        Get benchmarks
        """
        bench = {}
        
        if 'pcmag' in url:
            # iframe code is not directly accessible, so reading it from a .txt file
            with open("PCMag_script.txt", "r") as file:
            # Read the entire file content into a string
              text = file.read()
            
            # Remove unnecessary characters from the js code
            js_code = text.replace('\n', '').replace('\t', '').replace(' ', '')
            # Extract
            json_str = js_code[js_code.find('{'):js_code.rfind('}') + 1]

            # Convert to a Python dictionary
            data = json.loads(json_str)

            # get labels
            benchmark_labels = data['elements']['content']['content']['entities']['221a4a5b-d6f4-4c7a-b26c-a02911a7efa3']['props']['chartData']['sheetnames']
            
            # get data
            benchmark_data = data['elements']['content']['content']['entities']['221a4a5b-d6f4-4c7a-b26c-a02911a7efa3']['props']['chartData']['data']

            # Filter strings
            processor = ['Intel','AMD']

            # Iterate over data and create a list of lists
            data = []

            for i,l in enumerate(benchmark_data):        
                lf = [row for row in l if any(word in row[0] for word in ["Intel", "AMD"])] # row[0] for check the first element which is the processor. 
                data.append(lf)
                
            for l in data:
                for e in l:
                    #print('Element', e)
                    key = e[0]  
                    values = e[1:] 
                
                    # Check if the key exists
                    if key not in bench:
                        bench[key] = [values] 
                    else:
                        # Otherwise append the values
                        bench[key].append(values)

        elif 'verge' in url:
            table = soup.find('table', class_='font-sans text-xs relative w-full table-auto text-center')

            # Extract table headers
            headers = [th.text for th in soup.find_all('th')]
            # skip 'benchmark' label
            benchmark_labels = headers[1:5]

            data = []
            
            # benchmark list
            benchmark_lst = [
                             'Geekbench 5 single-thread',
                             'Geekbench 5 multithread',
                             'Cinebench R23 single-thread',
                             'Cinebench R23 multithread',
                             'Blender Fishy Cat',
                             'PugetBench for Premiere Pro',
                             'PugetBench for Photoshop',
                             '3DMark Time Spy CPU',
                             'Metro Exodus (ultra / high)',
                             'Shadow of the Tomb Raider',
                             'Gears 5',
                             'Assassin\'s Creed Valhalla',
                             'Watch Dogs: Legion',
                             'Cyberpunk 2077'
                            ]
            
            # Extract table data
            for row in table.find_all('tr'):
                
                cells = row.find_all('td')
            
                for cell in cells:
                    
                    # # Extract benchmark name (key)
                    text = cell.text.strip()
            
                    if text in benchmark_lst:
                        benchmark_name = text
                    else:
                        benchmark_value = text
            
                    if benchmark_name not in bench:
                        bench[benchmark_name] = []
                    else:
                        bench[benchmark_name].append(benchmark_value)

        """
        Construct dicts and save as pandas df
        """ 
        if org_name == None:
            if 'pcmag' in url:
                org_name = 'pcmag'
            elif 'verge' in url:
                org_name = 'theverge'
                
        if title and org_name and date_val and author and product_name and specs:
            
           dic = {
                        'Title': title,
                        'Organization': org_name,
                        'Link': url,
                        'Date': date_val,
                        'Author': author,
                        'Product': product_name,
                 }
                
           # Merge dicts
           dic.update(specs)
            
           metadata.append(dic)

           print(f"metadata: {metadata}, benchmark: {bench}, org: {org_name}")
    
           # save to pandas dataframe and excel
           self.save_data(url, metadata, bench, org_name, benchmark_labels)
    
        print('\n-------------------- SCRAPING COMPLETE --------------------\n')

        
    def save_data(self, url, metadata, benchmark, name, labels=None):

        df_meta = pd.DataFrame.from_dict(metadata).transpose()

        if 'pcmag' in name.lower():
            df_bench = pd.DataFrame.from_dict(benchmark, orient='index', columns=labels)
        else:
            df_bench = pd.DataFrame.from_dict(benchmark, orient="index", columns=labels)

        # Generate summary - call summarize() which invokes LLMs for summarization
        summary = self.summarize(url, metadata, benchmark)

        df_sum = pd.DataFrame({'summary': [summary]})
        
        # # save to excel
        filename = f"data_{name}.xlsx"

        # create a excel writer object
        with pd.ExcelWriter(filename) as writer:
   
            # use to_excel function and specify the sheet_name and index 
            df_meta.to_excel(writer, sheet_name="Specs")
            df_bench.to_excel(writer, sheet_name="Benchmark")
            df_sum.to_excel(writer, sheet_name="Summary")
```


```python
# URLs to scrape
url = web[0]
#url = videos[0]
url
```




    'https://www.pcmag.com/reviews/intel-core-i9-13900k'




```python
# pcmag
url = web[0] 
# Creating class object
IntelScraper1 = IntelScraper()

# Calling functions that are created above:
IntelScraper1.scrape(url)
```

    Scraping: https://www.pcmag.com/reviews/intel-core-i9-13900k
    Title: Intel Core i9-13900K Review
    Organization: PCMag
    Product: Intel Core i9-13900K
    Author: Michael Justin Allen Sexton
    Date: 2022-10-20
    metadata: [{'Title': 'Intel Core i9-13900K Review', 'Organization': 'PCMag', 'Link': 'https://www.pcmag.com/reviews/intel-core-i9-13900k', 'Date': '2022-10-20', 'Author': 'Michael Justin Allen Sexton', 'Product': 'Intel Core i9-13900K', 'Core Count': '24', 'Thread Count': '32', 'Base Clock Frequency': '3 GHz', 'Maximum Boost Clock': '5.8 GHz', 'Unlocked Multiplier?': '', 'Socket Compatibility': 'Intel LGA 1700', 'Lithography': '7 nm', 'L3 Cache Amount': '36 MB', 'Thermal Design Power (TDP) Rating': '253 watts', 'Integrated Graphics': 'Intel UHD Graphics 770', 'Integrated Graphics Base Clock': '1.65 MHz', 'Bundled Cooler': 'None'}], benchmark: {'IntelCorei9-13900K': [['39022', '2275'], ['1489', '1189'], ['172'], ['63'], ['306', '20']], 'AMDRyzen97950X': [['35063', '2020'], ['1325', '1209'], ['199'], ['70'], ['360', '22']], 'AMDRyzen77700X': [['19083', '2001'], ['1381', '975'], ['254'], ['127'], ['349', '38']], 'AMDRyzen95950X': [['24724', '1626'], ['1017', '986'], ['267'], ['97'], ['417', '30']], 'AMDRyzen75800X3D': [['14372', '1478'], ['1031', '857'], ['304'], ['168'], ['458', '50']], 'AMDRyzen75700X': [['14685', '1548'], ['1011', '849'], ['322'], ['174'], ['449', '52']], 'IntelCorei9-12900K': [['27131', '1987'], ['1320', '1059'], ['208'], ['92'], ['353', '29']], 'IntelCorei7-12700K': [['22797', '1940'], ['1272', '1002'], ['231'], ['110'], ['366', '35']], 'IntelCorei5-12600K': [['17411', '1917'], ['1260', '899'], ['276'], ['147'], ['377', '45']], 'IntelCorei9-11900K': [['14987', '1607'], ['894', '785'], ['329'], ['99'], ['439', '57']], 'IntelCorei9-10900K': [['14929', '1381'], ['848', '802'], ['345'], ['137'], ['482', '48']]}, org: PCMag
    Exception Error code: 401 - {'error': {'message': 'Incorrect API key provided: xxx. You can find your API key at https://platform.openai.com/account/api-keys.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_api_key'}}
    
    -------------------- SCRAPING COMPLETE --------------------
    


 


```python
# theverge
url = web[4] 
# Creating class object
IntelScraper2 = IntelScraper()

# Calling functions that are created above:
IntelScraper2.scrape(url)
```

    Scraping: https://www.theverge.com/23410428/intel-core-i9-13900k-review
    Title: Intel Core i9-13900K review: an AMD Zen 4 beater
    Date: 2022-10-20
    Product: Core i9-13900K
    Author: Tom Warren
    metadata: [{'Title': 'Intel Core i9-13900K review: an AMD Zen 4 beater', 'Organization': 'theverge', 'Link': 'https://www.theverge.com/23410428/intel-core-i9-13900k-review', 'Date': '2022-10-20', 'Author': 'Tom Warren', 'Product': 'Core i9-13900K', 'CPU': 'Intel Core i9-13900K', 'CPU cooler': 'Corsair H150 Elite LCD', 'Motherboard': 'MSI MAG Z690 Carbon Wi-Fi', 'RAM': '32GB Corsair Dominator Platinum DDR5 6600', 'GPU': 'Nvidia RTX 3080 Ti Founders Edition', 'Storage': 'Western Digital SN850 1TB', 'Case': 'Corsair Crystal 570X', 'PSU': 'Corsair HX1000W'}], benchmark: {'Geekbench 5 single-thread': ['2202', '2143', '2202', '1946'], 'Geekbench 5 multithread': ['24207', '22492', '19742', '17963'], 'Cinebench R23 single-thread': ['2169', '1941', '1989', '1943'], 'Cinebench R23 multithread': ['38704', '34814', '28818', '26602'], 'Blender Fishy Cat': ['00:12.96', '00:12.52', '00:13.77', '00:14.72'], 'PugetBench for Premiere Pro': ['1227', '1148', '1075', '1150'], 'PugetBench for Photoshop': ['1517', '1497', '1440', '1351'], '3DMark Time Spy CPU': ['19205', '18650', '18323', '18927'], 'Metro Exodus (ultra / high)': ['150fps', '147fps', '146fps', '142fps'], 'Shadow of the Tomb Raider': ['244fps', '230fps', '227fps', '217fps'], 'Gears 5': ['187fps', '167fps', '169fps', '174fps'], "Assassin's Creed Valhalla": ['138fps', '138fps', '114fps', '112fps'], 'Watch Dogs: Legion': ['126fps', '123fps', '116fps', '119fps'], 'Cyberpunk 2077': ['144fps', '144fps', '138fps', '138fps']}, org: theverge
    Exception Error code: 401 - {'error': {'message': 'Incorrect API key provided: xxx. You can find your API key at https://platform.openai.com/account/api-keys.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_api_key'}}
    
    -------------------- SCRAPING COMPLETE --------------------
    



```python

```


```python

```


```python

```


```python

```


```python

```


```python

```


```python

```

**Testing Code below:** (Feel free to ignore)


```python
metadata = [{'Title': 'Intel Core i9-13900K Review', 'Organization': 'PCMag', 'Link': 'https://www.pcmag.com/reviews/intel-core-i9-13900k', 'Date': '2022-10-20', 'Author': 'Michael Justin Allen Sexton', 'Product': 'Intel Core i9-13900K', 'Core Count': '24', 'Thread Count': '32', 'Base Clock Frequency': '3 GHz', 'Maximum Boost Clock': '5.8 GHz', 'Unlocked Multiplier?': '', 'Socket Compatibility': 'Intel LGA 1700', 'Lithography': '7 nm', 'L3 Cache Amount': '36 MB', 'Thermal Design Power (TDP) Rating': '253 watts', 'Integrated Graphics': 'Intel UHD Graphics 770', 'Integrated Graphics Base Clock': '1.65 MHz', 'Bundled Cooler': 'None'}]
metadata
```




    [{'Title': 'Intel Core i9-13900K Review',
      'Organization': 'PCMag',
      'Link': 'https://www.pcmag.com/reviews/intel-core-i9-13900k',
      'Date': '2022-10-20',
      'Author': 'Michael Justin Allen Sexton',
      'Product': 'Intel Core i9-13900K',
      'Core Count': '24',
      'Thread Count': '32',
      'Base Clock Frequency': '3 GHz',
      'Maximum Boost Clock': '5.8 GHz',
      'Unlocked Multiplier?': '',
      'Socket Compatibility': 'Intel LGA 1700',
      'Lithography': '7 nm',
      'L3 Cache Amount': '36 MB',
      'Thermal Design Power (TDP) Rating': '253 watts',
      'Integrated Graphics': 'Intel UHD Graphics 770',
      'Integrated Graphics Base Clock': '1.65 MHz',
      'Bundled Cooler': 'None'}]




```python
df = pd.DataFrame.from_dict(metadata).transpose()
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Title</th>
      <td>Intel Core i9-13900K Review</td>
    </tr>
    <tr>
      <th>Organization</th>
      <td>PCMag</td>
    </tr>
    <tr>
      <th>Link</th>
      <td>https://www.pcmag.com/reviews/intel-core-i9-13...</td>
    </tr>
    <tr>
      <th>Date</th>
      <td>2022-10-20</td>
    </tr>
    <tr>
      <th>Author</th>
      <td>Michael Justin Allen Sexton</td>
    </tr>
    <tr>
      <th>Product</th>
      <td>Intel Core i9-13900K</td>
    </tr>
    <tr>
      <th>Core Count</th>
      <td>24</td>
    </tr>
    <tr>
      <th>Thread Count</th>
      <td>32</td>
    </tr>
    <tr>
      <th>Base Clock Frequency</th>
      <td>3 GHz</td>
    </tr>
    <tr>
      <th>Maximum Boost Clock</th>
      <td>5.8 GHz</td>
    </tr>
    <tr>
      <th>Unlocked Multiplier?</th>
      <td></td>
    </tr>
    <tr>
      <th>Socket Compatibility</th>
      <td>Intel LGA 1700</td>
    </tr>
    <tr>
      <th>Lithography</th>
      <td>7 nm</td>
    </tr>
    <tr>
      <th>L3 Cache Amount</th>
      <td>36 MB</td>
    </tr>
    <tr>
      <th>Thermal Design Power (TDP) Rating</th>
      <td>253 watts</td>
    </tr>
    <tr>
      <th>Integrated Graphics</th>
      <td>Intel UHD Graphics 770</td>
    </tr>
    <tr>
      <th>Integrated Graphics Base Clock</th>
      <td>1.65 MHz</td>
    </tr>
    <tr>
      <th>Bundled Cooler</th>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>




```python

```


```python

```


```python

```


```python
html_page = requests.get(url)
        
soup = BeautifulSoup(html_page.content, 'html.parser')
```


```python
table = soup.find('table', class_='font-sans text-xs relative w-full table-auto text-center')
```


```python
# Extract table headers
headers = [th.text for th in soup.find_all('th')]
# skip 'benchmark' label
headers[1:5]
```




    ['Intel Core i9-13900K', 'AMD 7950X', 'AMD 7900X', 'Intel Core i9-12900K']




```python
# Initialize dict
data_dict = {}

data_values = []

# benchmark list
benchmark_lst = [
                 'Geekbench 5 single-thread',
                 'Geekbench 5 multithread',
                 'Cinebench R23 single-thread',
                 'Cinebench R23 multithread',
                 'Blender Fishy Cat',
                 'PugetBench for Premiere Pro',
                 'PugetBench for Photoshop',
                 '3DMark Time Spy CPU',
                 'Metro Exodus (ultra / high)',
                 'Shadow of the Tomb Raider',
                 'Gears 5',
                 'Assassin\'s Creed Valhalla',
                 'Watch Dogs: Legion',
                 'Cyberpunk 2077'
                ]

# Extract table data
for row in table.find_all('tr'):
    
    cells = row.find_all('td')

    for cell in cells:
        
        # # Extract benchmark name (key)
        text = cell.text.strip()

        if text in benchmark_lst:
            benchmark_name = text
        else:
            benchmark_value = text

        if benchmark_name not in data_dict:
            data_dict[benchmark_name] = []
        else:
            data_dict[benchmark_name].append(benchmark_value)
            
        # # Extract benchmark value (value)
        #benchmark_value = cell.text.strip()
        #print('benchmark_value', benchmark_value)

        # if benchmark_name not in data_dict:
        #     data_dict[benchmark_name] = [benchmark_value]
        # else:
        #     data_dict[benchmark_name].append(benchmark_value)
```


```python
data_dict
```




    {'Geekbench 5 single-thread': ['2202', '2143', '2202', '1946'],
     'Geekbench 5 multithread': ['24207', '22492', '19742', '17963'],
     'Cinebench R23 single-thread': ['2169', '1941', '1989', '1943'],
     'Cinebench R23 multithread': ['38704', '34814', '28818', '26602'],
     'Blender Fishy Cat': ['00:12.96', '00:12.52', '00:13.77', '00:14.72'],
     'PugetBench for Premiere Pro': ['1227', '1148', '1075', '1150'],
     'PugetBench for Photoshop': ['1517', '1497', '1440', '1351'],
     '3DMark Time Spy CPU': ['19205', '18650', '18323', '18927'],
     'Metro Exodus (ultra / high)': ['150fps', '147fps', '146fps', '142fps'],
     'Shadow of the Tomb Raider': ['244fps', '230fps', '227fps', '217fps'],
     'Gears 5': ['187fps', '167fps', '169fps', '174fps'],
     "Assassin's Creed Valhalla": ['138fps', '138fps', '114fps', '112fps'],
     'Watch Dogs: Legion': ['126fps', '123fps', '116fps', '119fps'],
     'Cyberpunk 2077': ['144fps', '144fps', '138fps', '138fps']}




```python
pd.DataFrame.from_dict(data_dict, orient="index", columns=headers[1:5])
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Intel Core i9-13900K</th>
      <th>AMD 7950X</th>
      <th>AMD 7900X</th>
      <th>Intel Core i9-12900K</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Geekbench 5 single-thread</th>
      <td>2202</td>
      <td>2143</td>
      <td>2202</td>
      <td>1946</td>
    </tr>
    <tr>
      <th>Geekbench 5 multithread</th>
      <td>24207</td>
      <td>22492</td>
      <td>19742</td>
      <td>17963</td>
    </tr>
    <tr>
      <th>Cinebench R23 single-thread</th>
      <td>2169</td>
      <td>1941</td>
      <td>1989</td>
      <td>1943</td>
    </tr>
    <tr>
      <th>Cinebench R23 multithread</th>
      <td>38704</td>
      <td>34814</td>
      <td>28818</td>
      <td>26602</td>
    </tr>
    <tr>
      <th>Blender Fishy Cat</th>
      <td>00:12.96</td>
      <td>00:12.52</td>
      <td>00:13.77</td>
      <td>00:14.72</td>
    </tr>
    <tr>
      <th>PugetBench for Premiere Pro</th>
      <td>1227</td>
      <td>1148</td>
      <td>1075</td>
      <td>1150</td>
    </tr>
    <tr>
      <th>PugetBench for Photoshop</th>
      <td>1517</td>
      <td>1497</td>
      <td>1440</td>
      <td>1351</td>
    </tr>
    <tr>
      <th>3DMark Time Spy CPU</th>
      <td>19205</td>
      <td>18650</td>
      <td>18323</td>
      <td>18927</td>
    </tr>
    <tr>
      <th>Metro Exodus (ultra / high)</th>
      <td>150fps</td>
      <td>147fps</td>
      <td>146fps</td>
      <td>142fps</td>
    </tr>
    <tr>
      <th>Shadow of the Tomb Raider</th>
      <td>244fps</td>
      <td>230fps</td>
      <td>227fps</td>
      <td>217fps</td>
    </tr>
    <tr>
      <th>Gears 5</th>
      <td>187fps</td>
      <td>167fps</td>
      <td>169fps</td>
      <td>174fps</td>
    </tr>
    <tr>
      <th>Assassin's Creed Valhalla</th>
      <td>138fps</td>
      <td>138fps</td>
      <td>114fps</td>
      <td>112fps</td>
    </tr>
    <tr>
      <th>Watch Dogs: Legion</th>
      <td>126fps</td>
      <td>123fps</td>
      <td>116fps</td>
      <td>119fps</td>
    </tr>
    <tr>
      <th>Cyberpunk 2077</th>
      <td>144fps</td>
      <td>144fps</td>
      <td>138fps</td>
      <td>138fps</td>
    </tr>
  </tbody>
</table>
</div>




```python

```


```python
benchmark_labels = d['elements']['content']['content']['entities']['221a4a5b-d6f4-4c7a-b26c-a02911a7efa3']['props']['chartData']['sheetnames']
benchmark_labels
```




    ['CinebenchR23',
     'AdobeCreativeSuiteTests',
     'Handbrake1.5.1',
     'Blender2.93',
     'POV-Ray3.7']




```python
type(benchmark_labels)
```




    list




```python
benchmark_data = d['elements']['content']['content']['entities']['221a4a5b-d6f4-4c7a-b26c-a02911a7efa3']['props']['chartData']['data']
benchmark_data
```




    [[['FURMARK', 'Multi-Threaded', 'Single-Threaded'],
      ['IntelCorei9-13900K', '39022', '2275'],
      ['AMDRyzen97950X', '35063', '2020'],
      ['AMDRyzen77700X', '19083', '2001'],
      ['AMDRyzen95950X', '24724', '1626'],
      ['AMDRyzen75800X3D', '14372', '1478'],
      ['AMDRyzen75700X', '14685', '1548'],
      ['IntelCorei9-12900K', '27131', '1987'],
      ['IntelCorei7-12700K', '22797', '1940'],
      ['IntelCorei5-12600K', '17411', '1917'],
      ['IntelCorei9-11900K', '14987', '1607'],
      ['IntelCorei9-10900K', '14929', '1381']],
     [['', 'Photoshop(PugetBench)', 'PremierePro(PugetBench)'],
      ['IntelCorei9-13900K', '1489', '1189'],
      ['AMDRyzen97950X', '1325', '1209'],
      ['AMDRyzen77700X', '1381', '975'],
      ['AMDRyzen95950X', '1017', '986'],
      ['AMDRyzen75800X3D', '1031', '857'],
      ['AMDRyzen75700X', '1011', '849'],
      ['IntelCorei9-12900K', '1320', '1059'],
      ['IntelCorei7-12700K', '1272', '1002'],
      ['IntelCorei5-12600K', '1260', '899'],
      ['IntelCorei9-11900K', '894', '785'],
      ['IntelCorei9-10900K', '848', '802']],
     [['', 'Renderof"TearsofSteel"filefrom4Kto1080p'],
      ['IntelCorei9-13900K', '172'],
      ['AMDRyzen97950X', '199'],
      ['AMDRyzen77700X', '254'],
      ['AMDRyzen95950X', '267'],
      ['AMDRyzen75800X3D', '304'],
      ['AMDRyzen75700X', '322'],
      ['IntelCorei9-12900K', '208'],
      ['IntelCorei7-12700K', '231'],
      ['IntelCorei5-12600K', '276'],
      ['IntelCorei9-11900K', '329'],
      ['IntelCorei9-10900K', '345']],
     [['', 'ImageRenderingTime'],
      ['IntelCorei9-13900K', '63'],
      ['AMDRyzen97950X', '70'],
      ['AMDRyzen77700X', '127'],
      ['AMDRyzen95950X', '97'],
      ['AMDRyzen75800X3D', '168'],
      ['AMDRyzen75700X', '174'],
      ['IntelCorei9-12900K', '92'],
      ['IntelCorei7-12700K', '110'],
      ['IntelCorei5-12600K', '147'],
      ['IntelCorei9-11900K', '99'],
      ['IntelCorei9-10900K', '137']],
     [['', 'Single-ThreadedTest', 'Multi-ThreadedTest'],
      ['IntelCorei9-13900K', '306', '20'],
      ['AMDRyzen97950X', '360', '22'],
      ['AMDRyzen77700X', '349', '38'],
      ['AMDRyzen95950X', '417', '30'],
      ['AMDRyzen75800X3D', '458', '50'],
      ['AMDRyzen75700X', '449', '52'],
      ['IntelCorei9-12900K', '353', '29'],
      ['IntelCorei7-12700K', '366', '35'],
      ['IntelCorei5-12600K', '377', '45'],
      ['IntelCorei9-11900K', '439', '57'],
      ['IntelCorei9-10900K', '482', '48']]]




```python
processor = ['Intel','AMD']

data = []

for i,l in enumerate(benchmark_data):
                    
    lf = [row for row in l if any(word in row[0] for word in ["Intel", "AMD"])] # row[0] for check the first element which is the processor. 

    data.append(lf)
    
```


```python
 # Initialize a dict
d = {}
    
for l in data:
    for e in l:
        #print('Element', e)
        key = e[0]  
        values = e[1:] 

        #print(f'key {key} and values {values}')
    
        # # Check if the key exists
        if key not in d:
            d[key] = [values] 
        else:
            # Otherwise append the values
            d[key].append(values)
```


```python
# Create a DataFrame with keys as row indices
df = pd.DataFrame.from_dict(d, orient='index', columns=benchmark_labels)
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>CinebenchR23</th>
      <th>AdobeCreativeSuiteTests</th>
      <th>Handbrake1.5.1</th>
      <th>Blender2.93</th>
      <th>POV-Ray3.7</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>IntelCorei9-13900K</th>
      <td>[39022, 2275]</td>
      <td>[1489, 1189]</td>
      <td>[172]</td>
      <td>[63]</td>
      <td>[306, 20]</td>
    </tr>
    <tr>
      <th>AMDRyzen97950X</th>
      <td>[35063, 2020]</td>
      <td>[1325, 1209]</td>
      <td>[199]</td>
      <td>[70]</td>
      <td>[360, 22]</td>
    </tr>
    <tr>
      <th>AMDRyzen77700X</th>
      <td>[19083, 2001]</td>
      <td>[1381, 975]</td>
      <td>[254]</td>
      <td>[127]</td>
      <td>[349, 38]</td>
    </tr>
    <tr>
      <th>AMDRyzen95950X</th>
      <td>[24724, 1626]</td>
      <td>[1017, 986]</td>
      <td>[267]</td>
      <td>[97]</td>
      <td>[417, 30]</td>
    </tr>
    <tr>
      <th>AMDRyzen75800X3D</th>
      <td>[14372, 1478]</td>
      <td>[1031, 857]</td>
      <td>[304]</td>
      <td>[168]</td>
      <td>[458, 50]</td>
    </tr>
    <tr>
      <th>AMDRyzen75700X</th>
      <td>[14685, 1548]</td>
      <td>[1011, 849]</td>
      <td>[322]</td>
      <td>[174]</td>
      <td>[449, 52]</td>
    </tr>
    <tr>
      <th>IntelCorei9-12900K</th>
      <td>[27131, 1987]</td>
      <td>[1320, 1059]</td>
      <td>[208]</td>
      <td>[92]</td>
      <td>[353, 29]</td>
    </tr>
    <tr>
      <th>IntelCorei7-12700K</th>
      <td>[22797, 1940]</td>
      <td>[1272, 1002]</td>
      <td>[231]</td>
      <td>[110]</td>
      <td>[366, 35]</td>
    </tr>
    <tr>
      <th>IntelCorei5-12600K</th>
      <td>[17411, 1917]</td>
      <td>[1260, 899]</td>
      <td>[276]</td>
      <td>[147]</td>
      <td>[377, 45]</td>
    </tr>
    <tr>
      <th>IntelCorei9-11900K</th>
      <td>[14987, 1607]</td>
      <td>[894, 785]</td>
      <td>[329]</td>
      <td>[99]</td>
      <td>[439, 57]</td>
    </tr>
    <tr>
      <th>IntelCorei9-10900K</th>
      <td>[14929, 1381]</td>
      <td>[848, 802]</td>
      <td>[345]</td>
      <td>[137]</td>
      <td>[482, 48]</td>
    </tr>
  </tbody>
</table>
</div>




```python

```


```python

```


```python
from transformers import AutoTokenizer, PegasusForConditionalGeneration

model = PegasusForConditionalGeneration.from_pretrained("google/pegasus-xsum")
tokenizer = AutoTokenizer.from_pretrained("google/pegasus-xsum")

ARTICLE_TO_SUMMARIZE = (
    "PG&E stated it scheduled the blackouts in response to forecasts for high winds "
    "amid dry conditions. The aim is to reduce the risk of wildfires. Nearly 800 thousand customers were "
    "scheduled to be affected by the shutoffs which were expected to last through at least midday tomorrow."
)
inputs = tokenizer(ARTICLE_TO_SUMMARIZE, max_length=1024, return_tensors="pt")

# Generate Summary
summary_ids = model.generate(inputs["input_ids"])
tokenizer.batch_decode(summary_ids, skip_special_tokens=True, clean_up_tokenization_spaces=False)[0]
```

 

 

 
