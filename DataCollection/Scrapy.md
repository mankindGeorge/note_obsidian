
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

## 章节小结

| 组件 | 作用 | 关键点 |
| --- | --- | --- |
| **Item** | 定义数据结构 | 类型提示、字段验证 |
| **ItemLoader** | 批量加载数据 | Input/Output Processor |
| **Pipeline** | 处理 Item | 验证、清洗、去重、存储 |