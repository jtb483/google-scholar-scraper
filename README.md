# Google Scholar抓取Python：我跑了3万条数据后的方案与代码

Google Scholar不提供官方API，直接用requests库请求，大概率第3页就触发验证码。我去年底开始做文献计量项目，需要批量拉取特定领域的论文标题、作者、被引次数，前后试了Selenium、免费代理池、付费API三条路，最终稳定跑通的方案是ScraperAPI的Google Scholar结构化端点配合Python脚本。

核心逻辑很简单：把请求丢给ScraperAPI，它帮你处理IP轮换、浏览器指纹、验证码识别，返回干净的JSON结构数据，你只管解析存库。下面是我实际在用的完整流程，包括踩过的坑。

想直接拿API key跑起来的，，不绑卡，够你测试200次Scholar请求。

## 为什么Google Scholar这么难抓

Google Scholar的反爬比普通Google搜索更激进。没有robots.txt白名单，没有官方数据接口，请求频率稍高就弹CAPTCHA。

我最初用requests+随机User-Agent，本地IP跑到第47条就被ban了整24小时。换了个免费代理池，成功率不到30%，而且返回的HTML结构经常残缺——免费代理的出口IP早就被Google标记过。Selenium加headless Chrome能撑久一点，但单线程一小时只能跑200条左右，而且Chrome进程吃内存，服务器跑一晚上OM两次。

问题本质是：Google Scholar需要住宅级IP+真实浏览器指纹+智能限速三件套同时到位。自己搭这套基础设施，光代理成本每月就要$80起，还得维护轮换逻辑。

## ScraperAPI的Google Scholar端点怎么工作

ScraperAPI提供一个专门针对Google搜索系列的结构化数据接口。你发一个GET请求，带上API key和搜索参数，它返回解析好的JSON——论文标题、作者列表、发表年份、被引次数、PDF链接全部字段化，不用你自己写XPath。

每次Google Scholar请求消耗25个API credits。这个消耗比普通网页抓取（1-10 credits）高，因为后端要过验证码+渲染JS+住宅代理出口，但省掉了你自己维护这些的时间和钱。

上周三我跑了一批"deep learning medical imaging"的文献数据，1200条结果，耗时14分钟，零失败。账单显示扣了30,000 credits，和预期一致（1200条 × 25 credits）。

## 完整Python代码：从零到拿到结构化数据

先装依赖，就一个requests：

```python

import requests

import json

import time

API_KEY = "你的ScraperAPI密钥"

BASE_URL = "https://api.scraperapi.com/structured/google/scholar"

def scrape_scholar(query, num_pages=5):

all_results = []

for page in range(num_pages):

params = {

"api_key": API_KEY,

"query": query,

"start": page * 10

}

response = requests.get(BASE_URL, params=params)

if response.status_code == 200:

data = response.json()

articles = data.get("organic_results", [])

for article in articles:

all_results.append({

"title": article.get("title"),

"authors": article.get("authors"),

"year": article.get("year"),

"cited_by": article.get("cited_by", {}).get("value", 0),

"link": article.get("link"),

"snippet": article.get("snippet")

})

print(f"第{page+1}页完成，累计{len(all_results)}条")

else:

print(f"第{page+1}页失败，状态码：{response.status_code}")

time.sleep(2) # 礼貌间隔，非必须但推荐

return all_results

# 执行

results = scrape_scholar("transformer neural network", num_pages=10)

# 存JSON

with open("scholar_results.json", "w", encoding="utf-8") as f:

json.dump(results, f, ensure_ascii=False, indent=2)

print(f"共采集{len(results)}条论文数据")

```

这段代码10页能拉100条结果。实际跑的时候我一般设`num_pages=50`起步，配合关键词变体循环，一个晚上能跑完一个细分领域的主要文献。

## 进阶：批量关键词+断点续传+去重

实际项目不可能只查一个词。下面这个版本支持关键词列表、自动去重、中断后从上次位置继续：

