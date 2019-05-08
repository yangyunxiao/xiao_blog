---
layout:     post
title:      "Scrapy 笔记"
subtitle:   "应用scrapy爬虫框架，轻松爬取网站"
date:       2019-05-06
author:     "yunxiao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Python
    - 爬虫
---


```
public static void main (String[] args){
    System.out.println("hello world!")
}
```

---

<p style="font-size:24px;font-weight:800">Scrapy爬虫练习</p>

<p>开发环境的搭建</p>
**pipenv**  

pip 使用豆瓣源安装 scrapy

`pip install -i https://pypi.douban.com/simple/ scrapy `

创建Scrapy项目   
`scrapy startproject ArtcleSpider`  

创建网站爬虫模板  
`scrapy genspider jobber blog.jobble.com` 
 
生成如下代码
```python
import scrapy


class JobberSpider(scrapy.Spider):
    name = 'jobber'
    allowed_domains = ['blog.jobber.com']
    start_urls = ['http://blog.jobber.com/']

    def parse(self, response):
        pass

```
pip问题降级  
```
python -m pip install pip==18.0

#其中，-m参数的解释：
run library module as a script (terminates option list)
将库中的python模块用作脚本去运行。
```

<p>settings.py中取消robot协议防止爬虫终止  </p>

```python
# Obey robots.txt rules
ROBOTSTXT_OBEY = False
```
###Xpath使用方法

表达式|说明
:---:|:---
nodename | 选取此节点的所有节点
/ | 从根节点选取
// | 从匹配选择的当前节点选择文档中的节点，而不考虑他们的位置
.|选取当前节点
..|选取当前节点的父节点
@|选取属性

使用scrapy  shell  url 终端分析网页
```
 title = response.xpath('//*[@id="post-107390"]/div[1]/h1/text()')
 text()取出值
```


###CSS选择器
...


##Scrapy Item

###Pipelines 代表数据流处理管道
```python
# 取消注释 数字代表管道处理的优先级  数字越小  则优先级越高
# Configure item pipelines
# See https://doc.scrapy.org/en/latest/topics/item-pipeline.html
ITEM_PIPELINES = {
   'ArticleSpider.pipelines.ArticlespiderPipeline': 300,
   'scrapy.pipelines.images.ImagesPipeline':50,
}
```

爬去数据录入数据库
```
pipenv install pymysql
```

配置mysql异步插入 提高解析速度   
mysql的配置信息在setting文件中
```python
#setting.py 文件中
MYSQL_HOST = ''
MYSQL_DBNAME = ''
MYSQL_USER = ''
MYSQL_PASSWORD = ''
MYSQL_PORT = 3306

from twisted.enterprise import adbapi 
import pymysql
#使用twisted实现异步录入
class MysqlTwistedPipeline(object):
    def __init__(self,dbpool):
        self.dbpool = dbpool
        
    #此方法会被自动调用  将setting文件中的配置信息传入读取
    @classmethod
    def from_setting(cls,settings):
        host = settings['MYSQL_HOST']
        db = settings['MYSQL_DBNAME']
        user = settings['MYSQL_USER']
        password = settings['MYSQL_PASSWORD']
        port = settings['MYSQL_PORT']
        
        db_params = dict(
            host = host,
            db = db,
            user = user,
            password = password,
            charset = 'utf-8',
            cursorclass = pymysql.cursors.DictCursor,
            use_unicode = True
        )
        
        dbpool = adbapi.ConnectionPool('MySQLdb',**db_params)
        return cls(dbpool)
        
    def process_item(self,item,spider):
        # 使用twisted将mysql插入变成异步执行
        query = self.dbpool.runInteraction(self.do_insert,item)
        #处理异常
        query.addErrback(self.handle_error)
        
    def handle_error(self,failure):
        #处理异常
        print(failure)
        
    def do_insert(self,cursor,item):
        pass
```

