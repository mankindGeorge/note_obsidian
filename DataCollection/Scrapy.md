
- 7大组件组成：
	- 引擎
	- 调度器
	- 下载器
	- **爬虫**
	- **实体管道**
	- 下载器中间件
	- 爬虫中间件

---
# Scrapy 请求与响应流程 (Request Flow)

在 Scrapy 中，爬取不是简单的循环，而是一个**异步闭环**：

1. **初始**：Spider 将第一个 URL 封装成 `Request` 交给 Engine。
2. **入队**：Engine 将 Request 转交给 Scheduler 排序入队。
3. **取样**：Engine 从 Scheduler 索取下一个 Request。
4. **下载**：Engine 通过 Downloader Middleware 将 Request 发给 Downloader。
5. **回传**：下载完成后，Response 经过 Downloader Middleware 返回给 Engine。
6. **解析**：Engine 发送 Response 给 Spider，Spider 执行 `parse()` 回调函数。
7. **分流**：如果提取出 **数据 (Item)** -> 发往 Pipeline。

> 如果提取出 **新链接 (Request)** -> 再次进入调度器循环。

### 运行第一个 Scrapy 程序

Scrapy 是一个项目框架，必须遵循特定的目录结构。

#### **第一步：安装与创建项目**

在终端（或 Conda 环境）执行：

```Bash
pip install scrapy
scrapy startproject my_first_spider  # 创建项目文件夹
cd my_first_spider

# 或者使用 genspider 直接生成爬虫文件
scrapy genspider quotes quotes.toscrape.com  # 自动创建 quotes.py
```

#### **第二步：编写爬虫 (Spider)**

进入 `spiders` 文件夹，创建一个 `example_spider.py`：

```Python
import scrapy

class ExampleSpider(scrapy.Spider):
    name = "quotes"  # 爬虫的唯一标识名
    start_urls = [
        'https://quotes.toscrape.com/' # 初始抓取地址
    ]

    def parse(self, response):
        """解析响应的回调函数"""
        # 抓取页面上的所有名言
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
            }

        # 自动发现下一页链接（对应之前学的“通过链接抓取”）
        next_page = response.css('li.next a::attr(href)').get()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)
```

#### **第三步：运行爬虫**

在项目根目录下执行：

```Bash
# -o 表示 output，将结果直接保存到 JSON 文件
scrapy crawl quotes -o results.json
```

## Selector 选择器详解

Scrapy 内置了 CSS 和 XPath 两种选择器，通过 `response.css()` 和 `response.xpath()` 调用。

### CSS 选择器基础

#### 基础语法

```python
response.css('选择器')      # 返回 SelectorList
response.css('选择器').get()      # 返回第一个结果（字符串）
response.css('选择器').getall()   # 返回所有结果（列表）
```

#### 常用选择器语法

| 选择器 | 说明 | 示例 |
| --- | --- | --- |
| `tag` | 按标签名 | `css('div')` |
| `.class` | 按类名 | `css('.quote')` |
| `#id` | 按 ID | `css('#content')` |
| `tag.class` | 组合条件 | `css('div.quote')` |
| `tag#id` | 组合条件 | `css('span.text')` |
| `parent > child` | 子元素 | `css('div > span')` |
| `ancestor descendant` | 后代元素 | `css('div span')` |
| `[attr]` | 含属性 | `css('a[href]')` |
| `[attr=value]` | 属性等于 | `css('a[href="url"]')` |
| `[attr*=value]` | 属性包含 | `css('a[href*="login"]')` |
| `[attr^=value]` | 属性开头 | `css('a[href^="https"]')` |
| `[attr$=value]` | 属性结尾 | `css('img[src$=".jpg"]')` |

#### 伪类选择器

| 选择器 | 说明 | 示例 |
| --- | --- | --- |
| `:first-child` | 第一个子元素 | `css('li:first-child')` |
| `:last-child` | 最后一个子元素 | `css('li:last-child')` |
| `:nth-child(n)` | 第 n 个子元素 | `css('li:nth-child(2)')` |
| `:contains(text)` | 包含文本 | `css('span:contains("Hello")')` |
| `:empty` | 空元素 | `css('div:empty')` |

### 提取内容

#### 获取文本

```python
# ::text 获取元素的文本内容（不包含 HTML）
response.css('div.quote span.text::text').get()
# 结果: "The world as we have created..."

# 获取所有匹配的文本
response.css('div.tags a.tag::text').getall()
# 结果: ["love", "inspirational", "life"]
```

#### 获取属性

```python
# ::attr(name) 获取元素属性值
response.css('a::attr(href)').get()           # 获取链接
response.css('img::attr(src)').get()          # 获取图片地址
response.css('input::attr(placeholder)').get() # 获取输入框占位符
```

#### 链式调用