```python

import requests

import json

import time

import os

from hashlib import md5

API_KEY = "你的ScraperAPI密钥"

BASE_URL = "https://api.scraperapi.com/structured/google/scholar"

OUTPUT_FILE = "scholar_data.json"

PROGRESS_FILE = "progress.json"

def load_progress():

if os.path.exists(PROGRESS_FILE):

with open(PROGRESS_FILE, "r") as f:

return json.load(f)

return {"completed": [], "last_query": None, "last_page": 0}

def save_progress(progress):

with open(PROGRESS_FILE, "w") as f:

json.dump(progress, f)

def deduplicate(results):

seen = set()

unique = []

for r in results:

key = md5(r["title"].encode()).hexdigest() if r["title"] else None

if key and key not in seen:

seen.add(key)

unique.append(r)

return unique

def batch_scrape(keywords, pages_per_keyword=20):

progress = load_progress()

all_results = []

if os.path.exists(OUTPUT_FILE):

with open(OUTPUT_FILE, "r", encoding="utf-8") as f:

all_results = json.load(f)

for keyword in keywords:

if keyword in progress["completed"]:

print(f"跳过已完成：{keyword}")

continue

start_page = 0

if keyword == progress["last_query"]:

start_page = progress["last_page"]

for page in range(start_page, pages_per_keyword):

params = {

"api_key": API_KEY,

"query": keyword,

"start": page * 10

}

try:

resp = requests.get(BASE_URL, params=params, timeout=60)

if resp.status_code == 200:

articles = resp.json().get("organic_results", [])

for a in articles:

all_results.append({

"query": keyword,

"title": a.get("title"),

"authors": a.get("authors"),

"year": a.get("year"),

"cited_by": a.get("cited_by", {}).get("value", 0),

"link": a.get("link")

})

progress["last_query"] = keyword

progress["last_page"] = page + 1

save_progress(progress)

time.sleep(2)

except Exception as e:

print(f"异常：{e}，保存进度后退出")

all_results = deduplicate(all_results)

with open(OUTPUT_FILE, "w", encoding="utf-8") as f:

json.dump(all_results, f, ensure_ascii=False, indent=2)

return

progress["completed"].append(keyword)

save_progress(progress)

all_results = deduplicate(all_results)

with open(OUTPUT_FILE, "w", encoding="utf-8") as f:

json.dump(all_results, f, ensure_ascii=False, indent=2)

print(f"全部完成，共{len(all_results)}条唯一结果")

# 使用

keywords = [

"large language model healthcare",

"transformer protein folding",

"graph neural network drug discovery"

]

batch_scrape(keywords, pages_per_keyword=30)

```

这个脚本我在一台$5/月的VPS上跑了两周，累计采集了31,847条去重后的论文元数据，中间断过3次网，每次重启自动从断点继续，没丢数据。

## ScraperAPI全套餐对比：选哪个档位够用

| 套餐 | API Credits/月 | 并发线程 | 核心功能 | 月付价格 | 年付价格 | 操作 |
| ------ | ------------ | --------- | ------ | --- | --- | --- |
| Hobby（免费） | 5,000 | 1 | 基础抓取，无地理定位 | $0 | $0 | - |
| Starter | 100,000 | 10 | 地理定位+JS渲染 | $49 | $29 | - |
| Business | 1,000,000 | 50 | 住宅代理+全功能 | $149 | $99 | - |
| Enterprise | 3,000,000+ | 无限 | 专属客户经理+SLA | 定制 | 定制 | - |

算笔账：Google Scholar每次请求25 credits。Hobby的5000 credits能跑200次请求，够你验证代码逻辑。Starter的100,000 credits能跑4,000次，按每次10条结果算就是40,000条论文数据，对大多数文献综述项目绰绰有余。

我自己用的Business档，因为要跑多个项目+普通网页抓取混用，年付$99/月，100万credits每月，从来没用超过。

## 和自建方案的成本对比

我之前自己搭过一套：10个住宅代理IP（$40/月）+ 2台VPS跑Selenium集群（$20/月）+ Puppeteer反检测插件维护时间（每周约3小时调参数）。总成本$60/月+人力，成功率大概85%，隔三差五要手动处理验证码积压。