###Item Loader机制 
封装匹配规则 及对提取出的字段数据做特殊处理
```python
from scrapy.loader import ItemLoader
import scrapy
from scrapy.loader.processors import MapCompose,TakeFirst
from ArticleSpider.items import ArticleDetailItem
def parser_detail(response):
    # article_item = ArticleDetailItem()
    item_loader = ItemLoader(item = ArticleDetailItem(),response=response)
    item_loader.add_css("title",".entry-header h1::text")
    item_loader.add_value("url",response.url)
    article_item = item_loader.load_item()
    yield article_item 
    
#使用 MapCompose 可以传入多个处理函数  会依次调用
title = scrapy.Field(
    input_processor=MapCompose(lambda x : x + "prefix"),
    output_processor = TakeFirst() 
)
``` 


#知乎爬虫
####知乎的登录  
selenium 安装  pip install selenium  
安装浏览器驱动 
```python
from selenium import webdriver
from scrapy.selector import Selector

browser = webdriver.Chrome(executable_path="驱动路径")
browser.get("url")

#动态加载完成之后的网页内容
page_source = browser.page_source

selector  = Selector(text=page_source)

```

####模拟知乎登录
```python
#兼容模式导入python2或python3
try:
    import cookielib
except:
    import http.cookiejar as cookiejar
    
import requests
    

def get_xsrf():
    response = requests.get("https://www.zhihu.com",headers=header)
```

requests和requests.session()区别  session 复用链接不需要再次建立连接了

新建知乎爬虫项目 scrapy genspider zhihu www.zhihu.com   

```bash
scrapy shell -s USER-AGENT="" url 
#设置user-agent
```

####数据表设计

爬取代码
```python
try:
    import urlparse as parse
except:
    from urllib import parse 
    
from scrapy import Request

import scrapy
import re

class ZhihuSpider(scrapy.Spider):
    
    def parse(self,response):
        post_urls = response.css("a:attr(href)").extract()
        post_urls = [parse.urljoin(response.url,url) for url in post_urls if url.startWith("https://")]
        # post_urls = filter(lambda x : True if x.startswith("https://") else False,post_urls)
        
        for url in post_urls:
            match_obj =  re.match(r"(.*zhihu.com/question/(\d+))($|/).*",url)
            if match_obj:
                question_url = match_obj.group(1)
                question_id = match_obj.group(2)
                
            yield Request(url=url,callback=self.parse_question)
    def parse_question(self,response):
        pass
```
安装本地软件包 "pip install file_path" 


#招聘网站整站爬取

以上爬虫使用的都是默认basic模板
```bash
#列出所有可用scrapy模板
scrapy genspider --list
Available templates:
  basic
  crawl
  csvfeed
  xmlfeed

#指定scrapy模板生成爬虫项目
scrapy genspider -t crawl lagou www.lagou.com
```