```python
# 先选中父元素，再从中筛选子元素
for quote in response.css('div.quote'):
    text = quote.css('span.text::text').get()
    author = quote.css('small.author::text').get()
    # quote 是当前 QuoteItem 的 Selector，可以继续用 .css() 筛选
```

#### 层级关系

```python
# 单个元素 vs 多个元素
response.css('h1::text').get()     # 返回 str 或 None
response.css('li a::text').getall() # 返回 list

# get() 带默认值
response.css('div.missing::text').get()          # 无结果返回 None
response.css('div.missing::text').get(default='N/A')  # 无结果返回 'N/A'
```

### 实际示例解析

```python
for quote in response.css('div.quote'):
    # 1. 选中所有名言容器 <div class="quote">
    yield {
        # 2. 在当前容器内找 <span class="text">，获取其文本
        'text': quote.css('span.text::text').get(),

        # 3. 在当前容器内找 <small class="author">，获取其文本
        'author': quote.css('small.author::text').get(),

        # 4. 在当前容器内找所有 <a class="tag">，获取所有文本
        'tags': quote.css('div.tags a.tag::text').getall(),
    }
```

### XPath 选择器

XPath 是一种在 XML/HTML 文档中查找节点的语言，比 CSS 更强大，支持函数和条件判断。

#### 基础语法

```python
response.xpath('xpath表达式')      # 返回 SelectorList
response.xpath('xpath').get()      # 返回第一个结果
response.xpath('xpath').getall()   # 返回所有结果
```

#### 常用路径表达式

| 表达式 | 说明 | 示例 |
| --- | --- | --- |
| `//tag` | 任意位置的标签 | `//div` |
| `/tag` | 根目录下的标签 | `/html/body/div` |
| `//tag[@attr]` | 含属性的标签 | `//a[@href]` |
| `//tag[@attr="value"]` | 属性等于 | `//a[@href="url"]` |
| `.` | 当前节点 | `.//div` |
| `..` | 父节点 | `..//div` |

#### 轴（Axes）

| 表达式 | 说明 | 示例 |
| --- | --- | --- |
| `//div/child::p` | 子元素 | `//div/child::p` |
| `//div/descendant::span` | 后代元素 | `//div/descendant::span` |
| `//p/parent::div` | 父元素 | `//p/parent::div` |
| `//span/following-sibling::p` | 后续兄弟 | `//span/following-sibling::p` |
| `//span/preceding-sibling::p` | 前置兄弟 | `//span/preceding-sibling::p` |

#### 常用函数

| 函数 | 说明 | 示例 |
| --- | --- | --- |
| `text()` | 获取文本 | `//h1/text()` |
| `normalize-space()` | 去除空白 | `normalize-space(//p/text())` |
| `contains(@attr, 'text')` | 属性包含 | `//a[contains(@href, 'login')]` |
| `starts-with(@attr, 'text')` | 属性开头 | `//img[starts-with(@src, 'https')]` |
| `ends-with(@attr, 'text')` | 属性结尾 | `//img[ends-with(@src, '.jpg')]` |
| `position()` | 位置 | `//li[position()=1]` |
| `last()` | 最后一个 | `//li[last()]` |
| `count()` | 计数 | `count(//div)` |
| `string-length()` | 字符串长度 | `string-length(//title)` |

#### 谓词（条件筛选）

```python
# 按位置
response.xpath('//li[1]')                  # 第一个 li
response.xpath('//li[last()]')              # 最后一个 li
response.xpath('//li[position() < 3]')      # 前两个 li

# 按属性条件
response.xpath('//a[@class="link"]')        # class="link"
response.xpath('//a[@href]')                # 有 href 属性
response.xpath('//div[@id and @class]')    # 同时有 id 和 class

# 按文本内容
response.xpath('//span[text()="Hello"]')   # 文本等于 "Hello"
response.xpath('//p[contains(text(), "error")]')  # 包含 "error"
```

#### 提取内容

```python
# 获取文本
response.xpath('//h1/text()').get()
response.xpath('//div/text()').getall()

# 获取属性
response.xpath('//a/@href').get()          # 获取链接
response.xpath('//img/@src').get()         # 获取图片

# 组合：获取属性值
response.xpath('//a[@class="author"]/@href').get()
```

#### CSS 与 XPath 对照

| CSS | XPath |
| --- | --- |
| `div` | `//div` |
| `div > p` | `//div/p` |
| `div p` | `//div//p` |
| `div.quote` | `//div[contains(@class,'quote')]` |
| `div#content` | `//div[@id='content']` |
| `a[href]` | `//a[@href]` |
| `a[href="url"]` | `//a[@href='url']` |
| `a::text` | `//a/text()` |
| `a::attr(href)` | `//a/@href` |
| `li:first-child` | `//li[1]` |
| `li:last-child` | `//li[last()]` |
| `li:nth-child(2)` | `//li[2]` |

#### 实际示例解析

