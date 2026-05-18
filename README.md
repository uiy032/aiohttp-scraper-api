# aiohttp scraper api：用 ScraperAPI 搞定反爬的完整实战指南

我折腾 aiohttp 写爬虫大概三年了，最让我头疼的从来不是代码逻辑，而是 IP 被封、验证码弹窗、Cloudflare 五秒盾。自己搭代理池？维护成本高得离谱，三天两头掉节点。后来我把代理层整个外包给 ScraperAPI，aiohttp 负责并发调度，ScraperAPI 负责绑定 IP 轮换和浏览器指纹，两边各干各的活，效率直接翻了一个量级。

这篇文章就是我这套组合拳的完整拆解——从 aiohttp 怎么接 ScraperAPI、并发怎么控制、到不同套餐怎么选，踩过的坑全写在里面。

👉 [去 ScraperAPI 官网看完整套餐和免费额度](https://www.scraperapi.com/?fp_ref=coupons)

## 为什么 aiohttp + ScraperAPI 是个好搭配

aiohttp 的异步架构天生适合高并发请求。一个事件循环里同时跑几百个请求，CPU 几乎不怎么吃力。但问题来了：并发越高，目标站点封你越快。

ScraperAPI 的思路很简单——你把目标 URL 丢给它的端点，它帮你处理代轮换、请求头伪装、JavaScript 渲染、CAPTCHA 破解。你拿到的就是干净的 HTML。对 aiohttp 来说，它只是在请求一个固定的 API 地址，完全不用关心底层代理逻辑。

我实测过一个电商价格监控项目，用裸 aiohttp 直连目标站，跑到第 200 个请求就开始大面积 403。换成 ScraperAPI 之后，同样的并发量，5000 个请求跑完成功率在 98% 以上。

## 实际接入代码：三种姿势

### 姿势一：最简单的 GET 请求

```python
import aiohttp
import asyncio

API_KEY = "你的ScraperAPI密钥"
BASE_URL = "http://api.scraperapi.com"

async def fetch(session, target_url):
    params = {
        "api_key": API_KEY,
        "url": target_url,
        "render": "false"
    }
    async with session.get(BASE_URL, params=params) as resp:
        return await resp.text()

async def main():
    urls = ["https://example.com/page1", "https://example.com/page2"]
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, u) for u in urls]
        results = await asyncio.gather(*tasks)
    for html in results:
        print(html[:200])

asyncio.run(main())
```

够直白吧。`render=false` 是静态页面用的，省额度。

### 姿势二：需要 JS 渲染的 SPA 页面

把 `render` 改成 `"true"` 就行。ScraperAPI 会用无头浏览器帮你执行 JavaScript，返回渲染后的 DOM。代价是每个请求消耗的额度翻 10 倍，所以别无脑开。

### 姿势三：用代理端口模式（适合老项目迁移）

如果你项目里已经有一套代理切换逻辑，不想改请求结构，可以用 ScraperAPI 的代理网关模式：

```python
proxy = f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001"

async def fetch_via_proxy(session, target_url):
    async with session.get(target_url, proxy=proxy) as resp:
        return await resp.text()
```

这样 aiohttp 的代码几乎不用动，只是把代理地址换一下。

## 并发控制：别把自己的额度烧光

ScraperAPI 的并发上限跟你买的套餐挂钩。免费版只有 5 个并发线程，Business 套餐能到 100 个。aiohttp 这边用 `asyncio.Semaphore` 卡一下就行：

```python
sem = asyncio.Semaphore(20)  # 根据你的套餐调整

async def fetch_with_limit(session, url):
    async with sem:
        return await fetch(session, url)
```

我吃过一次亏：没加限流，aiohttp 一口气打了 200 个并发出去，ScraperAPI 那边直接返回 429。额度没少扣，数据没拿到。血亏。

## 套餐怎么选：看你的量级

我把 ScraperAPI 目前在售的所有套餐整理成一张表。价格是月付的情况，年付会便宜不少。

| 套餐名 | 请求额度 | 并发线程数 | 地理定位 | 价格（月付） | 适合谁 | 购买链接 |
| --- | --- | --- | --- | --- | --- | --- |
| Free | 5,000 次/月 | 5 | 有 | $0 | 试水、验证可行性 | [免费注册拿 5000 次额度](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 次/月 | 10 | 有 | $49 | 个人项目、小规模监控 | [开通 Hobby 套餐开始干活](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 500,000 次/月 | 50 | 有 | $149 | 中型爬虫、多站点采集 | [拿下 Startup 套餐解锁 50 并发](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 次/月 | 100 | 有 | $299 | 生产级数据管线、团队协作 | [上Business 套餐跑满 100 并发](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | 有 | 联系销售 | 超大规模、定制需求 | [联系销售定制 Enterprise 方案](https://www.scraperapi.com/?fp_ref=coupons) |

年付的话大概能省 30%-50%，如果确定长期用，年付划算很多。

## 我踩过的几个坑

**坑一：超时设置太短。** ScraperAPI 开了 JS 渲染之后，响应时间可能到 15-20 秒。aiohttp 默认超时 5 分钟倒是够，但如果你自己设了 `timeout=aiohttp.ClientTimeout(total=10)`，渲染请求会大面积超时。我现在统一设 60 秒，渲染请求单独设 120 秒。

**坑二：没处理重试。** ScraperAPI 偶尔会返回 500 或 503，特别是目标站点本身不稳定的时候。加个简单的指数退避重试就行：

```python
async def fetch_with_retry(session, url, max_retries=3):
    for in range(max_retries):
        try:
            result = await fetch(session, url)
            return result
        except Exception:
            await asyncio.sleep(2 ** i)
    return None
```

**坑三：忘了关 session。** aiohttp 的 `ClientSession` 如果不用 `async with` 管理，连接池会泄漏。跑几千个请求之后系统文件描述符耗尽，整个程序卡死。这个跟 ScraperAPI 无关，但新手容易忽略。

## 什么场景值得用、什么场景没必要

值得用的：电商价格监控、SERP 抓取、社交媒体公开数据采集、新闻聚合。这些站点反爬强度高，自己维护代理池的时间成本远超 ScraperAPI 的订阅费。

没必要的：抓自己的站、抓没有任何反爬的政府公开数据、请求量每月不超过几百次的小脚本。这些场景裸 aiohttp 就够了，花钱买 ScraperAPI 纯属浪费。

👉 [用免费额度先跑通你的 aiohttp 爬虫流程](https://www.scraperapi.com/?fp_ref=coupons)

## 和自建代理池的成本对比

我之前用过自建方案：买了 20 个住宅代理 IP，月费大概 $80，加上维护脚本、监控掉线、轮换逻辑的开发时间，折算下来每月隐性成本至少 $200。而且成功率只有 85% 左右，碰到 Cloudflare 保护的站基本废了。

ScraperAPI 的 Startup 套餐 $149/月给 50 万次请求，成功率能稳定在 95% 以上，Cloudflare、Akamai 这些它都能过。算下来单次请求成本不到 $0.003，比自建便宜，还省心。

## 常见问题

### aiohttp 接 ScraperAPI 需要装额外的库吗？

不需要。ScraperAPI 就是个 HTTP API，aiohttp 本身就能调。你只需要 `aiohttp` 和标准库的 `asyncio`，没有额外依赖。

### 免费套餐的 5000 次请求够干什么？

够你验证整个技术方案。跑通代码、测试目标站点的成功率、确认返回数据格式。真要上生产，5000 次肯定不够，但试水绰有余。

### JS 渲染请求为什么消耗额度更多？

因为 ScraperAPI 后端要启动无头浏览器实例来执行 JavaScript，资源消耗是普通请求的好几倍。一个渲染请求大概消耗 10 个普通请求的额度。能不开就别开，先试试不渲染能不能拿到数据。

### 并发超过套餐限制会怎样？

超出的请求会返回 429 状态码。不会额外扣费，但数据拿不到。所以 aiohttp 这边一定要用 Semaphore 控制并发数，别超过你套餐的上限。

### ScraperAPI 支持 POST 请求吗？

支持。你可以在参数里指定 HTTP 方法，也可以通过代理网关模式直接发 POST。抓需要提交表单的页面完全没问题。

### 请求失败会扣额度吗？

ScraperAPI 官方说法是：如果他们这边返回的状态码不是 200，不扣额度。但如果目标站点本身返回 404 之类的，那算成功请求，会扣。所以你的 URL 列表最好提前做一轮清洗。

### 能指定代理 IP 的地理位置吗？

能。在请求参数里加 `country_code=us`（或其他国家代码）就行。抓本地化内容、测试地区定价差异的时候很有用。

## 写在最后

aiohttp 负责并发调度，ScraperAPI 负责突破反爬，这个组合我用了快两年，稳定性和开发效率都让我满意。如果你也在用 aiohttp 写爬虫，反爬问题又让你头疼，建议先用免费额度跑一轮，看成功率和响应速度是不是符合预期。合适了再升级套餐，不合适也没损失。

👉 [现在注册拿 5000 次免费请求，跑通你的 aiohttp 爬虫](https://www.scraperapi.com/?fp_ref=coupons)
