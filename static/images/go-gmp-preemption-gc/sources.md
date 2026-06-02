# go-gmp-preemption-gc 图片与来源记录

## 图片

- `cover.png`：writer 自制 PNG，基于 SVG/HTML 页面用 Microsoft Edge headless 截图导出。无外部版权素材。
- `inline-preemption-flow.png`：writer 自制 PNG，基于 researcher 交付的抢占流程图 `assets/go-gmp-series/03-preemption-gc/preemption-flow.mmd` 重新设计为正文解释图。无外部版权素材。

## 事实来源

- Researcher 交接单：`/Users/zampo/Documents/renbo-blog/assets/go-gmp-series/03-preemption-gc/researcher-to-writer-handoff.md`
- Go 1.14 Release Notes：`https://go.dev/doc/go1.14`
- Go runtime source: `preempt.go`：`https://go.dev/src/runtime/preempt.go`
- Go runtime source: `proc.go`：`https://go.dev/src/runtime/proc.go`
- Go runtime source: `signal_unix.go`：`https://go.dev/src/runtime/signal_unix.go`
- Go runtime source: `mgc.go`：`https://go.dev/src/runtime/mgc.go`
- 本机 Go 1.25.4 源码：`/opt/homebrew/Cellar/go/1.25.4/libexec/src/runtime`

## 发布备注

正文不写图片来源说明，避免留下交付痕迹；如发布门需要版权说明，可按“作者自制示意图，无外部素材”处理。
