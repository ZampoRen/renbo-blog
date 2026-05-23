# go-array-design 图片与来源说明

## 图片

本目录图片均为 writer 自制技术示意图，先写 SVG，再转换为 PNG，用于 Hugo 与微信公众号兼容检查。

- cover.svg / cover.png：Go 数组的确定形状封面图
- value-semantics.svg / value-semantics.png：数组值语义与指针共享语义示意
- bce-flow.svg / bce-flow.png：BCE 证明流程图
- fixed-vs-dynamic.svg / fixed-vs-dynamic.png：固定长度数组与动态视图对比表

## 参考来源

- Go Specification: Array types
  https://go.dev/ref/spec#Array_types
- Go Specification: Comparison operators
  https://go.dev/ref/spec#Comparison_operators
- Go Specification: Index expressions
  https://go.dev/ref/spec#Index_expressions
- Go Blog: Go Slices: usage and internals
  https://go.dev/blog/slices-intro
- Effective Go: Arrays
  https://go.dev/doc/effective_go#arrays
- Go compiler README
  https://go.dev/src/cmd/compile/README
- Go compiler BCE test
  https://go.dev/test/checkbce.go
- Researcher handoff
  /Users/zampo/Documents/renbo-blog/assets/go-array-series/02-array-design/researcher-to-writer-handoff.md
- 本机 benchmark / BCE 诊断
  /Users/zampo/Documents/renbo-blog/assets/go-array-series/02-array-design/examples/
