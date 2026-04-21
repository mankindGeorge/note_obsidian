
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