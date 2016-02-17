Ниже приведен алгоритм, позволяющий парсить сайт интернет-магазина, собирая данные о товарах.

# Библиотеки
Я использую [Numpy](http://www.numpy.org/) для операций с базовыми типами, [Pandas](http://pandas.pydata.org/) для операций с данными (в данном случае, для создания индексированной двумерной таблицы и записи в файл), [Requests](http://docs.python-requests.org/en/master/) для передачи запроса по URL-адресу и [lxml](http://lxml.de/) для разбора HTML-документа. Вместо [Requests](http://docs.python-requests.org/en/master/) можно пользоваться стандартной библиотекой [urllib2](https://docs.python.org/2/library/urllib2.html), однако я использую первую из-за хорошего API.

```python
import numpy as np
import requests
from lxml import html
import pandas
```

# Сбор ссылок на страницы с описанием смартфонов
Для разбора контента интернет-магазина я взял URL-адрес _http://www.enter.ru/catalog/electronics/telefoni-smartfoni-2348_, которому соответствует список смартфонов, предлагаемых магазином Enter.ru. Из этого списка я хочу получить URL-адреса страниц с подробным описанием каждого из сартфонов. Для этого достаточно пройтись в цикле по страницам со смартфонами и выцепить все HTML-элементы _<a></a>_ (ссылки) с атрибутом _class="listing__title"_ (сам элемент выглядит как _<a class="listing__title" href=...>_):

```python
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
```

# Сбор данных о смартфоне
Теперь, когда у меня есть набор URL-адресов страниц с описанием смартфонов, я могу разобрать каждую страницу по отдельности, находя нужные мне данные. Я соберу описания всевозможных свойств смартфона, расположенные в блоке "Характеристики", а также URL-адреса двух различных фотографий смартфона, его цену и наименование:
- наименование смартфона расположено в заголовке первого уровня _\<h1 class="product-name">\</h1>_:
```python
doc.xpath('//h1[@class="product-name"]/text()')
```
- наименования различных характеристик смартфона спрятаны в элементы _<span class="props-list__name-i"></span>_, а значения этих характеристик располагаются в элементах _<dd class="props-list__val">_:
```python
doc.xpath('//span[@class="props-list__name-i"]/text()')  # returns list of prop names
[s.strip() for s in doc.xpath('//dd[@class="props-list__val"]/text()') if s.strip() != '']  # trimmed values for props
```
- фотографии на этих страницах в HTML-структуре имеют один класс, так что их просто найти по классу _<img class="product-card-photo-thumbs__img">_ и взять первые две из них (думаю, этого будет достаточно):
```python
doc.xpath('//img[@class="product-card-photo-thumbs__img"]/@src')[0:2]
```
- наконец, цена смартфона извлекается из элемента _<span class="product-card-price__val i-info__tx"></span>_, который на всей странице один:
```python
doc.xpath('//span[@class="product-card-price__val i-info__tx"]/text()')
```

После всего этого, из собранных на предыдущем этапе свойств (кроме наименования) составляется один словарь, и я начинаю обрабатывать следующий URL-адрес. После обработки всех имеющихся страниц данные о смартфонах преобразуются в таблицу DataFrame, где строки соответствуют наименованию смартфона, а все его характеристики записаны в столбцы. Получившаяся таблица записывается в **существующий** Excel файл.

```python
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

# Converting all collected data to a DataFrame object, replace NaN values with negative value and
# writing it to an existing Excel file
dt = pandas.DataFrame(props, index=np.array(product_names))
dt = dt.fillna('нет')
dt.to_excel('C:\output.xlsx', sheet_name='Smatrphones')\