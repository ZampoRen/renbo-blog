# go-memory-model 图片与来源说明

## 自制图片

以下图片均为 writer 基于文章内容自制 SVG，并用 macOS sips 转换 PNG 以兼容博客封面与微信公众号发布链路：

- happens-before.svg / happens-before.png
  - 用途：happens-before 关系图，展示 ordinary write → sync operation → ordinary read 的可见性链。
- race-detector-flow.svg / race-detector-flow.png
  - 用途：race detector 检测流程图，展示 -race 插桩、运行路径、报告和修复回归。
- memory-barrier.svg / memory-barrier.png
  - 用途：内存屏障示意图，区分硬件层屏障概念与 Go 同步原语抽象。

## 事实来源

- The Go Memory Model: https://go.dev/ref/mem
- sync package Go 1.25.4: https://pkg.go.dev/sync@go1.25.4
- sync/atomic package Go 1.25.4: https://pkg.go.dev/sync/atomic@go1.25.4
- Data Race Detector: https://go.dev/doc/articles/race_detector
- Introducing the Go Race Detector: https://go.dev/blog/race-detector
- cmd/trace: https://go.dev/cmd/trace
- Go Diagnostics: https://go.dev/doc/diagnostics
- Russ Cox, Updating the Go Memory Model: https://research.swtch.com/gomm

## 版权/署名风险

无外部图片素材。图片为自制示意图，不需要额外署名。
