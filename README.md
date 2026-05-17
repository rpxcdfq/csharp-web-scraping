# C# 网页抓取完整实战指南：从零搭建爬虫到绕过反爬机制，ScraperAPI 各套餐怎么选

## 为什么我最终放弃了自己维护代理池

说实话，我用 C# 写爬虫已经快四年了。最早是用 HttpClient 硬撸，后来换 HtmlAgilityPack 解析 DOM，再后来上了 Selenium 处理动态渲染页面。每一步都踩过坑。

最让我崩溃的不是解析逻辑，而是反爬。IP 被封、验证码弹出、Cloudflare 五秒盾……自己搭代理池维护成本太高，免费代理质量又烂得离谱。后来我开始用 ScraperAPI，把「怎么拿到页面」这件事彻底外包出去，自己只专注写解析逻辑。效率直接翻了一倍不止。

这篇文章我会把 C# 网页抓取的核心流程走一遍，顺便讲我用 ScraperAPI 的真实体验——哪些场景好用，哪些地方有局限。

## C# 网页抓取的基本技术栈

做 C# 爬虫，绑定的几个库基本跑不掉：

- **HttpClient**：.NET 原生 HTTP 请求库，轻量够用
- **HtmlAgilityPack**：解析静态 HTML 的老牌选手
- **AngleSharp**：更现代的 DOM 解析器，API 设计更贴近浏览器
- **Selenium / Playwright**：处理 JavaScript 渲染的页面

静态页面用 HttpClient + HtmlAgilityPack 就够了。但现在越来越多网站是 SPA 架构，数据靠 JS 动态加载，这时候要么上无头浏览器，要么用能渲染 JS 的 API 服务。

我个人的选择是：简单页面自己写，复杂反爬场景交给 ScraperAPI。原因很简单——它帮你处理 IP 轮换、请求头伪装、JS 渲染，你只需要发一个 GET 请求就能拿到完整 HTML。

## 实际代码：用 ScraperAPI + HttpClient 抓取页面

```csharp
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        var client = new HttpClient();
        string apiKey = "你的API密钥";
        string targetUrl = "https://example.com/products";
        
        string requestUrl = $"http://api.scraperapi.com?api_key={apiKey}&url={targetUrl}&render=true";
        
        var response = await client.GetAsync(requestUrl);
        string html = await response.Content.ReadAsStringAsync();
        
        // 拿到完整渲染后的 HTML，接下来用 HtmlAgilityPack 解析
        var doc = new HtmlAgilityPack.HtmlDocument();
        doc.LoadHtml(html);
        
        var nodes = doc.DocumentNode.SelectNodes("//div[@class='product-item']");
        if (nodes != null)
        {
            foreach (var node in nodes)
            {
                Console.WriteLine(node.InnerTextTrim());
            }
        }
    }
}
```

就这么简单。`render=true` 参数让 ScraperAPI 用无头浏览器帮你渲染 JS，返回的就是完整 DOM。不用自己装 Chrome、不用管 ChromeDriver 版本兼容问题。

## 我踩过的坑和 ScraperAPI 的实际表现

先说好的：

- **IP 池够大**。我抓过电商站、招聘网站、新闻聚合站，基本没遇到连续封 IP 的情况
- **地理定位好用**。加个 `country_code=us` 参数就能拿到美国区的内容，做跨境电商选品调研很方便
- **并发稳定**。我同时跑 20 个线程请求，响应时间波动不大

再说不完美的地方：

- **渲染模式下响应偏慢**。开了 `render=true` 之后，单次请求大概 3-8 秒，比纯静态抓取慢不少。批量任务要做好异步调度
- **每次渲染请求消耗的 API 额度是普通请求的 10 倍**。如果你的目标页面其实不需要 JS 渲染，别随手开这个参数，费额度
- **偶尔会返回验证页面**。概率很低，但不是零。我的做法是加个重试逻辑，一般第二次就正常了

## 什么场景适合用 ScraperAPI，什么场景不需要

**适合的场景：**
- 目标网站有 Cloudflare、Akamai 等反爬保护
- 需要大规模并发抓取（几千到几十万页面）
- 需要多地区 IP 轮换
- 目标页面依赖 JS 渲染

**不需要的场景：**
- 抓自己公司内部系统
- 目标网站没有任何反爬措施，HttpClient 直接就能拿到数据
- 每天只抓几十个页面，手动换 IP 也能搞定

说白了，ScraperAPI 解决的是「规模化抓取时的基础设施问题」。如果你的需求很轻量，原生 HttpClient 就够了。

## ScraperAPI 全套餐对比