```python
# HTML 结构
# <div class="quote">
#     <span class="text">"Hello World"</span>
#     <small class="author">Author Name</small>
# </div>

# XPath 版本
for quote in response.xpath('//div[@class="quote"]'):
    yield {
        'text': quote.xpath('.//span[@class="text"]/text()').get(),
        'author': quote.xpath('.//small[@class="author"]/text()').get(),
    }

# 等价的 CSS 版本
for quote in response.css('div.quote'):
    yield {
        'text': quote.css('span.text::text').get(),
        'author': quote.css('small.author::text').get(),
    }
```

#### XPath 特有的强大用法

```python
# 1. 获取父元素
response.xpath('//span[@class="author"]/..').get()

# 2. 获取兄弟元素
response.xpath('//h2[text()="Title"]/following-sibling::p/text()').getall()

# 3. 组合条件
response.xpath('//div[contains(@class,"item") and @data-id]//a/@href').getall()

# 4. 统计数量
count = response.xpath('count(//div[@class="product"])').get()
# count: 25

# 5. 字符串处理
response.xpath('//title/text()').re(r'(\d+)')  # 提取数字
```

#### 链式调用

```python
# XPath 同样支持链式调用
for item in response.xpath('//div[@class="item"]'):
    price = item.xpath('.//span[@class="price"]/text()').re(r'[\d.]+')
    # 继续处理...
```

---

# CrawlSpider 规则爬虫

CrawlSpider 是 Scrapy 提供的**基于规则的自动爬虫类**，适合结构固定的网站（如新闻、博客、商品列表）。

## 与普通 Spider 的区别

| 特性 | Spider | CrawlSpider |
| --- | --- | --- |
| 链接发现 | 手动 `follow` | 自动匹配规则 |
| 代码量 | 多 | 少 |
| 适合场景 | 复杂逻辑 | 结构固定 |
| 去重 | 需手动处理 | 自动处理 |

## 基本结构

```python
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class MySpider(CrawlSpider):
    name = 'myspider'
    allowed_domains = ['example.com']
    start_urls = ['https://www.example.com/']

    rules = (
        Rule(LinkExtractor(...), callback='parse_item'),
        Rule(LinkExtractor(...), follow=True),
    )
```

## Rule 参数说明

| 参数 | 说明 |
| --- | --- |
| `LinkExtractor` | 链接提取器，定义匹配规则 |
| `callback` | 回调函数，解析链接页面（不设则只跟进不解析） |
| `follow` | 是否跟进匹配到的链接 |
| `cb_kwargs` | 传递给回调函数的参数 |

## LinkExtractor 常用参数

```python
# 按正则匹配
LinkExtractor(allow=r'/article/\d+')

# 按正则排除
LinkExtractor(deny=r'/admin|/login')

# 限制域名
LinkExtractor(allow_domains=['news.com'])

# 限制区域（只在特定范围内找链接）
LinkExtractor(restrict_xpaths='//div[@class="pagination"]')
LinkExtractor(restrict_css='.pagination a')
```

## 实战示例

### 示例：爬取新闻列表

```python
import scrapy
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class NewsSpider(CrawlSpider):
    name = 'news'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/news']

    rules = (
        # 匹配文章链接，交给 parse_item 解析
        Rule(
            LinkExtractor(allow=r'/news/\d+\.html'),
            callback='parse_item'
        ),
        # 匹配分页链接，继续跟进（不设 callback）
        Rule(
            LinkExtractor(allow=r'/news/page_\d+\.html'),
            follow=True
        ),
    )

    def parse_item(self, response):
        yield {
            'title': response.css('h1.title::text').get(),
            'content': response.css('div.content::text').getall(),
            'date': response.css('span.date::text').get(),
            'url': response.url,
        }
```

### 示例：爬取博客园

```python
class CnblogsSpider(CrawlSpider):
    name = 'cnblogs'
    allowed_domains = ['cnblogs.com']
    start_urls = ['https://www.cnblogs.com/']

    rules = (
        # 匹配文章链接
        Rule(
            LinkExtractor(
                allow=r'/.*/p/\d+\.html',
                restrict_css='div.post-item'
            ),
            callback='parse_item'
        ),
        # 匹配分页链接，继续跟进
        Rule(
            LinkExtractor(
                allow=r'/sitehome/p/\d+',
                restrict_css='div.pager'
            ),
            follow=True
        ),
    )

    def parse_item(self, response):
        yield {
            'title': response.css('a.post-title::text').get(),
            'author': response.css('a.author::text').get(),
            'date': response.css('span.post-date::text').get(),
            'content': response.css('div.post-content').get(),
            'url': response.url,
        }
```

## 常用配置组合

