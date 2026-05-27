# 图片来源记录

文章：Hermes 今天挂了？别重装，一行补丁先救 Codex
Slug：hermes-codex-fix-tutorial

## cover.png / cover.svg
- 类型：作者自制 SVG，使用 macOS sips 转 PNG
- 用途：Hugo 头图 / 微信封面候选
- 尺寸：900 x 383
- 设计说明：终端错误 + 绿色 FIX 按钮，呼应“Codex/OpenAI 故障快速止血”
- 外部素材：无

## inline-01.png / inline-01.svg
- 类型：作者自制 SVG，使用 macOS sips 转 PNG
- 用途：正文解释图
- 尺寸：1200 x 675
- 设计说明：Codex backend → openai-python parse_response → TypeError 的故障链路
- 外部素材：无

## 技术来源
- Hermes issue #32892：https://github.com/NousResearch/hermes-agent/issues/32892
- openai-python issue #3312：https://github.com/openai/openai-python/issues/3312
- openai-python issue #3313：https://github.com/openai/openai-python/issues/3313
- openai-python PR #3317：https://github.com/openai/openai-python/pull/3317
- Hermes issue #26599（任务单提及，但核查后确认为另一类 extra_headers 问题）：https://github.com/NousResearch/hermes-agent/issues/26599