| 套餐名称 | API 请求额度 | 并发数 | 地理定位 | 价格 | 适合谁 | 链接 |
| ------ | ---------- | ---- | ------- | --- | --- | --- |
| Hobby | 100,000 次/月 | 20 | 支持 | $49/月 | 个人开发者、小项目验证 | [ 开通 Hobby 套餐查看完整配置](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 次/月 | 50 | 支持 | $149/月 | 中小团队、日常数据采集 | [ 开通 Startup 套餐锁定官方价格](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 次/月 | 100 | 支持 | $299/月 | 数据密集型业务、电商监控 | [ 开通 Business 套餐享完整并发能力](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | 支持 + 高级定制 | 联系销售 | 大型企业、定制化需求 | [ 联系销售获取 Enterprise 定制方案](https://www.scraperapi.com/?fp_ref=coupons) |
| Free | 5,000 次 | 1 | 有限 | $0 | 测试体验、技术验证 | [ 免费注册体验 ScraperAPI 基础功能](https://www.scraperapi.com/signup?fp_ref=coupons) |

说一下我的选择逻辑：我目前用的是 Startup 套餐。100 万次请求对我来说刚好够，50 并发跑批量任务也不卡。如果你刚开始接触，建议先用免费的 5000 次额度把流程跑通，确认适合再升级。

[👉 直接去官网比较全部套餐配置和最新价格](https://www.scraperapi.com/?fp_ref=coupons)

## 进阶技巧：让 C# 爬虫更稳定

### 异步批量请求的正确姿势

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

class BatchScraper
{
    private static readonly HttpClient _client = new HttpClient();
    private static readonly SemaphoreSlim _semaphore = new SemaphoreSlim(20); // 控制并发

    public static async Task<List<string>> ScrapeUrls(List<string> urls, string apiKey)
    {
        var tasks = urls.Select(url => ScrapeWithThrottle(url, apiKey));
        var results = await Task.WhenAll(tasks);
        return results.ToList();
    }

    private static async Task<string> ScrapeWithThrottle(string url, string apiKey)
    {
        await _semaphore.WaitAsync();
        try
        {
            string requestUrl = $"http://api.scraperapi.com?api_key={apiKey}&url={url}";
            var response = await _client.GetAsync(requestUrl);
            return await response.Content.ReadAsStringAsync();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

关键点：用 `SemaphoreSlim` 控制并发数，别超过你套餐的并发上限。超了会返回 429 状态码。

### 重试机制

网络请求不可能百分百成功。加个 Poly 重试策略：

```csharp
// 安装 NuGet 包：Microsoft.Extensions.Http.Polly
// 配置 3 次指数退避重试
builder.Services.AddHttpClient("ScraperAPI")
    .AddTransientHttpErrorPolicy(p => 
        p.WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
```

我实际跑下来，加了重试之后成功率从 96% 提到了 99.5% 以上。

## 常见问题

#### C# 网页抓取违法吗？

抓取公开可访问的数据本身不违法，但要注意目标网站的 robots.txt 和服务条款。我的原则是：不抓需要登录的私人数据，不对目标服务器造成过大压力，不抓取个人隐私信息。

#### ScraperAPI 支持 .NET Framework 还是只支持 .NET Core？

都支持。它本质上就是个 HTTP API，你用什么版本的 .NET 发请求都行。HttpClient 在 .NET Framework 4.5+ 和 .NET Core/5/6/7/8 里都能用。

#### 免费额度用完了会怎样？

API 会返回 403 状态码，不会自动扣费。你需要手动升级套餐才能继续使用。这点我觉得挺厚道的，不会偷偷扣钱。

[👉 免费注册先拿 5000 次请求额度测试](https://www.scraperapi.com/signup?fp_ref=coupons)

#### HtmlAgilityPack 和 AngleSharp 选哪个？

如果你习惯 XPath，用 HtmlAgilityPack。如果你更喜欢 CSS 选择器和类似 jQuery 的 API，用 AngleSharp。性能上差别不大，AngleSharp 对标准的支持更完整一些。我两个都用，看心情。

#### ScraperAPI 的响应速度怎么样？

普通请求（不开渲染）大概 1-3 秒返回。开了 JS 渲染的请求 3-8 秒。如果你对延迟敏感，建议批量任务用异步并发跑，单次查询场景下这个延迟基本可以接受。

#### 被目标网站封了怎么办？

这正是用 ScraperAPI 的核心价值。它自动轮换 IP、处理验证码、模拟真实浏览器指纹。我用了大半年，被封的情况屈指可数，偶尔遇到重试一次就好了。如果你自己维护代理池，光是处理封禁逻辑就够喝一壶的。

## 如果让我重新选一次

我还是会选 ScraperAPI 配合 C# 原生 HttpClient 这套组合。原因很实际：我的时间值钱，花在维护代理池和对抗反爬上的精力，不如花在业务逻辑和数据分析上。Startup 套餐每月 149 美元，算下来比我自己租服务器搭代理池还便宜，稳定性还更好。

唯一的建议是：先用免费额度把你的目标网站跑一遍，确认渲染模式和响应速度符合预期，再决定买哪个档位。别一上来就冲最贵的。

[👉 立即注册 ScraperAPI 免费体验完整抓取能力](https://www.scraperapi.com/signup?fp_ref=coupons)