```python
# 只抓取匹配正则的链接
rules = (
    Rule(LinkExtractor(allow=r'/article/\d+'), callback='parse_item'),
)

# 匹配多个规则
rules = (
    Rule(LinkExtractor(allow=r'/category/\w+'), follow=True),
    Rule(LinkExtractor(allow=r'/product/\d+'), callback='parse_product'),
    Rule(LinkExtractor(allow=r'/page/\d+'), follow=True),
)

# 排除不需要的链接
rules = (
    Rule(LinkExtractor(
        allow=r'/article/\d+',
        deny=[r'/admin', r'/user/profile']
    ), callback='parse_item'),
)
```

## 注意事项

1. **callback 不能用 `parse`**：CrawlSpider 的 `parse` 方法已被框架使用
2. **follow 的作用**：
   - `follow=True`：匹配到的链接会被加入待抓队列
   - `follow=False`：只抓匹配到的链接，不跟进
3. **多 Rule 执行顺序**：按定义顺序依次匹配

---

# Item 与 ItemLoader

## Item 的基本使用

Item 是 Scrapy 中用于定义抓取数据结构的类，比普通字典更规范、支持字段验证。

### 定义 Item

```python
import scrapy

class QuoteItem(scrapy.Item):
    text = scrapy.Field()           # 名言内容
    author = scrapy.Field()         # 作者
    tags = scrapy.Field()           # 标签（列表）
```

### 使用 Item

```python
def parse(self, response):
    for quote in response.css('div.quote'):
        item = QuoteItem()
        item['text'] = quote.css('span.text::text').get()
        item['author'] = quote.css('small.author::text').get()
        item['tags'] = quote.css('div.tags a.tag::text').getall()
        yield item
```

### Item 的优势

| 特性 | 说明 |
| --- | --- |
| **类型提示** | IDE 自动补全、类型检查 |
| **字段验证** | 可定义字段默认值、序列化器 |
| **扩展性** | 方便继承和复用 |
| **Pipeline 集成** | 与 Pipeline 无缝对接 |

## ItemLoader

ItemLoader 解决的是**选择器与 Item 之间的冗余映射问题**。当一个字段需要多个选择器、或需要清洗处理时，代码会变得混乱。

### 基本用法

```python
from scrapy.loader import ItemLoader
from scrapy.loader.processors import TakeFirst, MapCompose, Join

class QuoteItem(scrapy.Item):
    text = scrapy.Field()
    author = scrapy.Field()
    tags = scrapy.Field()

class QuoteItemLoader(ItemLoader):
    # 默认输出处理器：取第一个非空值
    default_output_processor = TakeFirst()
    
    # 为特定字段指定输入/输出处理器
    text_in = MapCompose(str.strip)          # 输入时去除空白
    author_in = MapCompose(str.title)        # 输入时首字母大写
    tags_out = Join(', ')                     # 输出时用逗号连接
```

### 在 Spider 中使用

```python
def parse(self, response):
    for quote in response.css('div.quote'):
        loader = QuoteItemLoader(item=QuoteItem(), response=response)
        
        # 使用 CSS 选择器加载字段
        loader.add_css('text', 'span.text::text')
        loader.add_css('author', 'small.author::text')
        loader.add_css('tags', 'div.tags a.tag::text')
        
        yield loader.load_item()
```

### 使用 XPath 选择器

```python
loader.add_xpath('title', '//h1[@class="title"]/text()')
loader.add_xpath('content', '//div[@class="article"]//p/text()')
```

### 使用自定义函数加载

```python
def extract_date(response):
    # 自定义解析逻辑
    date_str = response.css('time::attr(datetime)').get()
    return parse_date(date_str)

loader.add_value('date', extract_date(response))
loader.add_value('url', response.url)  # 直接添加值
```

## 常用 Processor

| Processor | 说明 | 示例 |
| --- | --- | --- |
| `TakeFirst` | 返回第一个非空值 | 默认输出处理器 |
| `Identity` | 原样返回 | 不做任何处理 |
| `MapCompose(*funcs)` | 对列表每个元素执行函数链 | `MapCompose(str.strip, str.lower)` |
| `Join(separator)` | 用分隔符连接为字符串 | `Join(', ')` |
| `Compose(*funcs)` | 将多个函数组合成一个 | 对单个值处理 |
| `SelectJmes(json_path)` | 从 JSON 字段提取 | 用于 JSON 响应 |

### 实战：带数据清洗的 ItemLoader

```python
import scrapy
from scrapy.loader import ItemLoader
from scrapy.loader.processors import TakeFirst, MapCompose, Join
import re

def clean_price(value):
    """清洗价格：移除货币符号和逗号"""
    if value:
        return re.sub(r'[¥$,]', '', str(value))
    return value

def parse_tags(value):
    """处理标签：去重并转小写"""
    return value.lower().strip()

class ProductItem(scrapy.Item):
    name = scrapy.Field()
    price = scrapy.Field()
    tags = scrapy.Field()
    rating = scrapy.Field()

class ProductLoader(ItemLoader):
    default_output_processor = TakeFirst()
    
    name_in = MapCompose(str.strip, str.title)
    price_in = MapCompose(clean_price)
    tags_in = MapCompose(parse_tags)
    tags_out = Join(',')
    rating_in = MapCompose(lambda x: float(x) if x else 0.0)
```

