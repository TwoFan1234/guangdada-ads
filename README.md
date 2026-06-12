# guangdada-ads

> 扔一个关键词，给你去重后的竞品广告素材 + 飞书通知——广大大平台双模式（UI自动化 + 直接API）企业级采集方案。

## 什么时候需要它？

- **竞品分析**：想看看对手最近在投什么广告素材，批量下载回来慢慢看
- **素材灵感**：做投放方案前，快速收集热门广告创意作为参考
- **批量采集**：同时搜 10 个游戏的广告素材，全部自动搞定，完事发飞书通知

## 能得到什么？

一句话触发后，自动完成：搜索 → 排序 → 提取 → **去重** → 下载 → 元数据报告 → 飞书通知。最终产物是一组命名规范的素材文件 + 一份 `metadata.md` 摘要。

## 快速开始

### 前置依赖

```bash
# 1. 安装 browser-act-cli（Python 3.12+）
uv tool install browser-act-cli --python 3.12

# 2. pHash 去重依赖（仅视频素材需要）
pip install imagehash pillow
```

### 一次性配置

1. 在本地 Chrome 中登录 [guangdada.net](https://guangdada.net)
2. 导入 Chrome Profile 到 browser-act：
   ```bash
   browser-act browser create --type chrome
   ```

## 触发方式

对 Agent 说：

- 「搜一下 Monster Hunter 的广告素材」
- 「帮我下载 Last War 的前 20 条视频素材」
- 「看看 Honkai Star Rail 最近在投什么广告」
- 「批量下载这几个游戏的图片素材：A, B, C」
- 「帮我扒一下 Whiteout Survival 的竞品广告，发飞书通知」

## 使用示例

### 场景一：单个游戏，少量素材

```
用户：搜 Gossip Harbor®: Merge & Story，按展示估值排序，下载前 10 条视频
```

Agent 自动走 UI 路径：打开页面 → 搜索 → 排序 → 提取 → 下载 → 生成报告 → 飞书通知。

### 场景二：多个游戏，批量采集

```
用户：搜 A、B、C 三个游戏的图片素材，各下第一页
```

Agent 自动走 API 直连路径：批量调用 API → 筛选图片 → 下载 → 报告 → 通知。

## 它和同类有什么不同？

| 特性 | guangdada-ads | guangdada-scraper | 竞品分析类 Skill |
|------|:---:|:---:|:---:|
| 广大大平台 | ✅ | ✅ | ❌ |
| UI 自动化路径 | ✅ | ❌ | ❌ |
| API 直连路径 | ✅ | ✅ | ❌ |
| pHash 感知去重 | ✅ | ❌ | ❌ |
| 飞书通知 | ✅ | ❌ | ❌ |
| 多页翻页抓取 | ✅ | ✅ | ❌ |

核心竞争力：**双模式切换**（单次用 UI、批量用 API）+ **pHash 视觉去重**（不是 MD5，CDN 转码不误杀）。

## 安全边界

| ✅ 会做 | ❌ 不会做 |
|--------|----------|
| 在输出目录创建文件和下载素材 | 删除或修改用户本地文件 |
| 调用浏览器执行搜索和下载 | 泄露 API token / Cookie 到外部 |
| 发送飞书通知总结下载结果 | 发送数据到第三方服务器 |
| 暂停询问高风险操作 | 擅自创建浏览器 Profile |

详见 [SKILL.md](SKILL.md) 中的安全边界章节。

## 文件结构

```
guangdada-ads/
├── SKILL.md          # 完整工作流（13步执行流程 + FAQ）
├── README.md         # 本文件 — 快速了解 + 安装指南
└── assets/           # 示例截图（待补充）
```

## 常见问题

详见 [SKILL.md FAQ](SKILL.md#常见问题)，已覆盖：搜索异常、URL 为空、多页鉴权、0KB 下载、登录过期、素材筛选、速度优化等 7 个场景。