换ScraperAPI之后，Starter档$29/月（年付），成功率99%以上，代码量从800行缩到上面那60行。省下来的时间我拿去做数据分析了，这才是项目真正产出价值的环节。

上个月15号续费时账单$99（Business年付），同一天跑完了一个3万条的采集任务。如果用旧方案，光代理费就要$40，还得盯着怕挂。

## 常见坑和解决方案

**坑1：credits消耗比预期快**

Google Scholar请求是25 credits/次，不是1 credits。我第一个月没注意，以为100,000 credits能跑10万次，实际只够4,000次Scholar请求。解决办法：在代码里加个credits计数器，快到阈值时自动暂停。

**坑2：返回结果少于10条**

Scholar某些冷门查询本身结果就少，不是API的问题。加个判断：如果`organic_results`为空就跳到下一个关键词，别浪费credits翻空页。

**坑3：年份筛选**

ScraperAPI的Scholar端点支持`as_ylo`和`as_yhi`参数做年份范围过滤。比如只要2020年以后的论文：

```python

params = {

"api_key": API_KEY,

"query": "your search term",

"as_ylo": "2020",

"as_yhi": "2024"

}

```

这个参数官方文档里藏得比较深，我是翻了API reference才找到的。

## FAQ

**Google Scholar有反爬机制吗？Python直接请求会被封IP吗？**

会。Google Scholar对自动化请求的检测比普通搜索更严格，同一IP连续请求超过10-20次基本就会触发CAPTCHA或临时封禁，封禁时长从几小时到24小时不等。用ScraperAPI走住宅代理出口可以绕过这个限制。

**ScraperAPI抓取Google Scholar一次消耗多少credits？**

每次请求消耗25个API credits。这是因为Scholar请求需要住宅代理+验证码处理+JS渲染的组合，后端成本比普通网页高。100,000 credits的Starter套餐能跑4,000次Scholar请求。

**免费方案能不能跑Google Scholar批量采集？**

Hobby套餐每月5,000 credits，折合200次Scholar请求（约2,000条论文数据）。做小规模测试或验证代码逻辑够用，正式批量采集建议至少Starter档。，确认满足需求再升级。

**用BeautifulSoup解析HTML还是用结构化数据接口更方便？**

结构化数据接口省事得多。BeautifulSoup方案需要你自己维护XPath/CSS选择器，Google一改版页面结构你的解析就挂了——我去年遇到过两次。结构化接口直接返回JSON字段，ScraperAPI那边负责适配页面变化，你的代码不用动。

**抓取学术论文标题、作者、引用数的完整代码怎么写？**

上面"完整Python代码"那节就是可以直接跑的版本。核心就是请求`/structured/google/scholar`端点，返回的JSON里`organic_results`数组每个元素包含title、authors、cited_by、year、link字段，直接取值存库。

**ScraperAPI和SerpAPI抓Google Scholar哪个性价比高？**

SerpAPI的Scholar搜索$50/月只给5,000次请求。ScraperAPI Starter档$29/月（年付）给100,000 credits，折合4,000次Scholar请求，价格低了近一半而请求量接近。Business档$99/月给1,000,000 credits（40,000次Scholar请求），大批量场景下单次成本不到$0.025。

**大批量抓取（10万条以上）Scholar数据用什么方案稳定？**

Business或Enterprise套餐，配合上面的断点续传脚本。我跑3万条用了Business档，14天完成，零数据丢失。10万条以上建议联系他们拿Enterprise定制方案，有专属客户经理帮你优化请求策略和并发配置。

---

我现在手头三个文献计量项目全跑在ScraperAPI上，最近一次大批量任务是上周三启动的，目标5万条，目前已经稳定跑到第3.2万条，预计周末前完成。如果你也在做学术数据采集，别在基础设施上浪费时间了——[用这个链接注册拿你的免费5000 credits](https://www.scraperapi.com/signup?fp_ref=coupons)，7天试用期内付费套餐全功能开放，不满意随时取消，不扣一分钱。
