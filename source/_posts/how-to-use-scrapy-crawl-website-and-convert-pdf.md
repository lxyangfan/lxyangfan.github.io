title: 使用Scrapy爬取网站并生成PDF
date: 2023-01-15
tags:
- Scrapy
- wkhtmltopdf
- PDF
- 爬虫
- Python
---

# 介绍
我偶然发现了一个技术分享网站，里面的内容我都很喜欢，想把它们都下载下来变成PDF文件学习和收藏。后面有类似的需求也都可以通过这个方法实现，于是就研究起来网页爬虫和网页生成PDF的技术。

test

# 网页生成PDF

网页转PDF有如下方案：
- wkhtmltpdf
- [控制无头浏览器访问网页并导出PDF. ](https://juejin.cn/post/6969538044890185758)

我选择使用基于 wkhtmltopdf 的 Python 库：[pdfkit](https://pypi.org/project/pdfkit/). 原因是可以和爬虫技术无缝结合。它的安装方法如下：

1.  安装 python-pdfkit:

```bash
pip install pdfkit  (or pip3 for python3)
```

2.  安装 wkhtmltopdf:

- Debian/Ubuntu:

```bash
sudo apt-get install wkhtmltopdf
```

- macOS:

```bash
brew install homebrew/cask/wkhtmltopdf
```


- Windows 需要自行[下载](https://wkhtmltopdf.org/downloads.html)二进制安装包

可以写一个小脚本测试一下：
```python
import pdfkit
import os
import logging
logger = logging.getLogger()

options = {
    'page-size': 'A2',
    'encoding': "UTF-8",
    'cookie': [
    ],
}

def generate_pdf(url, dst, options):
    try:
        if os.path.exists(dst):
            logger.warning(f"dst -> [{dst}] already exists, pass it")
            return
        pdfkit.from_url(url, dst, options=options)
    except Exception as e:
        raise e
```

# Scrapy 使用入门

## [Scrapy 安装](https://docs.scrapy.org/en/latest/intro/install.html)

```bash
pip3 install Scrapy
```

## [快速入门](https://docs.scrapy.org/en/latest/intro/tutorial.html)

### 创建 Scrapy 项目

```bash
scrapy startproject your_project_name
```
会创建一个目录`your_project_name`, 目录下是如下结构：

```bash
   scrapy.cfg            # deploy configuration file

    your_project_name/             # project's Python module, you'll import your code from here
        __init__.py

        items.py          # project items definition file

        middlewares.py    # project middlewares file

        pipelines.py      # project pipelines file

        settings.py       # project settings file

        spiders/          # a directory where you'll later put your spiders
            __init__.py
```

### 我们第一个爬虫程序

Scrapy 爬虫程序需要做如下事情：
- 获取需要访问的 URL 列表，生成访问请求 Request
- 访问 Request 获取 Response, 解析它目的是：
	- 生成下一层的访问请求 yield Request
	- 在当前页面解析结果，生成最终的数据结果

```python
import scrapy

cookies = {
    'JSESSIONID': 'XXXX'
}

# 继承 Spider 类，写自己的爬虫程序
class IndexSpider(scrapy.Spider):

	# 取一个名字，启动时需要它
    name = "index"

	# 重载的方法，用于启动爬虫
    def start_requests(self):
        urls = [
            'http://example.com/index/index.html',
        ]
        for url in urls:
        # 生成下一层爬取的请求Request, 并在callback中写明访问成功时的回调
            yield scrapy.Request(url=url, cookies=cookies, callback=self.parse_categories)

    def parse_categories(self, response):
    # 访问成功，解析得出下一层爬取的请求 Request
        for li in response.xpath('//ul/li/a/@href').getall():
            yield scrapy.Request(url=response.urljoin(li), cookies=cookies, callback=self.parse_level2_categories,
            cb_kwargs={'from_url': li})


    def parse_level2_categories(self, response, from_url):
        for a in response.xpath('//a[contains(@class,"archive-article-title")]'):
            text = a.xpath('./text()').get()
            link = a.xpath('./@href').get()
            # 解析html后，生成最终返回的结果
            yield {
                'level1': from_url,
                'level2_categories': response.url,
                'detail_link': response.urljoin(link),
                'detail_title': text
            }
```

之后我们就可以在终端下，运行这个爬虫啦：
```bash
scrapy crawl index -o data.csv
```

- `index` 是我们爬虫类里面写明的`name`.
- `-o` 可以将生成的结果导出到 csv 或者 json格式


### [调试爬虫程序](https://docs.scrapy.org/en/latest/topics/debug.html)

可以使用 Scrapy shell 交互式的访问爬取的网页。
```bash
$ scrapy shell

>>> from scrapy import Request
>>> req = Request('example.com/index.html', cookies={'a':'b'})
>>> fetch(req) # 发起爬虫访问
>>> response.headers # response 是内置对象，可以直接查看
>>> view(response) # 开启一个浏览器，查看返回的响应
```

也可以直接在代码中插入 shell: 
```python
from scrapy.shell import inspect_response

def parse_details(self, response, item=None):
    if item:
        # populate more `item` fields
        return item
    else:
        inspect_response(response, self)
```

## [抓取需要登录的网页](https://scrapeops.io/python-scrapy-playbook/scrapy-login-form/#login-method-1-simple-formrequest)

通过 FormRequest 发起 form 表单提交请求。更复杂的可能需要使用 headless 浏览器发起登录请求，从而获取登录状态。

```python
import scrapy  
from scrapy import Spider  
from scrapy.http import FormRequest  
  
  
class HiddenDataLoginSpider(Spider):  
	name = 'hidden_data_login'  
  
	def start_requests(self):  
		login_url = 'http://quotes.toscrape.com/login'  
		return scrapy.Request(login_url, callback=self.login)  
  
	def login(self, response):  
		token = response.css("form input[name=csrf_token]::attr(value)").extract_first()  
		return FormRequest.from_response(response,  
		formdata={'csrf_token': token,  
		'password': 'foobar',  
		'username': 'foobar'},  
		callback=self.start_scraping)  
  
	def start_scraping(self, response):  
		## Insert code to start scraping pages once logged in  
		pass
```

# 总结
通过 Scrapy 获取需要爬取的网页地址，和登录状态后，就可以结合中 generate_pdf 方法就可以将网页生成PDF.
