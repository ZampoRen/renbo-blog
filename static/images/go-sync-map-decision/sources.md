# 图片来源记录：go-sync-map-decision

本文配图均为作者自制 SVG，无外部图片素材。

## 文件

- `cover.svg`：sync.Map 与 map+lock 选型封面图，作者自制。
- `decision-flow.svg`：并发 map 选型流程图，作者自制。

## 依据

图中判断来自：
- Go 官方 sync.Map 文档：https://pkg.go.dev/sync#Map
- Go 1.25.4 sync.Map 默认实现源码：https://raw.githubusercontent.com/golang/go/go1.25.4/src/sync/map.go
- 上游研究交接：`/Users/zampo/.hermes/workspace/content-artifacts/research/go-sync-map/researcher-to-writer-20260703-go-sync-map.md`
