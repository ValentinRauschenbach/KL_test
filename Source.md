```python
import numpy as np
import requests
from lxml import html
import pandas

# Link to the list of smartphones on enter.ru
url = 'http://www.enter.ru/catalog/electronics/telefoni-smartfoni-2348'
links = []

# Using search pages to collect URLs of smartphones descriptions
for i in range(1, 12):
    # Adding page number to the URL of the list
    if i == 1:
        url_page = url
    else:
        url_page = url + '?page=' + str(i)
    
    # Requesting the current URL. If the response is bad, breaking loop
    req = requests.get(url)
    if req.status_code != requests.codes.ok:
        break

    # Getting the corresponding HTML-document
    doc = html.fromstring(req.text)
    
    # Collecting all the links to pages with smartphones descriptions (class = "listing__title"),
    # using the xpath() method.
    # In these links the relative addresses are used, so in order to make further requests I must
    # turn them into absolute by adding the root address "http://www.enter.ru".
    links.extend(['http://www.enter.ru' + path for path in doc.xpath('//a[@class="listing__title"]/@href')])

# Different props lists definition
product_names = []
props_names = []
props_values = []
photo_url = []
price = []
props = []

# Loop through collected links to find information about smartphone
for url in links:
    html_doc = requests.get(url)
    doc = html.fromstring(html_doc.text)
    
    # Fetching smartphone name
    product_names.append(''.join(doc.xpath('//h1[@class="product-name"]/text()')))
    
    # Fetching smartphone features
    props_names = doc.xpath('//span[@class="props-list__name-i"]/text()')
    props_values = [s.strip() for s in doc.xpath('//dd[@class="props-list__val"]/text()') if s.strip() != '']
    
    # Fetching smartphone photos and taking the first two of them
    photo_url = doc.xpath('//img[@class="product-card-photo-thumbs__img"]/@src')[0:2]
    
    # Fetching smartphone price
    price = ''.join(doc.xpath('//span[@class="product-card-price__val i-info__tx"]/text()'))
    
    # Combine all smartphone properties
    props.append({'Цена': price})
    props[-1].update({'Фотография ' + str(i + 1): photo_url[i] for i in range(len(photo_url))})
    props[-1].update({props_names[i]: props_values[i] for i in range(len(props_names) - 1)})

# Converting all collected data to a DataFrame object and writing it to an existing Excel file
dt = pandas.DataFrame(props, index=np.array(product_names))
dt.to_excel('C:\output.xlsx', sheet_name='Smatrphones')
```