## ItemLoader 嵌套

当页面结构复杂时，可以使用**嵌套加载器**：

```python
class AuthorLoader(ItemLoader):
    default_output_processor = TakeFirst()
    name_out = MapCompose(str.strip)
    bio_out = Join()

class QuoteItemLoader(ItemLoader):
    default_output_processor = TakeFirst()
    
    text_in = MapCompose(str.strip)
    
    # 嵌套加载作者信息
    author_name = scrapy.Field()
    author_bio = scrapy.Field()

def parse(self, response):
    for quote in response.css('div.quote'):
        loader = QuoteItemLoader(item=QuoteItem(), response=response)
        loader.add_css('text', 'span.text::text')
        
        # 嵌套解析作者
        author_loader = AuthorLoader(selector=quote.css('small.author').get())
        author_loader.add_css('name', 'small.author::text')
        author_loader.add_css('bio', 'span.author::attr(title)')
        
        author_data = author_loader.load_item()
        loader.add_value('author_name', author_data.get('name'))
        loader.add_value('author_bio', author_data.get('bio'))
        
        yield loader.load_item()
```

---

# Item Pipeline 实战

Pipeline 是 Scrapy 处理 Item 的最后一环，常用于：
- 数据验证
- 数据清洗
- 数据去重
- 数据存储（数据库/文件）

## Pipeline 结构

```python
class MyPipeline:
    def open_spider(self, spider):
        """爬虫启动时调用（可选）"""
        pass
    
    def process_item(self, item, spider):
        """处理每个 Item（必须实现）"""
        return item  # 或 raise DropItem
    
    def close_spider(self, spider):
        """爬虫关闭时调用（可选）"""
        pass
```

## 启用 Pipeline

在 `settings.py` 中配置优先级（数字越小越先执行）：

```python
ITEM_PIPELINES = {
    'myproject.pipelines.PricePipeline': 100,
    'myproject.pipelines.ValidationPipeline': 200,
    'myproject.pipelines.DuplicatesPipeline': 300,
    'myproject.pipelines.MongoDBPipeline': 400,
}
```

## 实战一：数据验证

```python
import scrapy

class ValidationPipeline:
    """验证必填字段和类型"""
    
    def process_item(self, item, spider):
        # 检查必填字段
        required_fields = ['text', 'author']
        for field in required_fields:
            if not item.get(field):
                spider.logger.warning(f"缺少字段: {field}")
                raise DropItem(f"缺少字段: {field}")
        
        # 验证数据类型
        if not isinstance(item.get('tags', []), list):
            raise DropItem("tags 必须是列表")
        
        # 验证长度
        if len(item.get('text', '')) < 5:
            raise DropItem("内容过短")
        
        return item
```

## 实战二：数据清洗

```python
import re
from itemadapter import ItemAdapter

class CleanPipeline:
    """清洗和格式化数据"""
    
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # 清洗文本字段
        text_fields = ['text', 'author', 'title']
        for field in text_fields:
            if adapter.get(field):
                value = str(adapter[field])
                # 移除多余空白
                value = re.sub(r'\s+', ' ', value)
                # 移除特殊字符
                value = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', value)
                adapter[field] = value.strip()
        
        # 格式化价格
        if adapter.get('price'):
            price = re.sub(r'[¥$,]', '', str(adapter['price']))
            try:
                adapter['price'] = float(price)
            except ValueError:
                adapter['price'] = 0.0
        
        # 统一日期格式
        if adapter.get('date'):
            adapter['date'] = self.format_date(adapter['date'])
        
        return item
    
    def format_date(self, date_str):
        """转换为标准日期格式"""
        from datetime import datetime
        try:
            dt = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
            return dt.strftime('%Y-%m-%d %H:%M:%S')
        except:
            return date_str
```

## 实战三：数据去重

```python
from scrapy.exceptions import DropItem

class DuplicatesPipeline:
    """基于字段去重"""
    
    def __init__(self):
        self.seen_ids = set()
        self.seen_texts = set()
    
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # 按 ID 去重
        if adapter.get('id'):
            if adapter['id'] in self.seen_ids:
                raise DropItem(f"重复 ID: {adapter['id']}")
            self.seen_ids.add(adapter['id'])
        
        # 按内容指纹去重（模糊去重）
        if adapter.get('text'):
            text_fingerprint = self.fingerprint(adapter['text'])
            if text_fingerprint in self.seen_texts:
                raise DropItem("重复内容")
            self.seen_texts.add(text_fingerprint)
        
        return item
    
    @staticmethod
    def fingerprint(text):
        """生成文本指纹（去除空格和标点后比较）"""
        import re
        text = re.sub(r'[\s\W]', '', text.lower())
        return text[:50]  # 取前50字符作为指纹
```

