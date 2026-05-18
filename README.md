# Python Zillow 抓取教程：从零搭建房产数据采集脚本，绕过反爬封锁的完整方案

我去年开始做美国房产数据分析的副业项目，第一个要啃的硬骨头就是 Zillow。说真的，Zillow 的反爬机制比我预想的凶猛得多——IP 封禁、JavaScript 渲染、验证码弹窗，基本上裸跑 requests 库撑不过 20 个请求就会被 ban。折腾了两周后我才摸到一套稳定的方案，今天把整个流程拆开讲清楚。

👉 [立即获取 ScraperAPI 5000 次免费请求额度](https://www.scraperapi.com/?fp_ref=coupons)

## 为什么 Zillow 是公认最难抓的房产网站之一

Zillow 的技术栈对爬虫极不友好。页面大量依赖客户端 JavaScript 渲染，核心房源数据藏在动态加载的 JSON 里，而不是静态 HTML 中。更麻烦的是它的反爬策略组合拳：

- **行为指纹检测**：检鼠标轨迹、滚动模式，判断是否为真人浏览
- **IP 频率限制**：同一 IP 短时间内请求超过阈值直接返回 403 或 CAPTCHA
- **请求头校验**：缺少合理的 User-Agent、Referer 等头部信息会被即时拦截
- **动态 Token**：部分 API 端点需要页面内生成的临时 token 才能访问

我最初用 Selenium 硬刚，速度慢到离谱，一个城市的房源列表跑完要好几个小时。后来换了代理 API 的思路，效率直接翻了几十倍。

## 方案选型：三种主流抓取路径对比

### 路径一：Selenium / Playwright 模拟浏览器

优点是能完整渲染 JavaScript，缺点是资源消耗大、速度慢、容易被指纹检测识别。适合小规模验证，不适合批量采集。

### 路径二：逆向 Zillow 内部 API

Zillow 的搜索结果实际上走的是一个 GraphQL 接口。你可以在浏览器 DevTools 的 Network 面板里找到类似 `https://www.zillow.com/search/GetSearchPageState.htm` 的请求。直接调用这个接口能拿到结构化 JSON 数据，省去解析 HTML 的麻烦。但问题是——请求头和 cookie 校验很严格，IP 一旦被标记就彻底废了。

### 路径三：代理 API 服务 + 请求库

这是我目前在用的方案。把 IP轮换、请求头伪装、CAPTCHA 处理这些脏活交给专门的代理 API，自己只管写业务逻辑。ScraperAPI 在这块做得比较成熟，它会自动处理 IP 池轮换和浏览器指纹，你只需要把目标 URL 传进去就行。

老实讲，第三种路径的性价比最高。我自己维护代理池的时候，光是买住宅代理 IP 每月就要花 $80+，还得写一堆重试逻辑和异常处理。用 ScraperAPI 之后这些全省了。

## 实战代码：用 Python + ScraperAPI 抓取 Zillow 房源数据

### 环境准备

```python
pip install requests beautifulsoup4 pandas
```

### 基础抓取脚本

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import json
import time

# ScraperAPI 配置
API_KEY = "你的ScraperAPI密钥"
BASE_URL = "http://api.scraperapi.com"

def scrape_zillow_listings(search_url):
    """通过 ScraperAPI 抓取 Zillow 搜索结果页"""
    params = {
        "api_key": API_KEY,
        "url": search_url,
        "render": "true",  # 启用 JavaScript 渲染
        "country_code": "us"
    }
    
    response = requests.get(BASE_URL, params=params)
    if response.status_code == 200:
        return response.text
    else:
        print(f"请求失败，状态码：{response.status_code}")
        return None

def parse_listings(html_content):
    """解析房源列表数据"""
    soup = BeautifulSoup(html_content, "html.parser")
    
    # Zillow 把搜索结果数据嵌在 script 标签的 JSON 里
    script_tag = soup.find("script", {"id": "__NEXT_DATA__"})
    if script_tag:
        data = json.loads(script_tag.string)
        # 根据 Zillow 的数据结构提取房源列表
        search_results = data.get("props", {}).get("pageProps", {}).get("searchPageState", {})
        listings = search_results.get("cat1", {}).get("searchResults", {}).get("list", [])
        return listings
    
    return []

def extract_property_info(listings):
    """提取关键房产字段"""
    properties = []
    
    for listing in listings:
        property_data = {
            "地址": listing.get("address", ""),
            "价格": listing.get("unformattedPrice", ""),
            "卧室数": listing.get("beds", "),
            "浴室数": listing.get("baths", ""),
            "面积(sqft)": listing.get("area", ""),
            "房屋类型": listing.get("hdpData", {}).get("homeInfo", {}).get("homeType", ""),
            "详情链接": listing.get("detailUrl", ""),
            "经纪人": listing.get("brokerName", "")
        }
        properties.append(property_data)
    return properties

# 主流程
if __name__ == "__main__":
    # 示例：抓取旧金山的房源
    zilow_url = "https://www.zillow.com/san-francisco-ca/"
    
    print("正在抓取 Zillow 数据...")
    html = scrape_zillow_listings(zillow_url)
    if html:
        listings = parse_listings(html)
        properties = extract_property_info(listings)
        
        # 导出为 CSV
        df = pd.DataFrame(properties)
        df.to_csv("zillow_listings.csv", index=False, encoding="utf-8-sig")
        print(f"成功抓取 {len(properties)} 条房源数据")
```

### 批量翻页抓取

```python
def scrape_multiple_pages(base_search_url, max_pages=5):
    """批量翻页抓取"""
    all_properties = []
    
    for page in range(1, max_pages + 1):
        # Zillow 翻页参数
        page_url = f"{base_search_url}{page}_/"
        
        print(f"正在抓取第 {page} 页...")
        html = scrape_zillow_listings(page_url)
        if html:
            listings = parse_listings(html)
            properties = extract_property_info(listings)
            all_properties.extend(properties)
        # 控制请求频率，别太激进
        time.sleep(2)
    
    return all_properties
```

### 抓取单个房源详情页

```python
def scrape_property_detail(detail_url):
    """抓取单个房源的详细信息"""
    params = {
        "api_key": API_KEY,
        "url": f"https://www.zillow.com{detail_url}",
        "render": "true",
        "country_code": "us"
    }
    
    response = requests.get(BASE_URL, params=params)
    if response.status_code == 200:
        soup = BeautifulSoup(response.text, "html.parser")
        script_tag = soup.find("script", {"id": "__NEXT_DATA__"})
        
        if script_tag:
            data = json.loads(script_tag.string)
            property_info = data.get("props", {}).get("pageProps", {}).get("componentProps", {})
            # 提取详细字段
            gdp_data = property_info.get("gdpClientCache", {})
            # gdp_data 里包含完整的房产信息：历史价格、税务记录、学区等
            return gdp_data
    
    return None
```

## 几个容易踩的坑和解决办法

**坑一：render 参数没开**

Zillow 的核心数据靠 JavaScript 渲染。如果你调用 ScraperAPI 时没加 `"render": "true"`，拿回来的 HTML 里大概率是空壳子，看不到任何房源信息。这个参数会消耗更多 API 额度（每次请求算 5 个 credit），但没它真的拿不到数据。

**坑二：地理定位不对**

Zillow 会根据请求来源 IP 的地理位置返回不同内容。加上 `"country_code": "us"` 确保走美国节点，否则可能触发地区限制页面。

**坑三：数据结构变动**

Zillow 隔几个月就会调整前端数据结构。我的建议是把解析逻辑单独封装成函数，结构变了只改一处。另外 `__NEXT_DATA__` 这个 script 标签是Next.js 框架的标准输出，短期内不太会消失，但里面的 JSON 嵌套路径可能变。

**坑四：请求频率过高**

即使用了代理 API，也别一秒钟发 50 个请求。我一般控制在每秒 2-3 个，加上随机延迟。稳定跑比快速跑重要得多。

## ScraperAPI 套餐怎么选：全方案对比

我用 ScraperAPI 主要看中三点：自动 IP 轮换覆盖全球、JavaScript 渲染支持、以及失败请求不扣额度。下面是目前官网在售的所有套餐：

| 套餐名 | API 请求额度 | 并发数 | 价格 | 购买链接 |
| --- | --- | --- | --- | --- |
| Hobby | 100,000 次/月 | 20 并发 | $49/月 | [ 查看 Hobby 套餐详情](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 500,000 次/月 | 50 并发 | $149/月 | [ 查看 Startup 套餐详情](https://www.scraperapi.com/?fp_ref=coupons) |
| Business⭐推荐 | 3,000,000 次/月 | 100 并发 | $299/月 | [ 锁定 Business 套餐官方价格](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义额度 | 自定义并发 | 按需报价 | [ 联系官方获取 Enterprise 方案](https://www.scraperapi.com/?fp_ref=coupons) |

说一下我的选择逻辑：如果你只是做个人项目或者学习用途，免费的 5000 次额度够你跑通整个流程了。正式跑数据的话，Business 套餐的性价比最高——3 百万次请求配 100 并发，抓一个中等城市的全部房源绰有余。

注意一点：开启 JavaScript 渲染的请求每次消耗 5 个 credit，地理定位请求消耗 10-25 个 credit。所以实际可用次数要除以对应倍数来估算。

👉 [先用免费额度跑通你的 Zilow 抓取脚本](https://www.scraperapi.com/?fp_ref=coupons)

## 进阶技巧：提升抓取效率和数据质量

### 用 Async并发提速

```python
import asyncio
import aiohttp

async def async_scrape(session, url):
    params = {
        "api_key": API_KEY,
        "url": url,
        "render": "true",
        "country_code": "us"
    }
    async with session.get(BASE_URL, params=params) as response:
        return await response.text()

async def batch_scrape(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [async_scrape(session, url) for url in urls]
        # 控制并发数，别超过你套餐的并发上限
        semaphore = asyncio.Semaphore(10)
        async def limited_task(task):
            async with semaphore:
                return await task
        
        results = await asyncio.gather(*[limited_task(t) for t in tasks])
        return results
```

### 数据清洗和去重

```python
def clean_data(df):
    """清洗抓取到的原始数据"""
    # 去除重复房源（按地址去重）
    df = df.drop_duplicates(subset=["地址"], keep="first")
    
    # 价格字段转数值
    df["价格"] = pd.to_numeric(df["价格"], errors="coerce")
    
    # 过滤异常值
    df = df["价格"] > 1000]  # 排除明显错误的价格
    df = df[df["面积(sqft)"] > 100]  # 排除面积异常的记录
    
    return df.reset_index(drop=True)
```

### 定时任务监控价格变动

```python
import schedule

def daily_scrape_job():
    """每日定时抓取，监控价格变动"""
    today = time.strftime("%Y%m%d")
    properties = scrape_multiple_pages("https://www.zillow.com/san-francisco-ca/", max_pages=10)
    
    df = pd.DataFrame(properties)
    df["抓取日期"] = today
    df.to_csv(f"zillow_{today}.csv", index=False, encoding="utf-8-sig")

# 每天早上 8 点执行
schedule.every().day.at("08:00").do(daily_scrape_job)
```

## 常见问题解答

### Zillow 允许爬虫抓取数据吗？

Zillow 的 robots.txt 对大部分路径做了限制。从技术角度说，公开页面的数据抓取在很多司法管辖区属于灰色地带。如果你是用于个人研究、学术分析或者小规模的市场调研，风险较低。大规模商业用途建议咨询法律意见。关键是控制频率，别对目标服务器造成负担。

### ScraperAPI 的免费额度够测试用吗？

注册就送 5000 次 API 请求，不需要绑信用卡。对于跑通一个城市的抓取脚本、调试解析逻辑来说完全够用。我当时就是用免费额度把整套代码调通的，确认稳定后才升级付费套餐。

👉 [注册领取 5000 次免费 API 请求](https://www.scraperapi.com/?fp_ref=coupons)

### 开启 JavaScript 渲染后请求很慢怎么办？

渲染模式下单个请求耗时通常在 5-15 秒，这是正常的——毕竟后台要启动无头浏览器完整加载页面。提速的办法是用异步并发，同时发出多个请求。另外，如果你已经逆向出了 Zillow 的 API 端点，可以直接请求 JSON 接口而不开渲染，速度会快很多。

### 抓取过程中遇到 CAPTCHA 怎么处理？

ScraperAPI 内置了 CAPTCHA 自动处理机制。如果它检测到目标页返回了验证码，会自动重试并尝试绕过。从我的使用经验来看，Zillow 的 CAPTCHA 触发率在用了代理 API 之后降到了很低的水平，偶尔遇到也会被自动解决。

### 抓到的数据可以用来做什么？

我见过的用途包括：房产投资分析（比较不同区域的价格走势）、租售比计算、学区房筛选、批量估价模型训练、以及给房产中介做市场报告。数据拿到手之后配合 pandas 和 matplotlib 做可视化分析，或者喂给机器学习模型都行。

### 除了 Zillow 还能抓哪些房产网站？

同一套代码框架稍作修改就能用在 Redfin、Realtor.com、Trulia 这些平台上。ScraperAPI 不限制目标网站，只要你把 URL 传进去就行。不同网站的反爬强度不一样，Zillow 和 Redfin 算是最难的两个。

## 我的实际使用体验

跑了大半年 Zillow 数据采集项目，中间换过三种方案。最早自己搭代理池 + Selenium，维护成本高得离谱，代理 IP 三天两头失效，Selenium 的 Chrome 实例动不动就内存泄漏。后来试过 Scrapy + 免费代理列表，成功率大概只有 30%，根本没法用于生产环境。

最后切到 ScraperAPI 之后稳定多了。它的成功率能到 99% 以上，失败的请求不扣额度这点也很实在。唯一的小缺点是渲染模式下速度确实不算快，但配合异步并发基本能接受。

如果你也在做类似的房产数据项目，我的建议是别在基础设施上浪费时间。代理管理、指纹伪装、CAPTCHA 处理这些事情交给专业工具，自己专注写业务逻辑和数据分析才是正经事。

👉 [用我目前在用的方案开始你的 Zillow 数据采集项目](https://www.scraperapi.com/?fp_ref=coupons)