####Crawl爬取流程分析
+ 继承自CrawlSpider，默认复写了parse方法，因此我们不能再将parse作为默认的解析函数了
```python

import copy
import six

from scrapy.http import Request, HtmlResponse
from scrapy.utils.spider import iterate_spider_output
from scrapy.spiders import Spider

class CrawlSpider(Spider):
    rules = ()

    def __init__(self, *a, **kw):
        super(CrawlSpider, self).__init__(*a, **kw)
        #初始编译rule规则
        self._compile_rules()
    
    
    def _compile_rules(self):
        def get_method(method):
            if callable(method):
                return method
            elif isinstance(method, six.string_types):
                return getattr(self, method, None)
        #循环遍历rule规则
        self._rules = [copy.copy(r) for r in self.rules]
        for rule in self._rules:
            #解析回到函数
            rule.callback = get_method(rule.callback)
            rule.process_links = get_method(rule.process_links)
            rule.process_request = get_method(rule.process_request)
    
    #被覆写了默认处理解析函数
    def parse(self, response):
        return self._parse_response(response, self.parse_start_url, cb_kwargs={}, follow=True)


    #解析入口
    def _parse_response(self, response, callback, cb_kwargs, follow=True):
        #从start_url开始爬取时没有回调函数是parse_start_url [] 跳过 
        if callback:
            cb_res = callback(response, **cb_kwargs) or ()
            cb_res = self.process_results(response, cb_res)
            for requests_or_item in iterate_spider_output(cb_res):
                yield requests_or_item
        
        if follow and self._follow_links:
            for request_or_item in self._requests_to_follow(response):
                yield request_or_item
                
    #抽取页面中符合规则的url 比对所有url规则
    def _requests_to_follow(self, response):
        if not isinstance(response, HtmlResponse):
            return
        seen = set()
        #这里很nice 将rule的序号取出来 通过_build_request传递出去
        for n, rule in enumerate(self._rules):
            links = [lnk for lnk in rule.link_extractor.extract_links(response)
                     if lnk not in seen]
            if links and rule.process_links:
                links = rule.process_links(links)
            for link in links:
                seen.add(link)
                #符合规则的url创建Request
                r = self._build_request(n, link)
                yield rule.process_request(r)
                
    #符合规则的url创建Request
    def _build_request(self, rule, link):
        #设置回调  rule为序号
        r = Request(url=link.url, callback=self._response_downloaded)
        r.meta.update(rule=rule, link_text=link.text)
        return r

    def _response_downloaded(self, response):
        #通过rule在rules中的序号 定位到response对应的callback等信息
        rule = self._rules[response.meta['rule']]
        #再次进入循环  callback 没有设置的仅作为匹配页面不做提取操作
        return self._parse_response(response, rule.callback, rule.cb_kwargs, rule.follow)


                
    def parse_start_url(self, response):
        return []

    def process_results(self, response, results):
        return results



    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        spider = super(CrawlSpider, cls).from_crawler(crawler, *args, **kwargs)
        spider._follow_links = crawler.settings.getbool(
            'CRAWLSPIDER_FOLLOW_LINKS', True)
        return spider

    def set_crawler(self, crawler):
        super(CrawlSpider, self).set_crawler(crawler)
        self._follow_links = crawler.settings.getbool('CRAWLSPIDER_FOLLOW_LINKS', True)
```

##scrapy 运行机制

##### spider,engine,pipeline,downloader,scheduler五个重要组件，另外还有middleware

##### spider: 爬虫入口逻辑定义处，包括返回Response解析逻辑，开始url等条件定义  
##### engine: 中转engine  
##### pipeline: item处理逻辑  
##### downloader: 下载网页解析，并返回response  
##### scheduler: 调度器

spider 发送request到 engine 转发到scheduler返回Request到engine，engine发送Request到下载器downloader，解析返回Response到engine，
engine返回到spider中调用用户解析逻辑，如果yield Item到engine 则发送到pipeline处理，如果是Request则发送到engine重复上述逻辑

#爬虫和反爬虫
发爬虫误伤举例封禁ip这种方法是不合适的 比如一个学校可能只有一个ip  如果将此ip封禁  那么这个学校都不能访问网站了

middleware

###selenium
不加载图片  
```python
#固定写法
from selenium import webdriver

chrome_opt = webdriver.ChromeOptions()
prefs = {'profile.managed_default_content_settings.images':2}
chrome_opt.add_experimental_option("prefs",prefs)
browser = webdriver.Chrome(executable_path="path")
browser.get("url")
```

###plantomjs 无界面的浏览器
+ 在Linux服务器当中使用
+ 在多进程情况下性能会下降很严这功能

将Selenuim继承到Scrapy当中

在middleware中
```python
import scrapy
from selenium import webdriver

from scrapy.http import HtmlResponse

#引入信号机制  处理浏览器退出问题
from scrapy import signals 
from scrapy.xlib.pydispatch import dispatcher

class JobboleSpider(scrapy.Spider):
    def __init__(self):
        self.browser =  webdriver.Chrome(executable_path="path")
        super(JobboleSpider,self).__init__()
        dispatcher.connect(self.spider_closed,signals.spider_closed)

    def spider_closed(self,spider): 
        #爬虫退出时关闭浏览器
        self.browser.quit()
class MiddlewareSelenium(object):

    def process_request(self,request,spider):
        if spider.name == "jobbole":
            spider.browser.get(request.url)
            import time 
            time.sleep(3)
            return HtmlResponse(url=spider.current_url,body=spider.page_source)
```

###无界面运行chrome
```bash
pipenv install PyVirtualDisplay
```

```python
from pyvirtualdisplay import Display
from selenium import webdriver

display = Display(visible=0,size=(800,600))
display.start()

#之后操作与有界面一致 ...
browser =  webdriver.Chrome(executable_path="path")

```