## 实战四：数据存储到 JSON

```python
import json
from datetime import datetime

class JsonWriterPipeline:
    """将数据写入 JSON Lines 文件"""
    
    def __init__(self):
        self.file = None
        self.filename = None
    
    def open_spider(self, spider):
        # 使用爬虫名创建文件
        self.filename = f'{spider.name}_items.jl'
        self.file = open(self.filename, 'w', encoding='utf-8')
        spider.logger.info(f"写入文件: {self.filename}")
    
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # 添加元数据
        record = dict(adapter)
        record['_crawled_at'] = datetime.now().isoformat()
        record['_spider'] = spider.name
        
        # 写入 JSON Line
        line = json.dumps(record, ensure_ascii=False) + '\n'
        self.file.write(line)
        
        return item
    
    def close_spider(self, spider):
        if self.file:
            self.file.close()
            spider.logger.info(f"关闭文件: {self.filename}")
```

## 实战五：数据存储到 MongoDB

```python
import pymongo
from itemadapter import ItemAdapter

class MongoDBPipeline:
    """存储到 MongoDB"""
    
    collection_name = 'items'
    
    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db
        self.client = None
        self.db = None
    
    @classmethod
    def from_crawler(cls, crawler):
        """从 settings 获取配置"""
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE', 'scrapy')
        )
    
    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]
        spider.logger.info(f"连接 MongoDB: {self.mongo_db}")
    
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # 插入或更新（按 URL 去重）
        self.db[self.collection_name].update_one(
            {'url': adapter.get('url')},
            {'$set': dict(adapter)},
            upsert=True
        )
        
        return item
    
    def close_spider(self, spider):
        if self.client:
            self.client.close()
```

对应的 `settings.py`：

```python
MONGO_URI = 'mongodb://localhost:27017/'
MONGO_DATABASE = 'quotes_db'
```

## 实战六：图片下载 Pipeline

Scrapy 提供了 `ImagesPipeline` 用于下载图片：

```python
import scrapy
from scrapy.pipelines.images import ImagesPipeline
from itemadapter import ItemAdapter

class QuoteImagePipeline(ImagesPipeline):
    """下载名言配图"""
    
    def get_media_requests(self, item, info):
        adapter = ItemAdapter(item)
        image_url = adapter.get('image_url')
        if image_url:
            yield scrapy.Request(image_url)
    
    def item_completed(self, results, item, info):
        adapter = ItemAdapter(item)
        
        # 保存本地路径到 Item
        image_paths = [x['path'] for ok, x in results if ok]
        if image_paths:
            adapter['image_path'] = image_paths[0]
        else:
            adapter['image_path'] = None
        
        return item
```

启用图片 Pipeline：

```python
# settings.py
ITEM_PIPELINES = {
    'myproject.pipelines.QuoteImagePipeline': 1,
}

IMAGES_STORE = 'images/'           # 图片存储目录
IMAGES_MIN_WIDTH = 100             # 最小宽度
IMAGES_MIN_HEIGHT = 100            # 最小高度
IMAGES_EXPIRES = 90               # 过期天数
```

## Pipeline 组合使用

一个典型的 Pipeline 组合：

```python
ITEM_PIPELINES = {
    'myproject.pipelines.ValidationPipeline': 100,   # 1. 验证
    'myproject.pipelines.CleanPipeline': 200,         # 2. 清洗
    'myproject.pipelines.DuplicatesPipeline': 300,     # 3. 去重
    'myproject.pipelines.JsonWriterPipeline': 400,    # 4. 写文件
    'myproject.pipelines.MongoDBPipeline': 500,       # 5. 存数据库
}
```

执行顺序：**验证 → 清洗 → 去重 → 存储**

---

# Spider 进阶技巧

## follow_all 批量跟进链接

`follow_all` 是 `follow` 的批量版本，一次性处理多个链接。

### 基本用法

```python
def parse(self, response):
    # 一次性跟进所有作者链接
    yield from response.follow_all(
        response.css('div.quote a.author::attr(href)'),
        callback=self.parse_author
    )
```

### 等价写法

```python
# follow_all
yield from response.follow_all(
    response.css('a.author'),
    callback=self.parse_author
)

# 等价于手动循环
for href in response.css('a.author::attr(href)'):
    yield response.follow(href, callback=self.parse_author)
```

### 传选择器字符串

```python
# 直接传选择器，更简洁
yield from response.follow_all(
    'div.quote a.author',
    callback=self.parse_author
)
```

## Request 携带参数

### meta 参数（跨页面传递数据）

```python
def parse(self, response):
    # 提取列表页信息
    for item in response.css('div.item'):
        item_data = {
            'title': item.css('h2::text').get(),
            'category': item.css('span.cat::text').get(),
        }
        # 通过 meta 传递给详情页
        yield response.follow(
            item.css('a.detail::attr(href)').get(),
            callback=self.parse_detail,
            meta={'item_data': item_data}
        )

def parse_detail(self, response):
    # 从 meta 中取出之前的数据
    item_data = response.meta['item_data']
    item_data['content'] = response.css('div.content::text').get()
    yield item_data
```

