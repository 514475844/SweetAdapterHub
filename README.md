# Sweet 采集 AI 版 · 适配器市场

这是 [Sweet 采集 AI 版](https://github.com/514475844/SweetCollectorAI) 的**适配器规则市场**。

一条适配器规则就是一个 YAML（或 JSON）文本文件，描述"某个网站的图片放在页面的哪里、用什么属性取地址"。软件按域名自动匹配规则，从此那个站点的采集任务不需要任何配置。

**下载器本体是中性的**：它只按规则取公开网页上的公开图片，不绕过任何登录、付费或访问控制。规则由用户自行编写与提交，是否对目标内容拥有采集权限，由使用者自己负责。

---

## 目录结构

```
SweetAdapterHub/
├── index.json          # 市场清单（软件读这个文件展示可订阅的规则）
├── rules/              # 全部规则文件
│   └── pictoa.yaml
└── README.md           # 本文档：规则怎么写
```

## 在软件里订阅本市场

软件 → 适配器 → 适配器市场 → 添加订阅源，填：

```
https://514475844.github.io/SweetAdapterHub/index.json
```

即可浏览、搜索、一键安装本仓库收录的规则。

---

## 规则怎么写（完整字段说明）

### 最小可用规则

一个规则**必须有**下面三项，缺一软件会提示"有问题"且不会被使用：

```yaml
name: pictoa                        # 规则名：字母/数字/下划线/短横线
base_url: https://www.pictoa.com    # 或 entry_urls，二选一
allowed_domains: [pictoa.com, t1.pictoa.com, s2.pictoa.com]
image_link_selectors:               # 图片选择器：没有它一张图都采不到
  - kind: css
    expr: "img[data-lazy-src]"
    attr: data-lazy-src
```

### 全部字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `name` | ✅ | 规则唯一名。只允许字母、数字、下划线、短横线；会作为文件名 |
| `base_url` | 二选一 | 站点根地址。任务里粘这个域名下的任意网址都能起步 |
| `entry_urls` | 二选一 | 入口页列表（没有固定根路径时用它） |
| `allowed_domains` | ✅ | 允许抓取的域名白名单。**只写你自己有采集权限的站** |
| `list_page_selectors` | – | 列表页里"相册/图集链接"的选择器，见下 |
| `album_link_selectors` | – | 从入口页找相册链接 |
| `image_page_selectors` | – | 从相册页找"下一页/图片页"链接 |
| `image_link_selectors` | ✅ | **核心字段**：图片元素选择器，可写多条按优先级排列 |
| `original_image_attrs` | – | 从 img 元素取地址的属性名列表，默认 `[src, data-src, data-original]` |
| `image_url_replace` | – | URL 替换表：很多站缩略图和原图只差域名或路径 |
| `next_page_selectors` | – | 翻页链接选择器 |
| `paging` | – | JSON 接口分页（页面里没有翻页链接、但接口支持 offset 时） |
| `cookie` | – | 站点级登录态。只填公开可访问内容所需的，不绕过付费墙 |
| `user_agent` | – | 固定 UA（有些站只认浏览器 UA） |
| `max_depth` | – | BFS 深度上限，默认 3 |
| `max_pages` | – | 翻页上限，0 为不限；建议收紧到 50–500 |
| `delay` | – | 每页间隔秒数，默认 1.0。**不要调小，礼貌抓取** |
| `concurrency` | – | 下载并发，默认 4 |
| `note` | – | 备注：规则依据、维护说明 |

### 选择器的三种写法

```yaml
# 1) CSS 选择器（最常用）：attr 是从元素哪个属性取 URL，省略则取 href
- kind: css
  expr: "div.gallery img"
  attr: data-src

# 2) XPath
- kind: xpath
  expr: "//div[@class='gallery']//img/@data-original"

# 3) 正则（直接从 HTML 文本里抠链接）
- kind: regex
  expr: '"(https://img\\.example\\.com/[^"]+\\.jpg)"'
```

### 懒加载站点怎么对付

图片地址不在 `src`（占位图）而在 `data-src` / `data-lazy-src` 时，
`image_link_selectors` 按"真地址优先、占位兜底"写两条：

```yaml
image_link_selectors:
  - kind: css
    expr: "img[data-lazy-src]"
    attr: data-lazy-src
  - kind: css
    expr: "img[src]"
    attr: src
```

### 缩略图换原图

很多站缩略图域名和原图域名只差一个子域：

```yaml
image_url_replace:
  - from: "t1.example.com"
    to: "s2.example.com"
```

### JSON 接口分页

页面里没有翻页链接，但接口支持 `?offset=` 时：

```yaml
paging:
  mode: offset_json
  url: "https://api.example.com/list?offset={offset}&limit={limit}"
  offset_start: 20
  limit: 20
  link_key: g_url      # 从返回 JSON 的每个对象里取这个键作为图片地址
  max_pages: 500
```

---

## 怎么写一条规则（实操流程）

1. 浏览器打开目标站点的一个图集页，按 F12
2. 找到任意一张图片元素，右键 → 复制 → 复制选择器，先拿到 CSS 路径
3. 看图片真实地址在哪个属性（`src`? `data-src`?），点开确认是不是原图
4. 若是缩略图，对比原图 URL 的差异（域名?路径?尺寸参数?），写进 `image_url_replace`
5. 按上面的模板写 YAML，在软件「适配器 → 新建规则」里粘贴保存
6. 新建一个该站任务试跑一次，确认图片数量与原图质量

也可以让 AI 帮你起草：把页面结构和上面的字段表一起丢给它，生成的规则**务必**按第 5–6 步验证。

## 提交规则

两种方式，任选其一：

1. **提 Issue**：用 [提交规则](https://github.com/514475844/SweetAdapterHub/issues/new?template=submit_rule.yml) 模板，把 YAML 贴进来；
2. **发 PR**：把 `<name>.yaml` 放进 `rules/`，同时更新 `index.json` 追加一条，PR 描述里说明目标站点与采集依据（内容公开可访问、无登录墙绕过）。

维护者会实际跑一遍规则（图片数量、原图质量、限速参数）再合并。**不合格或涉嫌绕过访问控制的规则会被拒**。

## 收录边界（重要）

- 站点内容必须**公开可访问**（游客可见即算公开）；不收录需破解登录、付费墙、防盗链的规则
- 不收录以"批量搬运他人作品再分发"为目的的站点规则
- 规则作者的署名与来源会保留在 `index.json` 中
- 权利人要求下架某条规则：提 Issue 或邮件联系，24 小时内移除

## 许可

规则文件 MIT。