### Scrapy 暂停与重启
```bash
crawl spider lagou -s JOBDIR=job_info/001   # -s  set指令
#scrapy的启动与暂停接受的是Ctrl+C命令  IDE是不会发出这个命令的 Ctrl+C如果连续按两次同样是强制结束 不会善后处理 
kill -f main.py  
#杀死进程之前会发送一个中断信号  Scrapy会接受到这个信号并做善后处理
kill -f -9 main.py
#强制杀死进程   不会做善后处理 程序接收不到中断信号 相当于windows任务管理器强制关闭进程 直接关闭
```

### 分布式爬虫
多台服务器节点共同进行爬取，为了统一管理爬取任务，同步爬取进度，需要有一个节点做状态管理，协调多台服务器爬取  
多服务器节点爬取充分利用多个ip加快爬取速度

####redis的使用  
> 内存数据库  


##ElasticSearch搜索引擎
> 基于Java开发  分布式 基于RESTFUL接口  
elasticsearch-rtf中文发行版  
head插件(相当于phpMyAdmin) 和 kibana的安装  
使用elasticsearch-head连接 elasticsearch默认是连接不上的 因为ES默认禁止第三方插件连接，
需要更改配置文件添加：  
http.cors.enabled: true  
http.cors.allow-origin: "*"  
http.cors.allow-methods: OPTIONS,HEAD,GET,POST,PUT,DELETE  
http.cors.allow-headers: "X-Requested-With,Content-Type,Content-Length,X-User"  

####elasticsearch 本身是集合了数据库的搜索引擎，它和mysql的区别对比如下
elasticsearch|mysql
---|---
index(索引) | 数据库
type(类型) | 表
document(文档) | 行
fields | 列

####倒排索引
>



####关系型数据库和Nosql(Not only Sql)
> Mongodb Redis都是nosql数据库

###kinbana创建索引
```
PUT lagou
{
  "settings": {
    "index":{
      "number_of_shards":5,
      "number_of_replicas":1
    }
  }
}

GET lagou/_settings

GET _all/_settings

#插入文档（一行数据）
PUT lagou/job/1
{
  "title":"python 开发工程",
  "salary_min":15000,
  "city":"beijing",
  "company":{
    "name":"百度 ",
    "company_addr":"北京市海淀区西二旗百度软件园",
    "employee_number":12345
  },
  "publish_date":"2017-4-16",
  "comments":15
}

#插入文档（一行数据）不指定ID  自动生成
POST lagou/job/
{
  "title":"Android开发 开发工程",
  "salary_min":12000,
  "city":"北京",
  "company":{
    "name":"美团",
    "company_addr":"北京市海淀区西二旗",
    "employee_number":12342
  },
  "publish_date":"2017-4-14",
  "comments":15
}





#获取id为1的文档
GET lagou/job/1

#获取指定字段数据
GET lagou/job/1?_source=title,city


#更新字段
POST lagou/job/1/_update
{
  "doc":{
    "company":{"employee_number":12344}
  }
}
```
查询  match查询  
``` 
GET lagou/job/_search
{
    "query":{
        "match":{
            "title":"python"
        }
    }
}
#会自动分词  且大小写都可以

```

term查询  不会对查询的输入条件做解析的  只能全部匹配才能搜索到
``` 
GET lagou/_search
{
    "query"{
        "term":{
            "title":"python"
        }
    }
}
```

terms查询 
``` 
GET lagou/_search
{
    "query"{
        "terms":{
            "title":["python","python","python"]
        }
    }
}
```


``` 
#select * from testjob where title "python" or (title ="django" AND salary=30)

GET lagou/testjon/_search
{
    "query":{
        "bool":{
            "should":[
                {
                    "term":{"title":"python"}
                },
                {
                    bool:{
                        "must":[
                            {"term":{"title":"elasticsearch"}},
                            {"term":{"salary":30}}
                        ]
                    }
                }
            ]
        }
    }
}
```

###如何将Scrapy爬取到的数据写入到ElasticSearch中 
安装elasticsearch-dsl


###Django搭建后台 