### cb_kwargs 参数（推荐方式）

```python
def parse(self, response):
    for item in response.css('div.item'):
        yield response.follow(
            item.css('a.detail::attr(href)').get(),
            callback=self.parse_detail,
            cb_kwargs={'category': 'electronics'}  # 更直观
        )

def parse_detail(self, response, category):
    # 直接作为函数参数接收
    yield {
        'category': category,
        'content': response.css('div.content::text').get(),
    }
```

### headers 和 cookies

```python
yield scrapy.Request(
    url,
    headers={'Referer': 'https://example.com'},
    cookies={'session_id': 'abc123'},
    callback=self.parse
)
```

## 多个 start_urls 的写法

### 方式1：列表

```python
class MultiSpider(scrapy.Spider):
    name = 'multi'
    start_urls = [
        'https://example.com/category/1',
        'https://example.com/category/2',
        'https://example.com/category/3',
    ]
```

### 方式2：start_requests 方法

```python
class DynamicSpider(scrapy.Spider):
    name = 'dynamic'

    def start_requests(self):
        # 生成动态 URL
        for i in range(1, 100):
            yield scrapy.Request(
                f'https://example.com/page/{i}',
                callback=self.parse,
                meta={'page': i}
            )
```

## 多页爬取技巧

### 方式1：手动翻页

```python
def parse(self, response):
    # 解析当前页
    yield from self.parse_items(response)

    # 翻到下一页
    next_page = response.css('a.next::attr(href)').get()
    if next_page:
        yield response.follow(next_page, callback=self.parse)
```

### 方式2：条件翻页

```python
def parse(self, response):
    yield from self.parse_items(response)

    next_page = response.css('a.next::attr(href)').get()
    # 只爬取前 10 页
    current_page = response.meta.get('page', 1)
    if next_page and current_page < 10:
        yield response.follow(
            next_page,
            callback=self.parse,
            meta={'page': current_page + 1}
        )
```

### 方式3：伪分页（AJAX加载）

有些网站使用 JavaScript 动态加载，需要直接构造请求：

```python
def parse(self, response):
    # 解析当前数据...

def start_requests(self):
    # 直接请求 API
    for page in range(1, 100):
        yield scrapy.Request(
            f'https://api.example.com/items?page={page}&size=20',
            headers={'X-Requested-With': 'XMLHttpRequest'},
            callback=self.parse
        )
```

## 分类爬取（不同解析函数）

```python
class NewsSpider(scrapy.Spider):
    name = 'news'

    def parse(self, response):
        # 列表页
        for news in response.css('div.news-item'):
            yield response.follow(
                news.css('a::attr(href)').get(),
                callback=self.parse_news
            )

        # 分页
        yield response.follow(
            response.css('a.next::attr(href)').get(),
            callback=self.parse
        )

    def parse_news(self, response):
        # 新闻详情页
        yield {
            'title': response.css('h1.title::text').get(),
            'content': response.css('div.article::text').getall(),
            'date': response.css('span.date::text').get(),
        }

    def parse_video(self, response):
        # 视频详情页（如果列表页有视频链接）
        yield {
            'title': response.css('h1.title::text').get(),
            'video_url': response.css('video source::attr(src)').get(),
        }
```

## 处理重定向

```python
# settings.py
# 启用重定向
REDIRECT_ENABLED = True
# 处理重定向的中间件
HTTPCACHE_ENABLED = True

# meta 中禁用重定向
yield scrapy.Request(
    url,
    callback=self.parse,
    dont_filter=True,
    meta={'dont_redirect': True}
)
```

---

# Settings 常用配置

## 基础配置

```python
# settings.py

# 爬虫名称（运行命令用）
BOT_NAME = 'myproject'

# 爬虫模块路径
SPIDER_MODULES = ['myproject.spiders']
NEWSPIDER_MODULE = 'myproject.spiders'

# 是否启用 HTTP 缓存
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 0
HTTPCACHE_DIR = 'httpcache'
```

## 请求配置

```python
# 请求并发数
CONCURRENT_REQUESTS = 16           # 默认 16 个并发请求
CONCURRENT_REQUESTS_PER_DOMAIN = 8 # 每个域名 8 个并发
CONCURRENT_REQUESTS_PER_IP = 8     # 每个 IP 8 个并发（忽略域名限制）

# 下载延迟（秒）
DOWNLOAD_DELAY = 0.25              # 请求间隔，避免被封

# 请求超时（秒）
DOWNLOAD_TIMEOUT = 180            # 默认 180 秒

# 自动限速
AUTOTHROTTLE_ENABLED = True        # 启用自动限速
AUTOTHROTTLE_START_DELAY = 5       # 初始延迟（秒）
AUTOTHROTTLE_MAX_DELAY = 60        # 最大延迟
AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0  # 目标并发数
```

