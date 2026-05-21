# 图片与资料来源

## 图片

本目录 SVG 均为 writer 自制解释图：
- `gc-flow.svg`：Go GC 一轮循环示意，基于 Go GC Guide 对 sweeping/off/marking 与 GOGC/GOMEMLIMIT 的说明制作。
- `tri-color-marking.svg`：三色标记简化模型，基于 Go hybrid write barrier proposal 中对白/灰/黑与 tricolor invariant 的定义制作。
- `stw-phases.svg`：STW 阶段示意，基于 Go runtime gctrace 文档、Go GC Guide、Go 1.8 Release Notes 制作。

## 事实来源

- Go GC Guide：`https://go.dev/doc/gc-guide`
- Go runtime package docs：`https://pkg.go.dev/runtime`
- Go 1.8 Release Notes：`https://go.dev/doc/go1.8`
- Hybrid write barrier proposal：`https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md`
- GC pacer redesign proposal：`https://go.googlesource.com/proposal/+/master/design/44167-gc-pacer-redesign.md`
