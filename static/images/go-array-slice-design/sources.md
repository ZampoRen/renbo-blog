# 图片与来源说明

本文配图均为 writer 自制 SVG 示意图，用于解释技术结构，不来自外部图库。

## 文件
- `/images/go-array-slice-design/array-vs-slice-layout.png`：数组 vs slice 内存布局图，同时用作封面图。
- `/images/go-array-slice-design/slice-header.png`：slice header 三字段描述符图。
- `/images/go-array-slice-design/slice-growth.png`：Go 1.25 slice 扩容策略对比图。
- 同目录保留 SVG 源文件，便于后续修改；正文与封面统一引用 PNG，降低微信发布链路风险。

## 技术依据
- Go 1.25.4 runtime/slice.go：slice 结构和 nextslicecap。
- Go Blog: Go Slices: usage and internals。
- researcher handoff：/Users/zampo/Documents/renbo-blog/assets/go-array-series/01-slice-map-design/researcher-to-writer-handoff.md

## 版权/署名风险
- 无外部图片版权风险。
- 正文中不写“图片来源：作者制作”，避免交付痕迹。
