# 图片与资料来源：nvidia-n1x-break-x86

## 图片

- `cover.png`：作者自制图，由 SVG 转 PNG，900x383。用于文章封面，主题为 x86 软件墙与 RTX Spark/CUDA 冲击。
- `inline-01.png`：作者自制图，由 SVG 转 PNG，900x520。用于解释三次 Windows on Arm 挑战的失败路径。
- `inline-02.png`：作者自制图，由 SVG 转 PNG，900x520。用于解释 NVIDIA RTX Spark / N1X 的三个关键变量。
- `cover.svg` / `inline-01.svg` / `inline-02.svg`：原始 SVG 设计稿，保留供后续修改。

说明：图片来源信息不写入正文，避免留下交付痕迹。若发布门需要，可统一标注“作者自制”。

## 上游素材

- 视频：基地《黄仁勋这次能打破x86四十年的垄断吗？前面已经倒了三家！英伟达N1X深度拆解：3个变量，决定它能不能撞开x86这堵墙》
- URL：https://www.youtube.com/watch?v=gcCTxeLA6Mg
- 上游研究交接：`/Users/zampo/Documents/renbo-blog/assets/nvidia-n1x-video/researcher-to-writer-handoff.md`
- 清理后中文转写：`/Users/zampo/Documents/renbo-blog/assets/nvidia-n1x-video/video-01-nvidia-n1x/cleaned-transcript-zh.md`
- 中文写作素材：`/Users/zampo/Documents/renbo-blog/assets/nvidia-n1x-video/video-01-nvidia-n1x/writer-handoff-zh.md`

## 事实来源边界

- RTX Spark 的规格与官方口径主要来自 researcher 已核验的 NVIDIA Newsroom 2026-05-31 发布信息。
- Windows on Arm 三次失败、企业兼容、续航与本地 AI 矛盾等为视频作者观点/分析，正文按作者判断吸收，不写成官方事实。
- 视频中的市场份额、主板成本、TDP、电池续航推断未在本轮独立核验，正文未作为硬事实使用。

## 本轮外部核验状态

- `web_search` / `web_extract` 因 Firecrawl credits 不足失败。
- 通过 `curl` 访问候选 NVIDIA Newsroom URL 返回 News Archive 页面，未能稳定抓到具体新闻稿正文。
- 因此正文对官方规格采用 researcher 交接单中的“已确认外部事实”作为事实底座，并在 fact-check 报告中说明限制。
