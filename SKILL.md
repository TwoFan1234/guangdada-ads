---
name: guangdada-ads
description: "Search and download ad creatives (images & videos) from 广大大 (guangdada.net) display ads. Use when user wants to search for ads on guangdada, download ad images/videos by keyword, get top N ad creatives sorted by impression/duration/heat, bulk download ad creatives, or scrape 广大大 ad data."
metadata:
  author: yanqu1124
  version: "1.5.0"
  agent_created: true
  requires:
    tools: "browser-act"
    runtime: "Python 3.12+, uv, browser-act-cli"
  permissions:
    - "Network access — browser-act connects to guangdada.net"
    - "Filesystem write — saves downloaded files to output directory"
    - "Chrome browser — imports login state for authentication"
---

# 广大大广告素材搜索下载

从[广大大展示广告](https://guangdada.net/modules/creative/display-ads)搜索广告创意素材，按指定排序取 Top N，批量下载**图片**或**视频**文件。

## 触发方式

用户说出以下任意一句话即可触发本 Skill：

- 「搜一下 XX 游戏的广告素材」
- 「帮我下载 XX 的前 20 条视频素材」
- 「看看 XX 最近在投什么广告」
- 「批量下载这几个游戏的素材：A, B, C」
- 「帮我抓一下广大大上 XX 的图片广告，按热度排序」
- 「扒一下 XX 的竞品广告素材，发飞书通知」

## 前置条件

1. 用户需在本地 Chrome 中已登录 guangdada.net
2. `browser-act-cli` 和 pHash 依赖 — **Skill 会自动安装，无需手动操作**
3. 已有可用的 Chrome 浏览器（Skill 会自动创建，首次需用户确认）

## 参数

解析用户输入，确认以下参数（未指定的使用默认值）：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **关键词** | 搜索关键词 | 必填，向用户询问 |
| **素材类型** | `image`(图片) / `video`(视频) / `all`(全部) | 向用户询问 |
| **排序字段** | `impression`(展示估值) / `first_seen`(最新创意) / `heat`(热度) / `days`(投放天数) | 展示估值 |
| **下载数量** | Top N 条 | 10 |
| **输出目录** | 文件保存位置 | `桌面/guangdada-ads-{关键词}/` |

### 素材类型映射

| 用户说 | API ads_type | 下载字段 |
|--------|-------------|----------|
| 图片/图片素材 | `["1"]` | `resource_urls[].image_url` |
| 视频/视频素材 | `["2"]` | `resource_urls[].video_url` |
| 全部/都要 | 不传 | 两者都检查 |

### 排序字段映射

| 用户说 | 页面按钮 | API sort_field |
|--------|---------|---------------|
| 展示估值 | 展示估值 | `-impression` |
| 最新创意 | 最新创意 | `-first_seen` |
| 最后看见 | 最后看见 | `-last_seen` |
| 热度 | 热度 | `-heat` |
| 投放天数 | 投放天数 | `-days_count` |

---

## 执行流程

### Step 1: 环境检查（自动安装缺失依赖）

**本 Skill 会自动安装所有缺失的依赖，无需用户手动操作。**

```bash
# 1. 检查 uv（python 包管理器）
uv --version 2>/dev/null || pip install uv

# 2. 安装 browser-act-cli（如未安装）
browser-act --version 2>/dev/null || uv tool install browser-act-cli --python 3.12

# 3. 安装 pHash 去重依赖（仅视频需去重，已安装则跳过）
pip show imagehash 2>/dev/null || pip install imagehash pillow

# 4. 检查 ffmpeg（视频去重必需，未安装则提示用户）
ffmpeg -version 2>/dev/null || echo "⚠️ 请安装 ffmpeg: https://ffmpeg.org/download.html"

# 5. 检查可用浏览器
browser-act browser list
# 如无 Chrome 浏览器，需先创建：
# 1. 加载高级指南：browser-act get-skills advanced
# 2. 按 Confirmation Gate 协议创建 chrome 浏览器
```

如无可用浏览器，**必须先让用户确认后创建**（遵循 browser-act Confirmation Gate 协议）。

已有浏览器可直接复用。

### Step 2: 打开页面

```bash
browser-act --session guangdada browser open <browser_id> https://guangdada.net/modules/creative/display-ads
browser-act --session guangdada wait stable
browser-act --session guangdada state   # 获取搜索框和按钮索引
```

### Step 3: 设置 XHR 拦截器

**必须在搜索之前执行！**（页面导航会重置拦截器）

```bash
browser-act --session guangdada eval "
(() => {
  const XHR = XMLHttpRequest.prototype;
  const origOpen = XHR.open, origSend = XHR.send;
  XHR.open = function(m,u) { this._u = u; return origOpen.apply(this, arguments); };
  XHR.send = function(b) {
    const x = this;
    if (x._u && x._u.includes('creative/list')) {
      x.addEventListener('load', () => { try { window.__gdData = JSON.parse(x.responseText); } catch(e) {} });
    }
    return origSend.apply(this, arguments);
  };
  return 'ok';
})()
```

此拦截器将每次 `creative/list` API 的响应存入 `window.__gdData`。

### Step 4: 执行搜索

**⚠️ 关键：必须点击搜索按钮，绝对不能点击自动补全下拉中的广告主选项！**

点击广告主选项 = 广告主筛选（advertiser_key 过滤），结果只含该广告主。
点击搜索按钮 = 关键词搜索，结果包含所有匹配广告。

```bash
# 输入关键词到搜索框（state 中找到搜索 input 的索引，通常为 78）
browser-act --session guangdada input <search_input_idx> "关键词"

# 点击搜索按钮（搜索框右侧的按钮，不是"排除"按钮）
# 从 state 中找到搜索按钮的索引（通常紧挨搜索 input 之后）
browser-act --session guangdada click <search_button_idx>

# 等待结果加载
browser-act --session guangdada wait stable
```

**如何区分搜索按钮和自动补全：**
- 搜索按钮：位于搜索输入框右侧，是页面上的固定按钮
- 自动补全：是输入后弹出的下拉框，包含广告主名称和图标

**验证搜索正确性：** 页面应显示 "共找到 XX 万 个相关广告"，且搜索结果中包含来自不同广告主的相关广告（如果关键词关联多个广告主的话）。

✅ **检查点 1 — 搜索验证：** 确认搜索结果数 ≠ 5.4亿，确认页面显示的是关键词过滤结果而非全站数据。失败则回到 Step 4 重新搜索。

### Step 5: 切换排序

从 `state` 中找到目标排序按钮的索引，点击切换：

```bash
browser-act --session guangdada click <sort_button_idx>
browser-act --session guangdada wait stable
```

排序完成后自动触发 API 请求，拦截器会更新 `window.__gdData`。

### Step 6: 提取数据

```bash
browser-act --session guangdada eval "
(() => {
  if (!window.__gdData) return JSON.stringify({error: 'no data'});
  const list = window.__gdData.data?.creative_list || [];
  const topN = list.slice(0, <N>).map((item, i) => {
    const videos = [];
    (item.resource_urls || []).forEach(r => {
      if (r.video_url) videos.push(r.video_url.trim());
    });
    return {
      rank: i + 1,
      title: item.title || item.advertiser_name,
      advertiser: item.advertiser_name,
      platform: item.platform,
      impression: item.impression,
      heat: item.heat,
      days_count: item.days_count,
      video_duration: item.video_duration,
      videos: videos
    };
  });
  window.__gdTopN = topN;
  return JSON.stringify({ total_extracted: topN.length, sample: topN[0] });
})()
"
```

验证提取的数据中包含 `videos` URL（`video_url` 字段，非 `image_url`）。

✅ **检查点 2 — 数据提取：** 确认 `total_extracted` ≥ 1，确认提取的 URL 列表非空且格式正确（以 `http` 开头）。失败则检查拦截器是否仍在工作，必要时回到 Step 3 重新设置。

### 路径选择

根据任务规模选择后续路径：

| 场景 | 路径 | 下一步 |
|------|------|--------|
| N ≤ 60（单页） | **UI 路径** | 跳到 Step 9 下载 |
| N > 60（多页） | **UI 多页路径** | Step 7 翻页抓取 |
| 批量 / Token 有效 | **API 直连路径** | Step 8 直接 API |

### Step 8: 直接 API 调用（推荐用于批量任务）

**当用户指定多个游戏或明确需要批量抓取时，优先使用直接 API 调用。**

原理：从浏览器 Network 面板 Copy as cURL 获取鉴权头后，直接用 PowerShell `Invoke-RestMethod` 调用 API。

```powershell
# 从 curl 命令中提取这些鉴权头
$headers = @{
    "Authorization" = "Bearer <jwt_token>"
    "x-nbs-user-token" = "<user_token>"
    "x-device-id" = "<device_id>"
    "x-product-id" = "2"
    "Content-Type" = "application/json"
    "Origin" = "https://guangdada.net"
}

# 请求体 — 关键参数
$body = @{
    keyword = @("游戏名")
    page = 1
    page_size = 60
    sort_field = "-impression"        # 展示估值降序
    ads_type = @("1")                 # ⚠️ "1"=图片 "2"=视频 不传=全部
    search_type = 1
    app_type = 1
    duplicate_removal = 0
    position = "0"
    complete_country_match = $false
    new_ads_flag = 0
    new_advertiser_flag = $false
    has_custom_store_page = $false
    seen_begin = <近90天开始时间戳>
    seen_end = <当前时间戳>
    fb_merge = $false
    is_web = 0
    original_flag = 0
    is_dynamic = 0
    landing_page = 0
} | ConvertTo-Json -Depth 5

$resp = Invoke-RestMethod -Uri "https://guangdada.net/napi/v1/creative/list" -Method Post -Headers $headers -Body $body
```

**图片/视频筛选关键：**
- 图片素材：`"ads_type": ["1"]` → 从 `resource_urls[].image_url` 下载
- 视频素材：`"ads_type": ["2"]` → 从 `resource_urls[].video_url` 下载
- 全部素材：不传 `ads_type` → 两者都检查
- ⚠️ 字段名是 `ads_type`，不是 `material_type` 或 `creative_type`

**优势：**
- 不需要 browser-act 交互，避免登录态过期问题
- 10 个游戏批量抓取只需几分钟
- Token 过期后从浏览器重新 Copy as cURL 即可

### Step 7: 多页抓取（当 N > 60 且使用 UI 方式时）

**⚠️ 不可直接通过 eval 调用 `/napi/v1/creative/list` API** — 页面 API 需要特定的鉴权头（`x-nbs-user-token`、`Authorization` 等），从 eval 发起的 XHR/fetch 不会自动携带这些头，会返回空结果。

**正确做法：通过分页组件翻页 + XHR 拦截器逐页捕获。**

```bash
# 在排序完成后，用 eval 同时设置拦截器并逐页点击
browser-act --session guangdada eval "
(async () => {
  // 存储所有捕获的响应
  const captured = [];
  
  // 重新 Patch XHR（页面跳转可能重置了拦截器）
  const XHR = XMLHttpRequest.prototype;
  const origOpen = XHR.open, origSend = XHR.send;
  XHR.open = function(m, u) { this._u = u; return origOpen.apply(this, arguments); };
  XHR.send = function(b) {
    const xhr = this;
    if (xhr._u && xhr._u.includes('creative/list')) {
      xhr.addEventListener('load', function() {
        try {
          const d = JSON.parse(xhr.responseText);
          captured.push({ page: JSON.parse(b).page, items: d.data?.creative_list || [] });
          window.__capturedAll = captured;
        } catch(e) {}
      });
    }
    return origSend.apply(this, arguments);
  };
  
  // 点击分页按钮的辅助函数
  const clickPage = (pageNum) => new Promise((resolve) => {
    const items = document.querySelectorAll('.ant-pagination-item');
    for (const item of items) {
      if (item.textContent.trim() === String(pageNum)) {
        item.click();
        break;
      }
    }
    // 轮询等待该页数据到达
    const check = setInterval(() => {
      const last = captured[captured.length - 1];
      if (last && last.page === pageNum) { clearInterval(check); resolve(true); }
    }, 500);
    setTimeout(() => { clearInterval(check); resolve(false); }, 30000);
  });
  
  // 逐页抓取（page 1 已由排序触发，从 page 2 开始）
  const totalPages = Math.ceil(<N> / 60);
  for (let p = 2; p <= totalPages; p++) {
    await clickPage(p);
  }
  
  return JSON.stringify({ pages_captured: captured.length });
})()
"
```

**原理：** 点击分页按钮会触发页面发起带完整鉴权头的 API 请求，XHR 拦截器自动捕获响应。这样绕过了从 eval 直接调 API 缺鉴权头的问题。

**页面合并提取：** 所有页面数据存入 `window.__capturedAll`，后续提取时遍历所有 page 的 items 即可。

### Step 9: 下载素材

**⚠️ 必须先创建子文件夹，再下载素材到里面！**

子文件夹命名规则：`{关键词}_前{N}页去重素材`

- 关键词中的特殊字符替换为下划线（保留字母、数字、中文）
- 示例：`Last_Asylum_Plague_前三页去重素材`、`Gossip_Harbor_Merge_Story_前五页去重素材`

完整目录结构：`{用户指定目录}/{关键词}_前{N}页去重素材/`

```powershell
# 第一步：在用户指定的输出目录下创建子文件夹
$baseDir = "{用户指定目录}"
$subDirName = "{清洗后关键词}_前{N}页去重素材"
$dir = Join-Path $baseDir $subDirName
New-Item -ItemType Directory -Force -Path $dir | Out-Null

# 第二步：从浏览器获取 URL 列表并下载到子文件夹，文件命名规则见下方说明
```

**下载去重策略：** 同一 `video_url` 只下载一次（同一素材在多个渠道投放时只存一份）。

**文件命名规则：** `{排名}_{展示量}_{游戏名}.{ext}`

命名示例：`01_10.9M_Whiteout_Survival.mp4`

命名字段说明：

| 字段 | 来源 | 格式 |
|------|------|------|
| 排名 | `item.rank` | 两位数字补零（01, 02, ...） |
| 展示量 | `item.impression` | **缩写格式**（见下方规则） |
| 游戏名 | 搜索关键词 | 去特殊字符，空格替换为下划线，截断 30 字符 |

展示量缩写规则（英文单位）：
- ≥ 10 亿：`{value}B`，如 `1.5B`
- ≥ 100 万：`{value}M`，如 `10.9M`、`795K`（10.9M = 1090万）
- ≥ 1000：`{value}K`，如 `200K`
- < 1000：原始数字

注：统一使用英文计数习惯，1M = 1,000,000，1B = 1,000,000,000。

游戏名清洗规则（基于搜索关键词）：
- 移除 `®`, `™`, `:`, `'`, `"`, 括号等特殊字符
- 空格替换为下划线 `_`
- 截断到 30 字符

**下载方式：** 使用 PowerShell `Invoke-WebRequest`，视频设置 120 秒超时。

✅ **检查点 3 — 下载完整性：** 确认下载文件数 ≥ 预期数 × 80%，确认文件大小 > 0KB。如有 0KB 文件，在 metadata 中标注跳过原因。

### Step 10: 下载后图像感知去重

同一广告创意在不同渠道投放时，CDN 会重新编码视频，导致 MD5、时长、文件大小都不同，但**视觉内容完全相同**。必须通过**感知哈希（pHash）** 逐帧比对来去重。

**前置依赖：** `pip install imagehash pillow`（ffmpeg 需已在 PATH 中）

```python
# 在下载目录运行：python img_dedup.py
import subprocess, os
from PIL import Image
import imagehash

files = [f for f in os.listdir('.') if f.endswith('.mp4')]

# 1. 每个视频提取第 2 秒帧，计算 pHash
hashes = {}
for i, f in enumerate(files):
    frame = f'temp_{i}.jpg'
    subprocess.run(['ffmpeg', '-y', '-ss', '2', '-i', f, '-vframes', '1',
                    '-q:v', '2', frame], capture_output=True, timeout=20)
    if os.path.exists(frame):
        hashes[f] = imagehash.phash(Image.open(frame))
        os.remove(frame)

# 2. 按 Hamming 距离 ≤ 5 分组，保留排名最高的
files_with_hash = list(hashes.keys())
processed = set()
os.makedirs('重复素材', exist_ok=True)

for i, f1 in enumerate(files_with_hash):
    if f1 in processed: continue
    group = [f1]
    for j, f2 in enumerate(files_with_hash):
        if j <= i or f2 in processed: continue
        if (hashes[f1] - hashes[f2]) <= 5:
            group.append(f2); processed.add(f2)
    if len(group) > 1:
        group.sort(key=lambda x: int(x.split('_')[0]))  # 按排名排序
        for dup in group[1:]:
            os.rename(dup, f'重复素材/{dup}')
            print(f'Dup: {dup} (same as {group[0]})')
```

### Step 11: 生成元数据

在输出目录生成 `metadata.md`：

```markdown
# {关键词} — 广大大广告素材 Top {N}

**排序**: {排序字段} | **时间**: {当前日期}

| 排名 | 标题 | 广告主 | 展示估值 | 热度 | 渠道 | 投放天数 |
|------|------|--------|----------|------|------|----------|
| #1 | ... | ... | ... | ... | ... | ... |
```

### Step 12: 清理

```bash
browser-act session close guangdada
```

### Step 13: 发送飞书通知（必须执行！）

**任何耗时较长的批量任务完成后，必须发送飞书通知。** 用户无法一直盯着终端。

通过 lark-cli 以 bot 身份发送 Markdown 摘要到用户飞书私聊。

**前置：** 需已配置 lark-cli（`lark-cli config init`），默认用户 open_id 为 `ou_73a2ae110cc751089eca3f0e1a62e5f8`。

```bash
# 1. 获取用户 open_id
lark-cli config show   # 从输出中取 users 字段的 open_id

# 2. 发送通知
lark-cli im +messages-send \
  --user-id <user_open_id> \
  --markdown "**🎬 {关键词} 广告素材下载完成**

📊 搜索关键词：{关键词}
📈 排序：{排序字段}
📄 抓取：{页数} 页 = {总条数} 条
🎬 独立视频：{去重后数量} 个
🗑️ 去重移除：{重复数量} 个

**🏆 Top 5：**
1. **{展示量}** — {渠道}
2. ...

📁 保存位置：桌面 \`{输出目录}\`" \
  --as bot
```

**消息模板要点：**
- 用 `--as bot` 以应用身份发送
- 用 `--markdown` 支持富文本格式
- 列出 Top 5 展示估值及对应渠道
- 标注保存路径方便用户找到文件

用户说：「搜索 Gossip Harbor®: Merge & Story，按展示估值排序，下载前 10 条」

**单游戏/少量：走 UI 流程**
1. 确认参数：关键词=`Gossip Harbor®: Merge & Story`，排序=展示估值，数量=10
2. 检查环境，打开页面
3. 设 fetch 拦截器
4. 输入关键词，**点搜索按钮**（非自动补全）
5. 验证结果数为「18万」而非「5.4亿」（说明搜索生效）
6. 点击「展示估值」排序
7. 提取 Top 10 数据
8. **先建子文件夹**，再下载视频到里面
9. 保存 metadata.md
10. **发送飞书通知**
11. 关闭会话

用户说：「搜 A、B、C 三个游戏的图片素材，各下第一页」

**多游戏/批量：优先用直接 API**
1. 向用户要 curl（或从上次保存的 headers 复用）
2. 用 PowerShell `Invoke-RestMethod` 批量调 API（`ads_type:["1"]` 筛选图片）
3. 提取 `creative_list` 中的 `image_url`
4. **先建子文件夹**，下载图片（命名如 `01_11.7M_Kingshot.jpg`）
5. 保存 metadata.md
6. **发送飞书通知**

---

## 安全边界

**本 Skill 绝对不会做以下事情：**

- ❌ 删除或修改用户本地文件（只创建输出目录和下载文件）
- ❌ 泄露 API token、Cookie 或登录凭据到外部
- ❌ 发送数据到第三方服务器（所有操作在本地浏览器和用户指定的输出目录完成）
- ❌ 修改浏览器配置或安装扩展
- ❌ 自动执行登录操作（需要用户预先在 Chrome 中登录）
- ❌ 覆盖已存在的同名文件（PowerShell 下载不设 `-Force`）

**本 Skill 会在以下情况暂停并询问用户：**

- 创建新的 Chrome 浏览器 Profile（遵循 browser-act Confirmation Gate）
- 输出目录已存在时是否覆盖
- 检测到搜索结果异常（5.4 亿全站数据而非关键词过滤结果）

## 不适用场景

以下情况**不应该**使用本 Skill，直接告知用户：

- 用户未在 Chrome 中登录 guangdada.net 且无法登录 → 建议先登录
- browser-act-cli 未安装且用户拒绝安装 → 无法继续
- 搜索返回 0 条结果 → 建议更换关键词
- Token 过期且用户无法重新获取（API 路径） → 建议走 UI 路径

---

## 常见问题

### Q: 搜索结果数是 5.4 亿（全部广告），而不是 XX 万？
**A:** 搜索未正确提交。可能是按了 Enter 键选中了自动补全项，或者点击了清除按钮。**解决：** 重新输入关键词，点击搜索按钮（不是旁边的清除/排除按钮）。

### Q: 提取的 video_url 为空？
**A:** 视频 URL 在 API 响应的 `resource_urls[].video_url` 字段中，不在 `image_url` 中。检查提取代码是否正确读取了 `r.video_url`。

### Q: 需要多页数据（> 60 条），但 eval 直接调 API 返回空？
**A:** 页面 API 需要 x-nbs-user-token / Authorization 等鉴权头，eval 中的 fetch/XHR 不会自动携带。**解决：** 使用 Step 7 的翻页方案 — 通过点击 `.ant-pagination-item` 分页按钮触发请求，用 XHR 拦截器捕获。详见 Step 7。

### Q: 下载的文件是 0KB 或失败？
**A:** 某些旧素材的视频文件可能已被 CDN 清理。跳过失败的 URL，在 metadata 中标注即可。

### Q: 需要重新登录？
**A:** 如页面重定向到登录页，说明 Chrome 登录态已过期。需用户在本地 Chrome 中重新登录 guangdada.net，然后重新导入 Profile。或直接走 API 直连方式（Step 8），token 未过期时无需登录。

### Q: 如何筛选纯图片素材（非视频缩略图）？
**A:** API 参数用 `"ads_type": ["1"]`，不是 `material_type`。UI 上点击「图片&视频」下拉 → 选「图片」，已选标签会显示「广告类型: 图片」。

### Q: 批量下载多个游戏时速度太慢？
**A:** 走直接 API 方式（Step 8）。单个游戏 2 秒出结果，10 个游戏 + 下载总共几分钟。
