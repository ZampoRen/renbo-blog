# 图片与素材来源记录

## 自制 SVG

- `cover.svg`：作者自制，主题为 Go 同步原语三笔账。
- `mutex-modes.svg`：作者自制，解释 Mutex 正常模式与饥饿模式。
- `rwmutex-writer.svg`：作者自制，解释 RWMutex writer pending 后新 reader 阻塞。
- `pool-gc.svg`：作者自制，解释 sync.Pool primary/victim cache 与 GC 轮转。

## 事实素材来源

- 上游 handoff：`/Users/zampo/.hermes/workspace/content-artifacts/research/go-sync-primitives/researcher-to-writer-20260702-go-sync-primitives.md`
- 研究报告：`/Users/zampo/.hermes/profiles/researcher/workspace/research-report-20260702-go-sync-primitives.md`
- Go 1.25.4 本地源码：`/opt/homebrew/Cellar/go/1.25.4/libexec/src/...`
- Go sync package docs: https://pkg.go.dev/sync
- Go memory model: https://go.dev/ref/mem
- sync.Pool buffer issue: https://github.com/golang/go/issues/23199
- RWMutex benchmark discussion: https://github.com/golang/go/issues/38813
