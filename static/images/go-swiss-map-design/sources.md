# 图片与来源说明：go-swiss-map-design

## 图片

本目录图片均为 writer 自制解释图，不使用外部素材。

- cover.svg / cover.png：Go map 引擎更换封面图
- swiss-table-structure.svg / swiss-table-structure.png：Swiss Table group、control word、slot 结构图
- hmap-vs-swiss.svg / hmap-vs-swiss.png：旧 hmap/bmap/overflow 与 Swiss open addressing 对比图
- map-growth.svg / map-growth.png：Go Swiss map 多 table + extendible hashing 增长示意图

PNG 由本地 `sips` 从 SVG 转换，用于 Hugo 与微信公众号兼容。

## 技术来源

- Go 1.25.4 本机源码：/opt/homebrew/Cellar/go/1.25.4/libexec/src/internal/runtime/maps/map.go
- Go 1.25.4 本机源码：/opt/homebrew/Cellar/go/1.25.4/libexec/src/internal/runtime/maps/table.go
- Go 1.25.4 本机源码：/opt/homebrew/Cellar/go/1.25.4/libexec/src/internal/abi/map_swiss.go
- Researcher 交接单：/Users/zampo/Documents/renbo-blog/assets/go-array-series/01-slice-map-design/researcher-to-writer-handoff.md
- Go Blog: Faster Go maps with Swiss Tables: https://go.dev/blog/swisstable
- Go 1.24 Release Notes: https://go.dev/doc/go1.24
- Go Spec: Map types: https://go.dev/ref/spec#Map_types

## 版权/署名风险

无外部图片版权风险。图片为作者自制解释图。