## 重试配置

```python
# 启用重试
RETRY_ENABLED = True
RETRY_TIMES = 3                   # 重试次数
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]  # 重试的状态码

# 429 = Too Many Requests（被限流）
```

## 请求头配置

```python
# 默认请求头
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
    'Referer': 'https://www.example.com/',
}

# 用户代理轮换（需要中间件配合）
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
```

## 中间件配置

```python
# 下载器中间件（数字越小越先执行）
DOWNLOADER_MIDDLEWARES = {
    'myproject.middlewares.RandomUserAgentMiddleware': 400,
    'myproject.middlewares.ProxyMiddleware': 100,
    'myproject.middlewares.CookieMiddleware': 700,
}

# 爬虫中间件
SPIDER_MIDDLEWARES = {
    'myproject.middlewares.MySpiderMiddleware': 543,
}
```

## Pipeline 配置

```python
# 启用的 Pipeline（数字越小越先执行）
ITEM_PIPELINES = {
    'myproject.pipelines.ValidationPipeline': 100,
    'myproject.pipelines.CleanPipeline': 200,
    'myproject.pipelines.MongoDBPipeline': 300,
}

# 图片/文件存储
IMAGES_STORE = 'images/'           # 图片存储目录
FILES_STORE = 'files/'             # 文件存储目录
IMAGES_MIN_WIDTH = 100
IMAGES_MIN_HEIGHT = 100
```

## 日志配置

```python
# 日志级别
LOG_LEVEL = 'INFO'  # DEBUG / INFO / WARNING / ERROR / CRITICAL

# 日志输出到文件
LOG_FILE = 'scrapy.log'           # 保存到文件
LOG_ENCODING = 'utf-8'

# 日志格式（自定义）
LOG_FORMAT = '%(asctime)s [%(name)s] %(levelname)s: %(message)s'
LOG_DATEFORMAT = '%Y-%m-%d %H:%M:%S'
```

## 禁用 Cookie（节省资源）

```python
COOKIES_ENABLED = False           # 禁用 cookie
COOKIES_DEBUG = False             # 记录 cookie 到日志
```

## Telnet 控制台（调试用）

```python
TELNETCONSOLE_ENABLED = True      # 启用 telnet 终端
TELNETCONSOLE_PORT = [6023, 6024] # 端口范围
TELNETCONSOLE_HOST = '127.0.0.1'
```

## 关闭 Robots.txt 检查

```python
ROBOTSTXT_OBEY = False           # 忽略 robots.txt 协议
```

## 常见配置组合

### 高并发快速爬取（慎用）

```python
CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 16
DOWNLOAD_DELAY = 0.1
RETRY_TIMES = 1
ROBOTSTXT_OBEY = False
COOKIES_ENABLED = False
```

### 低并发稳定爬取（防封禁）

```python
CONCURRENT_REQUESTS = 4
CONCURRENT_REQUESTS_PER_DOMAIN = 2
DOWNLOAD_DELAY = 2
AUTOTHROTTLE_ENABLED = True
RETRY_TIMES = 5
ROBOTSTXT_OBEY = True
```

### 生产环境配置

```python
# settings.py

BOT_NAME = 'production_spider'

# 禁用调试功能
TELNETCONSOLE_ENABLED = False
MEMUSAGE_ENABLED = True          # 监控内存使用

# 日志
LOG_LEVEL = 'WARNING'
LOG_FILE = f'spider_{datetime.now().strftime("%Y%m%d")}.log'

# Pipeline
ITEM_PIPELINES = {
    'myproject.pipelines.ValidationPipeline': 100,
    'myproject.pipelines.MongoDBPipeline': 300,
}

# 请求限制
CONCURRENT_REQUESTS = 8
DOWNLOAD_DELAY = 1
RETRY_TIMES = 3
AUTOTHROTTLE_ENABLED = True
```

## 命令行覆盖配置

运行爬虫时可以通过 `-s` 参数覆盖 settings：

```bash
# 临时修改并发数
scrapy crawl quotes -s CONCURRENT_REQUESTS=32

# 临时修改日志级别
scrapy crawl quotes -s LOG_LEVEL=DEBUG

# 临时禁用管道
scrapy crawl quotes -s ITEM_PIPELINES={}
```

---

## 章节小结

| 组件 | 作用 | 关键点 |
| --- | --- | --- |
| **Item** | 定义数据结构 | 类型提示、字段验证 |
| **ItemLoader** | 批量加载数据 | Input/Output Processor |
| **Pipeline** | 处理 Item | 验证、清洗、去重、存储 |
| **Spider** | 核心解析逻辑 | follow_all、meta、cb_kwargs |
| **Settings** | 全局配置 | 并发、延迟、重试、中间件 |