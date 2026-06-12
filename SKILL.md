---
name: guangdada-ads
description: Search and download Guangdada / SocialPeta display-ad creatives from `guangdada.net/modules/creative/display-ads` using the user's logged-in Chrome state through `browser-act`. Use when the user wants to search a keyword or game, sort by metrics such as 展示估值 / 热度 / 投放天数, process 1 to 5 pages of results, deduplicate repeated creatives, download images or videos, or rebuild manifest/summary artifacts from prior Guangdada checkpoints.
---

# Guangdada Ads

用 `browser-act` 驱动用户自己的 Chrome 登录态，在广大大展示广告页里搜索、排序、翻页、去重并批量下载素材。

## 默认策略

- 优先走 `browser-act` + 页面内真实登录态，不要假设有单独可用的公开 API token。
- 优先切到列表视图，不要在网格视图里批量处理。
- 优先走“两阶段流水线”：
  1. 先扫描列表页
  2. 再只对疑似唯一素材进详情
- 优先处理 1 到 5 页；页数更多时也沿用同一套 checkpoint 流程。

## 执行流程

1. 确认用户本地 Chrome 已登录 `guangdada.net`，并且本机可用 `browser-act-cli`。
2. 打开 `https://guangdada.net/modules/creative/display-ads`。
3. 在创意结果面板里输入关键词，不要误点自动补全里的广告主选项。
4. 按用户要求切换排序；未指定时默认按 `展示估值`。
5. 切到列表视图。
6. 先扫描请求页数内的列表行，记录：
   - `pageNo`
   - `rowIndex`
   - `rank`
   - `estimate`
   - `advertiser`
   - `thumbUrl`
7. 处理懒加载缩略图：
   - 先分块滚动
   - 再只回补缺图行
   - 不要把第一次没读到的缩略图直接当成“没有图”
8. 先按缩略图 URL 去重：
   - 去掉 query string 后再比较
   - 相同缩略图只保留最早排名
9. 把扫描结果按页或批次保存成 checkpoint。
10. 只对预去重后保留下来的行进入详情页，提取真实素材字段。
11. 如果详情明显串到了上一条素材，先关闭再重开同一条一次。
12. 再按真实素材地址做最终去重。
13. 用 `scripts/build_run_artifacts.py` 生成：
   - `candidates.json`
   - `manifest.json`
   - `summary.json`
14. 用 `scripts/download_manifest.py` 下载最终素材。
15. 如果用户要求单独存放，就用单独子目录。

## 详情提取优先级

进入详情后，按这个顺序拿素材字段：

1. `video.currentSrc`
2. `video.poster`
3. `单页打开` detail URL
4. 第一个外链官方地址
5. 第一个 `sp_opera` 预览图

## 去重规则

### 第一阶段：列表页预去重

- 键：标准化后的 `thumbUrl`
- 目标：减少重复点开详情的次数
- 注意：这只是提速，不是真正最终去重

### 第二阶段：详情页最终去重

按这个优先级：

1. `videoSrc`
2. `poster`
3. `previewImage`
4. `detailLink`
5. synthetic fallback key

始终保留最早排名的那条。

## 下载与命名

- 默认命名：`游戏名+展示估值+排名`
- 保留原始扩展名
- 如果详情始终拿不到稳定媒体，但列表缩略图有效，可以把列表缩略图作为低置信 fallback
- 对 fallback 项，在 `summary.json` 里计数

## Checkpoint 约定

推荐存这些中间文件：

- `<keyword>_scan_page_01.json`
- `<keyword>_scan_page_02.json`
- `<keyword>_scan_page_03.json`
- `<keyword>_candidates.json`
- `<keyword>_details_batch_01.json`
- `<keyword>_details_batch_02.json`
- `<keyword>_manifest.json`
- `<keyword>_summary.json`

中断后优先从 checkpoint 恢复，不要整轮重扫。

## 输出摘要

结束时至少汇报：

- 原始扫描条数
- 缩略图去重后条数
- 详情最终去重后条数
- 低置信 fallback 条数
- 下载完成数
- 各扩展名数量

## 资源

- 读取 [references/guangdada-workflow.md](references/guangdada-workflow.md) 获取页面路径、列表视图、点击策略、常见故障。
- 读取 [references/fast-pipeline.md](references/fast-pipeline.md) 获取懒加载补采、checkpoint、续跑和摘要流程。
- 用 `scripts/build_run_artifacts.py` 从扫描/详情 checkpoint 重建 `candidates`、`manifest` 和 `summary`。
- 用 `scripts/download_manifest.py` 下载最终 manifest。
